set(VMAF_API_VERSION_MAJOR ${vmaf_api_version_major})
set(VMAF_API_VERSION_MINOR ${vmaf_api_version_minor})
set(VMAF_API_VERSION_PATCH ${vmaf_api_version_revision})
configure_file(version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.h
    @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/version.h
    libvmaf.h
    DESTINATION ${CMAKE_INSTALL_FULL_INCLUDEDIR}/libvmaf)