cmake_minimum_required(VERSION 3.16)
project(cqf
        LANGUAGES C
        VERSION 0.1.1
        DESCRIPTION "Cmake for cqf library.")

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE STRING
            "Default BUILD_TYPE is ${default_build_type}" FORCE)
endif ()

# Use '-fPIC' or '-fPIE' when not in Debug
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(CheckIPOSupported)
check_ipo_supported(RESULT IPO_AVAILABLE)
if (IPO_AVAILABLE)
    message(STATUS "IPO/LTO available")
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
else ()
    message(STATUS "IPO/LTO not available")
endif ()

if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
    #    set(CMAKE_POSITION_INDEPENDENT_CODE OFF)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE)
endif ()

# Detect SSE4_2 and BMI1 and BMI2 capabilities (required for PDEP and TZCNT instructions)
file(READ /proc/cpuinfo CPUINFO LIMIT 4096)
string(REGEX MATCH "flags[\t ]+:[A-Za-z0-9_ ]+" CPUINFO_FLAGS "${CPUINFO}")
string(FIND "${CPUINFO_FLAGS}" sse4_2 HAS_SSE4_2)
string(FIND "${CPUINFO_FLAGS}" bmi1 HAS_BMI1)
string(FIND "${CPUINFO_FLAGS}" bmi2 HAS_BMI2)


find_package(OpenSSL REQUIRED)
find_package(Threads REQUIRED)

add_library(cqf SHARED)

if (IPO_AVAILABLE)
    set_target_properties(cqf PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)
endif ()
set_target_properties(cqf PROPERTIES
        C_STANDARD 99
        C_STANDARD_REQUIRED ON)

target_sources(cqf PRIVATE src/gqf.c
        PRIVATE src/gqf_file.c
        PRIVATE src/hashutil.c
        PRIVATE src/partitioned_counter.c)
target_link_libraries(cqf
        OpenSSL::SSL
        OpenSSL::Crypto
        Threads::Threads)

target_include_directories(cqf PUBLIC include)


if (${HAS_BMI1} GREATER -1)
    target_compile_options(cqf PRIVATE -mbmi)
endif ()
if (${HAS_BMI2} GREATER -1)
    target_compile_options(cqf PRIVATE -mbmi2)
endif ()
if (${HAS_SSE4_2} GREATER -1)
    target_compile_options(cqf PRIVATE -msse4.2)
endif ()

if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
    set(CMAKE_POSITION_INDEPENDENT_CODE OFF)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE)
    target_compile_options(cqf PRIVATE
            -fbuiltin
            -g3
            -Og
            -fsanitize=address
            -fno-omit-frame-pointer)
    target_link_options(cqf PRIVATE
            -fsanitize=address
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
            -Wall
            -Wextra
            -pedantic
            -O3)

endif ()
