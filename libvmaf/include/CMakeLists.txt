if(EXISTS "${PROJECT_SOURCE_DIR}/.git")
    find_package(Git)
    if(GIT_FOUND)
        execute_process(COMMAND ${GIT_EXECUTABLE} -C "${PROJECT_SOURCE_DIR}"
            describe --tags --long --match '?.*.*' --always
            OUTPUT_VARIABLE VCS_TAG
            ERROR_QUIET
            OUTPUT_STRIP_TRAILING_WHITESPACE)
    else()
        message(WARNING "Git directory detected but no git found")
    endif()
endif()

if(NOT DEFINED VCS_TAG)
    set(VCS_TAG "${PROJECT_VERSION}")
endif()

configure_file(vcs_version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/vcs_version.h
    @ONLY)

add_subdirectory(libvmaf)