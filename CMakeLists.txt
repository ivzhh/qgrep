cmake_minimum_required(VERSION 3.13)
# add_link_options requires 3.13 https://cmake.org/cmake/help/v3.13/command/add_link_options.html

project(qgrep)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if (WIN32)
    add_compile_options(/DUSE_SSE2 /DNOMINMAX /wd"4996" /wd"4267" /wd"4244")

    if (CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)")
        message(STATUS "use SSE2 of ${CMAKE_SYSTEM_PROCESSOR}")
        add_compile_options(/arch:SSE2)
    else()
        message(STATUS "MSVC on ${CMAKE_SYSTEM_PROCESSOR} has /arch:SSE2 by default")
    endif ()
else()
    add_compile_options(-c -g -Wall -Werror -O3 -mtune=native -MMD -MP)

    if (CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
        message(STATUS "use SSE2 of ${CMAKE_SYSTEM_PROCESSOR}")
        add_compile_options(-msse2 -DUSE_SSE2)
    else()
        message(STATUS "disable SSE2 due to ${CMAKE_SYSTEM_PROCESSOR}")
    endif ()

    if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        message(STATUS "turn on compile flags for ${CMAKE_SYSTEM_NAME}")
        add_compile_options(
            -force_cpusubtype_ALL
            -mmacosx-version-min=10.7
            -stdlib=libc++
        )
        add_link_options(
            -force_cpusubtype_ALL
            -mmacosx-version-min=10.7
            -stdlib=libc++
            "SHELL:-framework CoreFoundation"
            "SHELL:-framework CoreServices"
        )
    else()
        message(STATUS "turn on compile flags for *nix")
        add_link_options(-Wl,--dynamic-list=${CMAKE_SOURCE_DIR}/src/qgrep.dynlist)
    endif()
endif()

option(WITH_SUBMODULE_LZ4 "Use lz4 library from submodules" ON)

if(NOT (EXISTS ${CMAKE_SOURCE_DIR}/extern/re2/re2/re2.cc))
    message(FATAL_ERROR "run git submodule update --init to fetch re2")
endif()

# qgrep uses re2::Prefilter, which is not exposed by system re2
add_library(re2 STATIC 
    extern/re2/re2/bitstate.cc
    extern/re2/re2/compile.cc
    extern/re2/re2/dfa.cc
    extern/re2/re2/filtered_re2.cc
    extern/re2/re2/mimics_pcre.cc
    extern/re2/re2/nfa.cc
    extern/re2/re2/onepass.cc
    extern/re2/re2/parse.cc
    extern/re2/re2/perl_groups.cc
    extern/re2/re2/prefilter.cc
    extern/re2/re2/prefilter_tree.cc
    extern/re2/re2/prog.cc
    extern/re2/re2/re2.cc
    extern/re2/re2/regexp.cc
    extern/re2/re2/set.cc
    extern/re2/re2/simplify.cc
    extern/re2/re2/stringpiece.cc
    extern/re2/re2/tostring.cc
    extern/re2/re2/unicode_casefold.cc
    extern/re2/re2/unicode_groups.cc
    extern/re2/util/pcre.cc
    extern/re2/util/rune.cc
    extern/re2/util/strutil.cc
)

target_include_directories(re2 PUBLIC ${CMAKE_SOURCE_DIR}/extern/re2/)

set(SRC 
    src/blockpool.cpp
    src/build.cpp
    src/changes.cpp
    src/compression.cpp
    src/encoding.cpp
    src/files.cpp
    src/filestream.cpp
    src/fileutil.cpp
    src/fileutil_posix.cpp
    src/fileutil_win.cpp
    src/filter.cpp
    src/filterutil.cpp
    src/fuzzymatch.cpp
    src/highlight.cpp
    src/highlight_win.cpp
    src/info.cpp
    src/init.cpp
    src/main.cpp
    src/orderedoutput.cpp
    src/project.cpp
    src/regex.cpp
    src/search.cpp
    src/stringutil.cpp
    src/update.cpp
    src/watch.cpp
    src/workqueue.cpp
)

if (WITH_SUBMODULE_LZ4)
    if(NOT (EXISTS ${CMAKE_SOURCE_DIR}/extern/lz4/lib/lz4.c))
        message(FATAL_ERROR "run git submodule update --init to fetch lz4")
    endif()

    add_library(lz4 STATIC 
        ${CMAKE_SOURCE_DIR}/extern/lz4/lib/lz4.c
        ${CMAKE_SOURCE_DIR}/extern/lz4/lib/lz4hc.c
    )

    target_include_directories(lz4 PUBLIC ${CMAKE_SOURCE_DIR}/extern/lz4/lib)
else()
    if (WIN32)
        message(FATAL_ERROR "qgrep on Windows requires lz4 as submodule")
    endif()
    find_package(PkgConfig REQUIRED) 

    pkg_check_modules(ext_lz4 REQUIRED IMPORTED_TARGET liblz4)
endif()

add_executable(qgrep ${SRC})

if (WITH_SUBMODULE_LZ4)
    message(STATUS "use lz4 from submodule")
    target_link_libraries(qgrep PUBLIC lz4)
else()
    message(STATUS "use lz4 from system pkg-config")
    target_link_libraries(qgrep PUBLIC PkgConfig::ext_lz4)
endif()

target_link_libraries(qgrep PUBLIC re2)

if (NOT WIN32)
    target_link_libraries(qgrep PUBLIC pthread)
endif()

install(TARGETS qgrep DESTINATION bin)
