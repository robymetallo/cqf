cmake_minimum_required(VERSION 3.12)
project(cqf
        HOMEPAGE_URL "https://github.com/splatlab/cqf"
        LANGUAGES C
        VERSION 0.1.1
        DESCRIPTION "Counting Quotient Filter by splatlab")

cmake_policy(VERSION 3.12)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING
            "Default BUILD_TYPE is Release" FORCE)
endif ()

SET(WITH_BMI false CACHE BOOL "Compile with BMI and BMI2 turned on.")
SET(WITH_SSE42 false CACHE BOOL "Compile with SSE4.2 turned on.")

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
# Use '-fPIC' or '-fPIE'
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Detect SSE4_2 and BMI1 and BMI2 capabilities (required for PDEP and TZCNT instructions)
file(READ /proc/cpuinfo CPUINFO LIMIT 4096)
string(REGEX MATCH "flags[\t ]+:[A-Za-z0-9_ ]+" CPUINFO_FLAGS "${CPUINFO}")
string(FIND "${CPUINFO_FLAGS}" sse4_2 HAS_SSE4_2)
string(FIND "${CPUINFO_FLAGS}" bmi1 HAS_BMI1)
string(FIND "${CPUINFO_FLAGS}" bmi2 HAS_BMI2)

find_package(OpenSSL REQUIRED)
find_package(Threads REQUIRED)

add_library(cqf SHARED)

target_sources(cqf
        PRIVATE src/gqf.c
        PRIVATE src/gqf_file.c
        PRIVATE src/hashutil.c
        PRIVATE src/partitioned_counter.c)

target_link_libraries(cqf
        OpenSSL::SSL
        OpenSSL::Crypto
        Threads::Threads)

target_include_directories(cqf PUBLIC include)

if (${WITH_BMI})
    target_compile_options(cqf PRIVATE -mbmi -mbmi2)
    message(STATUS "Using BMI and BMI2 because -DWITH_BMI was specified.")
else ()
    if (${HAS_BMI1} GREATER -1)
        target_compile_options(cqf PRIVATE -mbmi)
        message(STATUS "BMI available")
    else ()
        message(STATUS "BMI not available")
    endif ()
    if (${HAS_BMI2} GREATER -1)
        target_compile_options(cqf PRIVATE -mbmi2)
        message(STATUS "BMI2 available")
    else ()
        message(STATUS "BMI2 not available")
    endif ()
endif ()
if (${WITH_SSE42})
    target_compile_options(cqf PRIVATE -msse4.2)
    message(STATUS "Using SSE4.2 because -DWITH_SSE42 was specified.")
else ()
    if (${HAS_SSE4_2} GREATER -1)
        target_compile_options(cqf PRIVATE -msse4.2)
        message(STATUS "SSE4.2 available")
    else ()
        message(STATUS "SSE4.2 not available")
    endif ()
endif ()

if (WITH_ASAN)
    message(STATUS "Enabling ASAN")
    target_compile_options(cqf PRIVATE -fsanitize=address)
    target_link_options(cqf PRIVATE -fsanitize=address)
elseif (WITH_TSAN)
    message(STATUS "Enabling TSAN")
    target_compile_options(cqf PRIVATE -fsanitize=thread)
    target_link_options(cqf PRIVATE -fsanitize=thread)
elseif (WITH_UBSAN) # https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
    message(STATUS "Enabling UBSAN")
    target_compile_options(cqf PRIVATE
            -fsanitize=signed-integer-overflow,null,alignment
            -fno-sanitize-recover=null
            -fsanitize-trap=alignment)
    target_link_options(cqf PRIVATE
            -fsanitize=signed-integer-overflow,null,alignment
            -fno-sanitize-recover=null
            -fsanitize-trap=alignment)
elseif (WITH_MSAN)
    message(STATUS "Enabling MSAN")
    target_compile_options(cqf PRIVATE -fsanitize=memory -fPIE -pie)
    target_link_options(cqf PRIVATE -fsanitize=memory -fPIE -pie)
endif ()

if (${CMAKE_BUILD_TYPE} MATCHES "^Debug*")
    message(STATUS "Disabling IPO/LTO because we are in Debug")
    message(STATUS "Disabling -fPIC because we are in Debug")
    set_target_properties(cqf PROPERTIES
            CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE
            CMAKE_POSITION_INDEPENDENT_CODE OFF)
    target_compile_options(cqf PRIVATE
            -fbuiltin
            -g3
            -Og
            -fno-omit-frame-pointer)
    target_link_options(cqf PRIVATE
            -fno-omit-frame-pointer
            -no-pie)
    if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        target_compile_options(cqf PRIVATE
                -fprofile-instr-generate
                -fcoverage-mapping)
    elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        target_compile_options(cqf PRIVATE
                -fprofile-arcs
                -ftest-coverage)
        target_link_options(cqf PRIVATE
                -fprofile-arcs
                -ftest-coverage)
    endif ()
else ()
    target_compile_options(cqf PRIVATE
            -O3)
endif ()
