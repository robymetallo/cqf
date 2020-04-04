cmake_minimum_required(VERSION 3.12)
project(cqf
        HOMEPAGE_URL "https://github.com/splatlab/cqf"
        LANGUAGES C
        VERSION 0.1.1
        DESCRIPTION "Cmake for cqf library.")

cmake_policy(VERSION 3.12)

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE STRING
            "Default BUILD_TYPE is ${default_build_type}" FORCE)
endif ()

SET(WITH_BMI false CACHE BOOL "Compile with BMI and BMI2 turned on.")
SET(WITH_SSE42 false CACHE BOOL "Compile with SSE4.2 turned on.")

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

if (${WITH_BMI})
    target_compile_options(cqf PRIVATE -mbmi -mbmi2)
    message(STATUS "Using BMI and BMI2 because -DWITH_BMI was specified.")
#else ()
#    if (${HAS_BMI1} GREATER -1)
#        target_compile_options(cqf PRIVATE -mbmi)
#        message(STATUS "BMI available")
#    else ()
#        message(STATUS "BMI not available")
#    endif ()
#    if (${HAS_BMI2} GREATER -1)
#        target_compile_options(cqf PRIVATE -mbmi2)
#        message(STATUS "BMI2 available")
#    else ()
#        message(STATUS "BMI2 not available")
#    endif ()
endif ()
if (${WITH_SSE42})
    target_compile_options(cqf PRIVATE -msse4.2)
    message(STATUS "Using SSE4.2 because -DWITH_SSE42 was specified.")
#else ()
#    if (${HAS_SSE4_2} GREATER -1)
#        target_compile_options(cqf PRIVATE -msse4.2)
#        message(STATUS "SSE4.2 available")
#    else ()
#        message(STATUS "SSE4.2 not available")
#    endif ()
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
    set(CMAKE_POSITION_INDEPENDENT_CODE OFF)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION FALSE)
    target_compile_options(cqf INTERFACE
            -fbuiltin
            -g3
            -Og
            -fno-omit-frame-pointer)
    target_link_options(cqf INTERFACE
            -fno-omit-frame-pointer
            -no-pie)
    if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        target_compile_options(cqf INTERFACE
                -fprofile-instr-generate
                -fcoverage-mapping)
    elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        target_compile_options(cqf INTERFACE
                -fprofile-arcs
                -ftest-coverage)
        target_link_options(cqf INTERFACE
                -fprofile-arcs
                -ftest-coverage)
    endif ()
else ()
    target_compile_options(cqf INTERFACE
            -O3)

endif ()
