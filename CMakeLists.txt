# DO NOT USE THIS FILE.
# IT'S ONLY FOR CUBLAS BUILD PURPOSES ON WINDOWS VISUAL STUDIO.
# IT WILL NOT BE UPDATED OR MAINTAINED !!!

message(STATUS "============== ============== ==============")
message(STATUS "WARNING! Recommend NOT to use this file. It is UNSUPPORTED for normal users. Use MAKE instead.")
message(STATUS "It is ONLY for CUBLAS builds on windows visual studio. IT WILL OVERWRITE YOUR EXISTING MAKEFILE !!!")
message(STATUS "IF YOU ARE SEEING THIS, you MUST ONLY be building CUBLAS BUILDS! NOTHING ELSE WILL BE SUPPORTED !!!")
message(STATUS "============== ============== ==============")

cmake_minimum_required(VERSION 3.12) # Don't bump this version for no reason
project("llama.cpp" C CXX)


set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS 1)
set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Release")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(LLAMA_STANDALONE ON)
else()
    set(LLAMA_STANDALONE OFF)
endif()

if (EMSCRIPTEN)
    set(BUILD_SHARED_LIBS_DEFAULT OFF)

    option(LLAMA_WASM_SINGLE_FILE "llama: embed WASM inside the generated llama.js" ON)
else()
    if (MINGW)
        set(BUILD_SHARED_LIBS_DEFAULT OFF)
    else()
        set(BUILD_SHARED_LIBS_DEFAULT ON)
    endif()
endif()


#
# Option list
#

# general
option(LLAMA_STATIC                     "llama: static link libraries"                          OFF)
option(LLAMA_NATIVE                     "llama: enable -march=native flag"                      OFF)
option(LLAMA_LTO                        "llama: enable link time optimization"                  OFF)

# debug
option(LLAMA_ALL_WARNINGS               "llama: enable all compiler warnings"                   ON)
option(LLAMA_ALL_WARNINGS_3RD_PARTY     "llama: enable all compiler warnings in 3rd party libs" OFF)
option(LLAMA_GPROF                      "llama: enable gprof"                                   OFF)

# sanitizers
option(LLAMA_SANITIZE_THREAD            "llama: enable thread sanitizer"                        OFF)
option(LLAMA_SANITIZE_ADDRESS           "llama: enable address sanitizer"                       OFF)
option(LLAMA_SANITIZE_UNDEFINED         "llama: enable undefined sanitizer"                     OFF)

# upstream example files
option(MAKE_MISC_FILES              "MAKE_MISC_FILES, llama: build examples"                    OFF)

# instruction set specific
option(LLAMA_AVX                        "llama: enable AVX"                                     ON)
option(LLAMA_AVX2                       "llama: enable AVX2"                                    OFF)
option(LLAMA_AVX512                     "llama: enable AVX512"                                  OFF)
option(LLAMA_AVX512_VBMI                "llama: enable AVX512-VBMI"                             OFF)
option(LLAMA_AVX512_VNNI                "llama: enable AVX512-VNNI"                             OFF)
option(LLAMA_FMA                        "llama: enable FMA"                                     OFF)
# in MSVC F16C is implied with AVX2/AVX512
if (NOT MSVC)
    option(LLAMA_F16C                   "llama: enable F16C"                                    ON)
endif()

# 3rd party libs
option(LLAMA_ACCELERATE                      "llama: enable Accelerate framework"               ON)
option(LLAMA_BLAS                            "llama: use BLAS"                                  OFF)
set(LLAMA_BLAS_VENDOR "Generic" CACHE STRING "llama: BLAS library vendor")
option(LLAMA_CUBLAS                          "llama: use CUDA"                                  OFF)
set(LLAMA_CUDA_MMQ_Y       "64" CACHE STRING "llama: y tile size for mmq CUDA kernels")
option(LLAMA_CUDA_FORCE_DMMV                 "llama: use dmmv instead of mmvq CUDA kernels"     OFF)
set(LLAMA_CUDA_DMMV_X      "32" CACHE STRING "llama: x stride for dmmv CUDA kernels")
set(LLAMA_CUDA_MMV_Y        "1" CACHE STRING "llama: y block size for mmv CUDA kernels")
option(LLAMA_CUDA_F16                        "llama: use 16 bit floats for some calculations"   OFF)
set(LLAMA_CUDA_KQUANTS_ITER "2" CACHE STRING "llama: iters./thread per block for Q2_K/Q6_K")
set(LLAMA_CUDA_PEER_MAX_BATCH_SIZE "128" CACHE STRING
                                             "llama: max. batch size for using peer access")
option(LLAMA_HIPBLAS                         "llama: use hipBLAS"                               ON)
option(LLAMA_K_QUANTS                        "llama: use k-quants"                              ON)

set(LLAMA_CUDA_DMMV_Y       "1" CACHE STRING "llama: y block size for dmmv CUDA kernels")


#
# Compile flags
#

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED true)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED true)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

if (NOT MSVC)
    if (LLAMA_SANITIZE_THREAD)
        add_compile_options(-fsanitize=thread)
        link_libraries(-fsanitize=thread)
    endif()

    if (LLAMA_SANITIZE_ADDRESS)
        add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
        link_libraries(-fsanitize=address)
    endif()

    if (LLAMA_SANITIZE_UNDEFINED)
        add_compile_options(-fsanitize=undefined)
        link_libraries(-fsanitize=undefined)
    endif()
endif()


if (LLAMA_K_QUANTS)
    set(GGML_SOURCES_EXTRA ${GGML_SOURCES_EXTRA} k_quants.c k_quants.h)
    add_compile_definitions(GGML_USE_K_QUANTS)
    if (LLAMA_QKK_64)
        add_compile_definitions(GGML_QKK_64)
    endif()
endif()
add_compile_definitions(LOG_DISABLE_LOGS)

set(GGML_SOURCES_CUDA ggml-cuda.cu ggml-cuda.h)
set(GGML_V2_CUDA_SOURCES otherarch/ggml_v2-cuda.cu otherarch/ggml_v2-cuda.h)
set(GGML_V2_LEGACY_CUDA_SOURCES otherarch/ggml_v2-cuda-legacy.cu otherarch/ggml_v2-cuda-legacy.h)

if (LLAMA_CUBLAS)
    cmake_minimum_required(VERSION 3.17)

    find_package(CUDAToolkit)
    if (CUDAToolkit_FOUND)
        message(STATUS "cuBLAS found")

        enable_language(CUDA)

        add_compile_definitions(GGML_USE_CUBLAS)
#        if (LLAMA_CUDA_CUBLAS)
#            add_compile_definitions(GGML_CUDA_CUBLAS)
#        endif()
        if (LLAMA_CUDA_FORCE_DMMV)
            add_compile_definitions(GGML_CUDA_FORCE_DMMV)
        endif()
        add_compile_definitions(GGML_CUDA_DMMV_X=${LLAMA_CUDA_DMMV_X})
        add_compile_definitions(GGML_CUDA_MMV_Y=${LLAMA_CUDA_MMV_Y})
        if (DEFINED LLAMA_CUDA_DMMV_Y)
            add_compile_definitions(GGML_CUDA_MMV_Y=${LLAMA_CUDA_DMMV_Y}) # for backwards compatibility
        endif()
        if (LLAMA_CUDA_F16 OR LLAMA_CUDA_DMMV_F16)
            add_compile_definitions(GGML_CUDA_F16)
        endif()
        add_compile_definitions(K_QUANTS_PER_ITERATION=${LLAMA_CUDA_KQUANTS_ITER})
        add_compile_definitions(GGML_CUDA_PEER_MAX_BATCH_SIZE=${LLAMA_CUDA_PEER_MAX_BATCH_SIZE})

        if (LLAMA_STATIC)
            set(LLAMA_EXTRA_LIBS ${LLAMA_EXTRA_LIBS} CUDA::cudart_static CUDA::cublas_static CUDA::cublasLt_static)
        else()
            set(LLAMA_EXTRA_LIBS ${LLAMA_EXTRA_LIBS} CUDA::cudart CUDA::cublas CUDA::cublasLt)
        endif()

    if (NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
        # 52 == lowest CUDA 12 standard
        # 60 == f16 CUDA intrinsics
        # 61 == integer CUDA intrinsics
        # 70 == compute capability at which unrolling a loop in mul_mat_q kernels is faster
        if (LLAMA_CUDA_F16 OR LLAMA_CUDA_DMMV_F16)
            set(CMAKE_CUDA_ARCHITECTURES "60;61;70") # needed for f16 CUDA intrinsics
        else()
            set(CMAKE_CUDA_ARCHITECTURES "52;61;70") # lowest CUDA 12 standard + lowest for integer intrinsics
        endif()
    endif()
    message(STATUS "Using CUDA architectures: ${CMAKE_CUDA_ARCHITECTURES}")

    else()
        message(WARNING "cuBLAS not found")
    endif()
endif()

if (LLAMA_HIPBLAS)
    if (MSVC)
        list(APPEND CMAKE_PREFIX_PATH "C:/Program Files/AMD/ROCm/5.5")
    else()
        list(APPEND CMAKE_PREFIX_PATH /opt/rocm)
    endif()


    if (NOT ${CMAKE_C_COMPILER_ID} MATCHES "Clang")
        message(WARNING "Only LLVM is supported for HIP, hint: CC=/opt/rocm/llvm/bin/clang")
    endif()
    if (NOT ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
        message(WARNING "Only LLVM is supported for HIP, hint: CXX=/opt/rocm/llvm/bin/clang++")
    endif()

    find_package(hip)
    find_package(hipblas)
    find_package(rocblas)

    if (${hipblas_FOUND} AND ${hip_FOUND})
        message(STATUS "HIP and hipBLAS found")
        add_compile_definitions(GGML_USE_HIPBLAS GGML_USE_CUBLAS)
        add_library(ggml-rocm OBJECT ${GGML_SOURCES_CUDA})
        if (LLAMA_CUDA_FORCE_DMMV)
            target_compile_definitions(ggml-rocm PUBLIC GGML_CUDA_FORCE_DMMV)
        endif()
        target_compile_definitions(ggml-rocm PUBLIC GGML_CUDA_DMMV_X=${LLAMA_CUDA_DMMV_X})
        target_compile_definitions(ggml-rocm PUBLIC GGML_CUDA_MMV_Y=${LLAMA_CUDA_MMV_Y})
        target_compile_definitions(ggml-rocm PUBLIC K_QUANTS_PER_ITERATION=${LLAMA_CUDA_KQUANTS_ITER})
        set_source_files_properties(ggml-cuda.cu PROPERTIES LANGUAGE CXX)
        target_link_libraries(ggml-rocm PUBLIC hip::device hip::host roc::rocblas roc::hipblas)


        add_library(ggml-v2-rocm OBJECT ${GGML_V2_CUDA_SOURCES})
        if (LLAMA_CUDA_FORCE_DMMV)
            target_compile_definitions(ggml-v2-rocm PUBLIC GGML_CUDA_FORCE_DMMV)
        endif()
        target_compile_definitions(ggml-v2-rocm PUBLIC GGML_CUDA_DMMV_X=${LLAMA_CUDA_DMMV_X})
        target_compile_definitions(ggml-v2-rocm PUBLIC GGML_CUDA_MMV_Y=${LLAMA_CUDA_MMV_Y})
        target_compile_definitions(ggml-v2-rocm PUBLIC K_QUANTS_PER_ITERATION=${LLAMA_CUDA_KQUANTS_ITER})
        set_source_files_properties(otherarch/ggml_v2-cuda.cu PROPERTIES LANGUAGE CXX)
        target_link_libraries(ggml-v2-rocm PUBLIC hip::device hip::host roc::rocblas roc::hipblas)


        add_library(ggml-v2-legacy-rocm OBJECT ${GGML_V2_LEGACY_CUDA_SOURCES})
        if (LLAMA_CUDA_FORCE_DMMV)
            target_compile_definitions(ggml-v2-legacy-rocm PUBLIC GGML_CUDA_FORCE_DMMV)
        endif()
        target_compile_definitions(ggml-v2-legacy-rocm PUBLIC GGML_CUDA_DMMV_X=${LLAMA_CUDA_DMMV_X})
        target_compile_definitions(ggml-v2-legacy-rocm PUBLIC GGML_CUDA_MMV_Y=${LLAMA_CUDA_MMV_Y})
        target_compile_definitions(ggml-v2-legacy-rocm PUBLIC K_QUANTS_PER_ITERATION=${LLAMA_CUDA_KQUANTS_ITER})
        set_source_files_properties(otherarch/ggml_v2-cuda-legacy.cu PROPERTIES LANGUAGE CXX)
        target_link_libraries(ggml-v2-legacy-rocm PUBLIC hip::device hip::host roc::rocblas roc::hipblas)




        if (LLAMA_STATIC)
            message(FATAL_ERROR "Static linking not supported for HIP/ROCm")
        endif()
        set(LLAMA_EXTRA_LIBS ${LLAMA_EXTRA_LIBS} ggml-rocm ggml-v2-rocm ggml-v2-legacy-rocm)
    else()
        message(WARNING "hipBLAS or HIP not found. Try setting CMAKE_PREFIX_PATH=/opt/rocm")
    endif()
endif()

if (LLAMA_ALL_WARNINGS)
    if (NOT MSVC)
        set(warning_flags -Wall -Wextra -Wpedantic -Wcast-qual -Wno-unused-function)
        set(c_flags -Wshadow -Wstrict-prototypes -Wpointer-arith -Wmissing-prototypes -Werror=implicit-int
            -Werror=implicit-function-declaration)
        set(cxx_flags -Wmissing-declarations -Wmissing-noreturn)

        if (CMAKE_C_COMPILER_ID MATCHES "Clang")
            set(warning_flags ${warning_flags} -Wunreachable-code-break -Wunreachable-code-return)
            set(cxx_flags ${cxx_flags} -Wmissing-prototypes -Wextra-semi)

            if (
                (CMAKE_C_COMPILER_ID STREQUAL "Clang"      AND CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL 3.8.0) OR
                (CMAKE_C_COMPILER_ID STREQUAL "AppleClang" AND CMAKE_C_COMPILER_VERSION VERSION_GREATER_EQUAL 7.3.0)
            )
                set(c_flags ${c_flags} -Wdouble-promotion)
            endif()
        elseif (CMAKE_C_COMPILER_ID STREQUAL "GNU")
            set(c_flags ${c_flags} -Wdouble-promotion)
            set(cxx_flags ${cxx_flags} -Wno-array-bounds)

            if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 7.1.0)
                set(cxx_flags ${cxx_flags} -Wno-format-truncation)
            endif()
            if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 8.1.0)
                set(cxx_flags ${cxx_flags} -Wextra-semi)
            endif()
        endif()
    else()
        # todo : msvc
    endif()

    add_compile_options(
            ${warning_flags}
            "$<$<COMPILE_LANGUAGE:C>:${c_flags}>"
            "$<$<COMPILE_LANGUAGE:CXX>:${cxx_flags}>"
    )

endif()

if (WIN32)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)

    if (BUILD_SHARED_LIBS)
        set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
    endif()
endif()

if (LLAMA_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT result OUTPUT output)
    if (result)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    else()
        message(WARNING "IPO is not supported: ${output}")
    endif()
endif()

# Architecture specific
# TODO: probably these flags need to be tweaked on some architectures
#       feel free to update the Makefile for your architecture and send a pull request or issue
message(STATUS "CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")
if (NOT MSVC)
    if (LLAMA_STATIC)
        add_link_options(-static)
        if (MINGW)
            add_link_options(-static-libgcc -static-libstdc++)
        endif()
    endif()
    if (LLAMA_GPROF)
        add_compile_options(-pg)
    endif()
    if (LLAMA_NATIVE)
        add_compile_options(-march=native)
    endif()
endif()

if ((${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm") OR (${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64"))
    message(STATUS "ARM detected")
    if (MSVC)
        # TODO: arm msvc?
    else()
        if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "armv6")
            # Raspberry Pi 1, Zero
            add_compile_options(-mfpu=neon-fp-armv8 -mfp16-format=ieee -mno-unaligned-access)
        endif()
        if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "armv7")
            # Raspberry Pi 2
            add_compile_options(-mfpu=neon-fp-armv8 -mfp16-format=ieee -mno-unaligned-access -funsafe-math-optimizations)
        endif()
        if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "armv8")
            # Raspberry Pi 3, 4, Zero 2 (32-bit)
            add_compile_options(-mfp16-format=ieee -mno-unaligned-access)
        endif()
    endif()
elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "^(x86_64|i686|AMD64)$")
    message(STATUS "x86 detected")
    if (MSVC)
        if (LLAMA_AVX512)
            add_compile_options($<$<COMPILE_LANGUAGE:C>:/arch:AVX512>)
            add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/arch:AVX512>)
            # MSVC has no compile-time flags enabling specific
            # AVX512 extensions, neither it defines the
            # macros corresponding to the extensions.
            # Do it manually.
            if (LLAMA_AVX512_VBMI)
                add_compile_definitions($<$<COMPILE_LANGUAGE:C>:__AVX512VBMI__>)
                add_compile_definitions($<$<COMPILE_LANGUAGE:CXX>:__AVX512VBMI__>)
            endif()
            if (LLAMA_AVX512_VNNI)
                add_compile_definitions($<$<COMPILE_LANGUAGE:C>:__AVX512VNNI__>)
                add_compile_definitions($<$<COMPILE_LANGUAGE:CXX>:__AVX512VNNI__>)
            endif()
        elseif (LLAMA_AVX2)
            add_compile_options($<$<COMPILE_LANGUAGE:C>:/arch:AVX2>)
            add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/arch:AVX2>)
        elseif (LLAMA_AVX)
            add_compile_options($<$<COMPILE_LANGUAGE:C>:/arch:AVX>)
            add_compile_options($<$<COMPILE_LANGUAGE:CXX>:/arch:AVX>)
        endif()
    else()
        if (LLAMA_F16C)
            add_compile_options(-mf16c)
        endif()
        if (LLAMA_FMA)
            add_compile_options(-mfma)
        endif()
        if (LLAMA_AVX)
            add_compile_options(-mavx)
        endif()
        if (LLAMA_AVX2)
            add_compile_options(-mavx2)
        endif()
        if (LLAMA_AVX512)
            add_compile_options(-mavx512f)
            add_compile_options(-mavx512bw)
        endif()
        if (LLAMA_AVX512_VBMI)
            add_compile_options(-mavx512vbmi)
        endif()
        if (LLAMA_AVX512_VNNI)
            add_compile_options(-mavx512vnni)
        endif()
    endif()
elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "ppc64")
    message(STATUS "PowerPC detected")
    add_compile_options(-mcpu=native -mtune=native)
    #TODO: Add  targets for Power8/Power9 (Altivec/VSX) and Power10(MMA) and query for big endian systems (ppc64/le/be)
else()
    message(STATUS "Unknown architecture")
endif()

#
# POSIX conformance
#

# clock_gettime came in POSIX.1b (1993)
# CLOCK_MONOTONIC came in POSIX.1-2001 / SUSv3 as optional
# posix_memalign came in POSIX.1-2001 / SUSv3
# M_PI is an XSI extension since POSIX.1-2001 / SUSv3, came in XPG1 (1985)
add_compile_definitions(_XOPEN_SOURCE=600)

# Somehow in OpenBSD whenever POSIX conformance is specified
# some string functions rely on locale_t availability,
# which was introduced in POSIX.1-2008, forcing us to go higher
if (CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
    remove_definitions(-D_XOPEN_SOURCE=600)
    add_compile_definitions(_XOPEN_SOURCE=700)
endif()

# Data types, macros and functions related to controlling CPU affinity and
# some memory allocation are available on Linux through GNU extensions in libc
if (CMAKE_SYSTEM_NAME MATCHES "Linux")
    add_compile_definitions(_GNU_SOURCE)
endif()

# RLIMIT_MEMLOCK came in BSD, is not specified in POSIX.1,
# and on macOS its availability depends on enabling Darwin extensions
# similarly on DragonFly, enabling BSD extensions is necessary
if (
    CMAKE_SYSTEM_NAME MATCHES "Darwin" OR
    CMAKE_SYSTEM_NAME MATCHES "iOS" OR
    CMAKE_SYSTEM_NAME MATCHES "tvOS" OR
    CMAKE_SYSTEM_NAME MATCHES "DragonFly"
)
    add_compile_definitions(_DARWIN_C_SOURCE)
endif()

# alloca is a non-standard interface that is not visible on BSDs when
# POSIX conformance is specified, but not all of them provide a clean way
# to enable it in such cases
if (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
    add_compile_definitions(__BSD_VISIBLE)
endif()
if (CMAKE_SYSTEM_NAME MATCHES "NetBSD")
    add_compile_definitions(_NETBSD_SOURCE)
endif()
if (CMAKE_SYSTEM_NAME MATCHES "OpenBSD")
    add_compile_definitions(_BSD_SOURCE)
endif()

#
# Build libraries
#

add_library(ggml OBJECT
            ggml.c
            ggml.h
            ggml-alloc.c
            ggml-alloc.h
            ggml-backend.c
            ggml-backend.h
            k_quants.h
            k_quants.c
            ${GGML_SOURCES_EXTRA}
            )
target_include_directories(ggml PUBLIC . ./otherarch ./otherarch/tools)
target_compile_features(ggml PUBLIC c_std_11) # don't bump
target_link_libraries(ggml PUBLIC Threads::Threads ${LLAMA_EXTRA_LIBS})
set_target_properties(ggml PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_library(ggml_v1 OBJECT
            otherarch/ggml_v1.c
            otherarch/ggml_v1.h)
target_include_directories(ggml_v1 PUBLIC . ./otherarch ./otherarch/tools)
target_compile_features(ggml_v1 PUBLIC c_std_11) # don't bump
target_link_libraries(ggml_v1 PUBLIC Threads::Threads ${LLAMA_EXTRA_LIBS})
set_target_properties(ggml_v1 PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_library(ggml_v2 OBJECT
            otherarch/ggml_v2.c
            otherarch/ggml_v2.h)
target_include_directories(ggml_v2 PUBLIC . ./otherarch ./otherarch/tools)
target_compile_features(ggml_v2 PUBLIC c_std_11) # don't bump
target_link_libraries(ggml_v2 PUBLIC Threads::Threads ${LLAMA_EXTRA_LIBS})
set_target_properties(ggml_v2 PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_library(common2
            common/common.cpp
            common/common.h
            common/grammar-parser.h
            common/grammar-parser.cpp)
target_include_directories(common2 PUBLIC . ./otherarch ./otherarch/tools ./examples ./common)
target_compile_features(common2 PUBLIC cxx_std_11) # don't bump
target_link_libraries(common2 PRIVATE ggml ${LLAMA_EXTRA_LIBS})
set_target_properties(common2 PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_library(gpttype_adapter
            gpttype_adapter.cpp)
target_include_directories(gpttype_adapter PUBLIC . ./otherarch ./otherarch/tools ./examples ./common)
target_compile_features(gpttype_adapter PUBLIC cxx_std_11) # don't bump
target_link_libraries(gpttype_adapter PRIVATE common2 ggml ${LLAMA_EXTRA_LIBS})
set_target_properties(gpttype_adapter PROPERTIES POSITION_INDEPENDENT_CODE ON)

if (LLAMA_CUBLAS)
    set(TARGET koboldcpp_cublas)
    add_library(${TARGET} SHARED expose.cpp expose.h)
    target_include_directories(${TARGET} PUBLIC . ./otherarch ./otherarch/tools ./examples ./common)
    target_compile_features(${TARGET} PUBLIC cxx_std_11) # don't bump
    set_target_properties(${TARGET} PROPERTIES PREFIX "")
    set_target_properties(${TARGET} PROPERTIES OUTPUT_NAME "koboldcpp_cublas")
    set_target_properties(${TARGET} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    target_link_libraries(${TARGET} PUBLIC Threads::Threads ggml ggml_v1 ggml_v2 common2 gpttype_adapter ${LLAMA_EXTRA_LIBS})
    target_compile_features(${TARGET} PRIVATE cxx_std_11)
endif()

if (LLAMA_HIPBLAS)
    set(TARGET koboldcpp_hipblas)
    add_library(${TARGET} SHARED expose.cpp expose.h)
    target_include_directories(${TARGET} PUBLIC . ./otherarch ./otherarch/tools ./examples ./common)
    target_compile_features(${TARGET} PUBLIC cxx_std_11) # don't bump
    set_target_properties(${TARGET} PROPERTIES PREFIX "")
    set_target_properties(${TARGET} PROPERTIES OUTPUT_NAME "koboldcpp_hipblas")
    set_target_properties(${TARGET} PROPERTIES POSITION_INDEPENDENT_CODE ON)
    target_link_libraries(${TARGET} PUBLIC Threads::Threads ggml ggml_v1 ggml_v2 common2 gpttype_adapter ${LLAMA_EXTRA_LIBS})
    target_compile_features(${TARGET} PRIVATE cxx_std_11)
endif()


if (MAKE_MISC_FILES)
add_subdirectory(common)
add_library(llama
        llama.cpp
        llama.h
        )
target_include_directories(llama PUBLIC .)
target_compile_features(llama PUBLIC cxx_std_11) # don't bump
target_link_libraries(llama PRIVATE
    ggml
    ${LLAMA_EXTRA_LIBS}
    )
add_subdirectory(examples)
endif()
