project(xmr-stak-nvidia)

cmake_minimum_required(VERSION 3.0.1)

set( CMAKE_VERBOSE_MAKEFILE on )
# enforce C++11
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 11)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}" CACHE PATH "install prefix" FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

# help to find cuda on systems with a software module system
list(APPEND CMAKE_PREFIX_PATH "$ENV{HIP_ROOT_DIR}")
# allow user to extent CMAKE_PREFIX_PATH via environment variable
list(APPEND CMAKE_PREFIX_PATH "$ENV{CMAKE_PREFIX_PATH}")

SET(CMAKE_MODULE_PATH /opt/rocm/hip/cmake)

################################################################################
# CMake user options
################################################################################

# gcc 5.1 is the first GNU version without CoW strings
# https://github.com/fireice-uk/xmr-stak-nvidia/pull/10#issuecomment-290821792
# If you remove this guard to compile with older gcc versions the miner will produce
# a high rate of wrong shares.
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.1)
        message(FATAL_ERROR "GCC version must be at least 5.1!")
    endif()
endif()

set(BUILD_TYPE "Release;Debug")
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build" FORCE)
endif()
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "${BUILD_TYPE}")

option(XMR-STAK_LARGEGRID "Support large CUDA block count > 128" ON)
if(XMR-STAK_LARGEGRID)
    add_definitions("-DXMR_STAK_LARGEGRID=${XMR-STAK_LARGEGRID}")
endif()

set(DEVICE_COMPILER "clang")
set(CUDA_COMPILER "${DEVICE_COMPILER}" CACHE STRING "Select the device compiler")

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    list(APPEND DEVICE_COMPILER "clang")
endif()

set_property(CACHE CUDA_COMPILER PROPERTY STRINGS "${DEVICE_COMPILER}")

################################################################################
# Find CUDA
################################################################################

find_package(HIP REQUIRED)
include(FindHIP)
set(LIBS ${LIBS} ${CUDA_LIBRARIES})

set(XMR-STAK_THREADS 0 CACHE STRING "Set maximum number of threads (for compile time optimization)")
if(NOT XMR-STAK_THREADS EQUAL 0)
    message(STATUS "xmr-stak-nvidia: set max threads per block to ${XMR-STAK_THREADS}")
    add_definitions("-DXMR_STAK_THREADS=${XMR-STAK_THREADS}")
endif()

set(CUDA_ARCH "20;30;35;37;50;52;60;61;62" CACHE STRING "Set GPU architecture (semicolon separated list, e.g. '-DCUDA_ARCH=20;35;60')")

# validate architectures (only numbers are allowed)
foreach(CUDA_ARCH_ELEM ${CUDA_ARCH})
    string(REGEX MATCH "^[0-9]+$" IS_NUMBER ${CUDA_ARCH})
    if(NOT IS_NUMBER)
        message(FATAL_ERROR "Defined compute architecture '${CUDA_ARCH_ELEM}' in "
                            "'${CUDA_ARCH}' is not an integral number, use e.g. '30' (for compute architecture 3.0).")
    endif()
    unset(IS_NUMBER)

    if(${CUDA_ARCH_ELEM} LESS 20)
        message(FATAL_ERROR "Unsupported CUDA architecture '${CUDA_ARCH_ELEM}' specified. "
                            "Use '20' (for compute architecture 2.0) or higher.")
    endif()
endforeach()

option(CUDA_SHOW_REGISTER "Show registers used for each kernel and compute architecture" OFF)
option(CUDA_KEEP_FILES "Keep all intermediate files that are generated during internal compilation steps" OFF)

if("${CUDA_COMPILER}" STREQUAL "clang")
    set(LIBS ${LIBS} cudart_static)
    set(CLANG_BUILD_FLAGS "-O3 -x cuda --cuda-path=${HIP_ROOT_DIR}")
    # activation usage of FMA
    set(CLANG_BUILD_FLAGS "${CLANG_BUILD_FLAGS} -ffp-contract=fast")

    if(CUDA_SHOW_REGISTER)
        set(CLANG_BUILD_FLAGS "${CLANG_BUILD_FLAGS} -Xcuda-ptxas -v")
    endif(CUDA_SHOW_REGISTER)

    if(CUDA_KEEP_FILES)
        set(CLANG_BUILD_FLAGS "${CLANG_BUILD_FLAGS} -save-temps=${PROJECT_BINARY_DIR}")
    endif(CUDA_KEEP_FILES)

    foreach(CUDA_ARCH_ELEM ${CUDA_ARCH})
        # set flags to create device code for the given architectures
        set(CLANG_BUILD_FLAGS "${CLANG_BUILD_FLAGS} --cuda-gpu-arch=sm_${CUDA_ARCH_ELEM}")
    endforeach()

elseif("${CUDA_COMPILER}" STREQUAL "/opt/rocm/bin/hipcc")
    # avoid that nvcc in CUDA < 8 tries to use libc `memcpy` within the kernel
    if(HIP_VERSION VERSION_LESS 8.0)
        add_definitions(-D_FORCE_INLINES)
    endif()
    foreach(CUDA_ARCH_ELEM ${CUDA_ARCH})
        # set flags to create device code for the given architecture
        set(HIP_NVCC_FLAGS ${HIP_NVCC_FLAGS}
            "--generate-code arch=compute_${CUDA_ARCH_ELEM},code=sm_${CUDA_ARCH_ELEM} --generate-code arch=compute_${CUDA_ARCH_ELEM},code=compute_${CUDA_ARCH_ELEM}")
    endforeach()

    # give each thread an independent default stream
    set(HIP_NVCC_FLAGS "${HIP_NVCC_FLAGS} --default-stream per-thread")

    option(CUDA_SHOW_CODELINES "Show kernel lines in cuda-gdb and cuda-memcheck" OFF)

    if(CUDA_SHOW_CODELINES)
        set(HIP_NVCC_FLAGS "${HIP_NVCC_FLAGS}" --source-in-ptx -lineinfo)
        set(CUDA_KEEP_FILES ON CACHE BOOL "activate keep files" FORCE)
    endif(CUDA_SHOW_CODELINES)

    if(CUDA_SHOW_REGISTER)
        set(HIP_NVCC_FLAGS "${HIP_NVCC_FLAGS}" -Xptxas=-v)
    endif(CUDA_SHOW_REGISTER)

    if(CUDA_KEEP_FILES)
        set(HIP_NVCC_FLAGS "${HIP_NVCC_FLAGS}" --keep --keep-dir "${PROJECT_BINARY_DIR}")
    endif(CUDA_KEEP_FILES)

else()
    message(FATAL_ERROR "selected CUDA compiler '${CUDA_COMPILER}' is not supported")
endif()

################################################################################
# Find PThreads
################################################################################

find_package(Threads REQUIRED)
set(LIBS ${LIBS} ${CMAKE_THREAD_LIBS_INIT})

################################################################################
# Find microhttpd
################################################################################

option(MICROHTTPD_REQUIRED "Enable or disable the requirement of microhttp (http deamon)" ON)
find_library(MHTD NAMES microhttpd)
if("${MHTD}" STREQUAL "MHTD-NOTFOUND")
    if(MICROHTTPD_REQUIRED)
        message(FATAL_ERROR "microhttpd NOT found: use `-DMICROHTTPD_REQUIRED=OFF` to build without http deamon support")
    else()
        message(STATUS "microhttpd NOT found: disable http server")
        add_definitions("-DCONF_NO_HTTPD")
    endif()
else()
        add_definitions("-DCONF_NO_HTTPD")
    # set(LIBS ${LIBS} ${MHTD})
endif()

###############################################################################
# Find OpenSSL
###############################################################################

option(OpenSSL_REQUIRED "Enable or disable the requirement of OpenSSL" ON)
find_package(OpenSSL)
if(OPENSSL_FOUND)
    include_directories(${OPENSSL_INCLUDE_DIR})
    set(LIBS ${LIBS} ${OPENSSL_LIBRARIES})
else()
    if(OpenSSL_REQUIRED)
        message(FATAL_ERROR "OpenSSL NOT found: use `-DOpenSSL_REQUIRED=OFF` to build without SSL support")
    else()
        if(NOT OPENSSL_FOUND)
            add_definitions("-DCONF_NO_TLS")
        endif()
    endif()
endif()

################################################################################
# Compile & Link
################################################################################

include_directories(.)
include_directories("/opt/rocm/hip/include")

# activate sse2 and aes-ni
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse2 -maes -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2 -maes -fPIC -Wall")

file(GLOB CUDASRCFILES "hip_code/*.cu")
file(GLOB SRCFILES "*[^cu].cpp" "crypto/*[^cu].cpp")
file(GLOB SRCFILES_CRYPTO "crypto/*.c")
file(GLOB SRCFILES_TEST "test/*.cpp")


if("${CUDA_COMPILER}" STREQUAL "clang")
    # build device code with clang
    add_library(xmr-stak-nvidiaCuda
        STATIC
        ${CUDASRCFILES}
    )
    set_target_properties(xmr-stak-nvidiaCuda PROPERTIES COMPILE_FLAGS ${CLANG_BUILD_FLAGS})
    set_target_properties(xmr-stak-nvidiaCuda PROPERTIES LINKER_LANGUAGE CXX)
    set_source_files_properties(${CUDASRCFILES} PROPERTIES LANGUAGE CXX)
else()
    # build device code with nvcc
    hip_add_library(xmr-stak-nvidiaCuda
        STATIC
        ${CUDASRCFILES}
    )
    #set(CMAKE_HIP_CREATE_STATIC_LIBRARY "${HIP_HIPCC_CMAKE_LINKER_HELPER} xmr-stak-nvidiaCuda")
    set_target_properties(xmr-stak-nvidiaCuda PROPERTIES LINK_FLAGS "-Bsymbolic -Bdynamic")
    #set_source_files_properties(${CUDASRCFILES} PROPERTIES LANGUAGE HIP)
endif()

add_library(xmr-stak-nvidiaCrypto
    STATIC
    ${SRCFILES_CRYPTO}
)
set_property(TARGET xmr-stak-nvidiaCrypto PROPERTY C_STANDARD 99)
#set_property(TARGET xmr-stak-nvidiaCrypto PROPERTY COMPILE_FLAGS "-msse2 -maes") 

hip_add_executable(xmr-stak-nvidia
    ${SRCFILES}
	)
  
if("${HIP_PLATFORM}" STREQUAL "hcc")
  hip_add_executable(xmr-stak-test
	${SRCFILES_TEST}
	)
endif()
#SET(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} -L/opt/rocm/hip/lib -L/opt/rocm/hsa/lib -L/opt/rocm/lib -L/usr/local/cuda-8.0/lib64 -lhip_hcc -lhsa-runtime64 -lhc_am -lhsakmt -lcudart")
#SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -rpath /opt/rocm/lib -rpath /opt/rocm/lib/libhip_hcc.so -rpath /opt/rocm/lib/libhip_device.a -rpath /usr/local/cuda-8.0/lib64 -Bsymbolic")

# for gnu compiler <5.1 wee need to activate c++11 by explicit for cuda and all cpp files
if(("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU") AND (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.1))
    set_target_properties(xmr-stak-nvidiaCuda PROPERTIES COMPILE_FLAGS -std=c++11)
    set_target_properties(xmr-stak-nvidia PROPERTIES COMPILE_FLAGS -std=c++11)
endif()

set(EXECUTABLE_OUTPUT_PATH "bin")

target_link_libraries(xmr-stak-nvidia ${LIBS} xmr-stak-nvidiaCuda xmr-stak-nvidiaCrypto)

if("${HIP_PLATFORM}" STREQUAL "hcc")
  target_link_libraries(xmr-stak-test ${LIBS} xmr-stak-nvidiaCuda xmr-stak-nvidiaCrypto)
endif()

################################################################################
# Install
################################################################################

# do not install the binary if the project and install are equal
if( NOT "${CMAKE_INSTALL_PREFIX}" STREQUAL "${PROJECT_BINARY_DIR}" )
    install(TARGETS xmr-stak-nvidia
            DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
endif()

# avoid overwrite of user defined settings
# install `config.txt`if file not exists in `${CMAKE_INSTALL_PREFIX}/bin`
install(CODE " \
    if(NOT EXISTS ${CMAKE_INSTALL_PREFIX}/bin/config.txt)\n   \
        file(INSTALL ${CMAKE_CURRENT_SOURCE_DIR}/config.txt   \
            DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)\n        \
    endif()"
)
