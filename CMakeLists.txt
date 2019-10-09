cmake_minimum_required(VERSION 3.5)

if(NOT DEFINED CMAKE_BUILD_TYPE)
    message(STATUS "No build type defined. Defaulting to Release")
    set(CMAKE_BUILD_TYPE "Release"
        CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE
        PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

if(EXISTS libvmaf/meson.build)
    file(STRINGS libvmaf/meson.build meson_version REGEX "version : '.*'")
    string(REGEX MATCH "([0-9]+)\\.([0-9]+)\\.([0-9]+)"
        meson_version ${meson_version})
endif()

if(NOT DEFINED meson_version)
    set(meson_version "1.3.16")
endif()

project(vmaf VERSION ${meson_version} LANGUAGES C CXX)

include(GNUInstallDirs)
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)
include(CTest)

find_package(Threads)

macro(add_test_compiler_flags C_OR_CXX)
    if(NOT (${C_OR_CXX} STREQUAL "C" OR ${C_OR_CXX} STREQUAL "CXX" OR ${C_OR_CXX} STREQUAL "CCXX"))
        message(AUTHOR_WARNING "In ${CMAKE_CURRENT_LIST_FILE} add_test_compiler_flags\n"
            "${C_OR_CXX} does not match C or CXX\n"
            "Defaulting to C")
    endif()

    foreach(flag ${ARGN})
        string(REGEX REPLACE "[^A-Za-z0-9]" "_" flag_var "${flag}")
        if(${C_OR_CXX} MATCHES "CXX")
            set(test_cxx_flag "CXX_FLAG${flag_var}")
            check_cxx_compiler_flag(${flag} "${test_cxx_flag}")
            if(${test_cxx_flag})
                set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${flag}")
            endif()
        endif()
        if(${C_OR_CXX} STREQUAL "C" OR ${C_OR_CXX} STREQUAL "CCXX" OR NOT ${C_OR_CXX} MATCHES "CXX")
            set(test_c_flag "C_FLAG${flag_var}")
            check_c_compiler_flag(${flag} "${test_c_flag}")
            if(${test_c_flag})
                set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${flag}")
            endif()
        endif()
    endforeach()
endmacro()

macro(target_test_compiler_flags TARGET C_OR_CXX)
    if(NOT (${C_OR_CXX} STREQUAL "C" OR ${C_OR_CXX} STREQUAL "CXX" OR ${C_OR_CXX} STREQUAL "CCXX"))
        message(AUTHOR_WARNING "In ${CMAKE_CURRENT_LIST_FILE} target_test_compiler_flags\n"
            "${C_OR_CXX} does not match C or CXX\n"
            "Defaulting to C")
    endif()

    foreach(flag ${ARGN})
        string(REGEX REPLACE "[^A-Za-z0-9]" "_" flag_var "${flag}")
        if(${C_OR_CXX} MATCHES "CXX")
            set(test_cxx_flag "CXX_FLAG${flag_var}")
            check_cxx_compiler_flag(${flag} "${test_cxx_flag}")
            if(${test_cxx_flag})
                target_compile_options(${TARGET} PRIVATE ${flag})
            endif()
        endif()
        if(${C_OR_CXX} STREQUAL "C" OR ${C_OR_CXX} STREQUAL "CCXX" OR NOT ${C_OR_CXX} MATCHES "CXX")
            set(test_c_flag "C_FLAG${flag_var}")
            check_c_compiler_flag(${flag} "${test_c_flag}")
            if(${test_c_flag})
                target_compile_options(${TARGET} PRIVATE ${flag})
            endif()
        endif()
    endforeach()
endmacro()

macro(target_add_vmaf_common_flags target C_OR_CXX)
    target_compile_definitions(${target} PRIVATE OC_NEW_STYLE_INCLUDES)
    if(MSVC)
        target_test_compiler_flags(${target} ${C_OR_CXX}
        -wd4028
        -wd4996
        ${ARGN})
    else()
        target_test_compiler_flags(${target} ${C_OR_CXX}
        -pedantic
        ${ARGN})
    endif()
endmacro()

set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(MSVC)
    set(CMAKE_C_FLAGS "/W3 ${CMAKE_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "/W3 ${CMAKE_CXX_FLAGS}")
else()
    set(CMAKE_C_FLAGS "-Wall -Winvalid-pch -Wextra ${CMAKE_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "-Wall -Winvalid-pch -Wextra ${CMAKE_CXX_FLAGS}")
endif()


set(vmaf_soname_version 0.0.0)
string(REPLACE "." ";" vmaf_api_version_array ${vmaf_soname_version})
list(GET vmaf_api_version_array 0 vmaf_api_version_major)
list(GET vmaf_api_version_array 1 vmaf_api_version_minor)
list(GET vmaf_api_version_array 2 vmaf_api_version_revision)

set(libvmaf_src_root "${CMAKE_CURRENT_LIST_DIR}/libvmaf")
set(libvmaf_inc "${libvmaf_src_root}/include")
install(DIRECTORY model DESTINATION ${CMAKE_INSTALL_DATADIR})

if(UNIX)
    if(APPLE)
        add_test_compiler_flags(C -D_DARWIN_C_SOURCE)
    else()
        add_test_compiler_flags(C -D_GNU_SOURCE)
    endif()
else()
    add_test_compiler_flags(C -D_POSIX_C_SOURCE=200112L)
endif()

check_include_file(stdatomic.h HAVE_STDATOMIC)
if(NOT HAVE_STDATOMIC AND MSVC)
    set(stdatomic_dependency "${include_dir}/compat/msvc")
elseif(NOT HAVE_STDATOMIC AND NOT DEFINED HAVE_GCC_ATOMICS)
    file(WRITE "${PROJECT_BINARY_DIR}/stdatomic.c" "
int main() {
    int v = 0;
    return __atomic_fetch_add(&v, 1, __ATOMIC_SEQ_CST);
}
")
    try_compile(HAVE_GCC_ATOMICS
        "${PROJECT_BINARY_DIR}"
        "${PROJECT_BINARY_DIR}/stdatomic.c")
endif()
if(HAVE_GCC_ATOMICS)
    set(stdatomic_dependency "${include_dir}/compat/gcc")
endif()

if(NOT HAVE_STDATOMIC AND NOT HAVE_GCC_ATOMICS)
    message(FATAL_ERROR "Atomics not supported")
endif()

set(src_dir "${CMAKE_CURRENT_LIST_DIR}/libvmaf/src")
set(ptools_path "${src_dir}/third_party/ptools")
set(opencontainers_path "${ptools_path}/opencontainers_1_8_4")
set(opencontainers_include "${opencontainers_path}/include")
set(feature_src_dir "${src_dir}/feature")
set(vmaf_include
    "${opencontainers_include}"
    "${src_dir}"
    "${feature_src_dir}"
    "${feature_src_dir}/common"
    "${ptools_path}")

add_subdirectory(libvmaf/src)
add_subdirectory(libvmaf/include)
add_subdirectory(libvmaf/tools)
add_subdirectory(libvmaf/doc)

# set_target_properties(vmaf_exec PROPERTIES OUTPUT_NAME vmaf)
