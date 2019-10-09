find_package(Doxygen)

if(DOXYGEN_FOUND)
    set(DOXYGEN_INPUT
        "${PROJECT_SOURCE_DIR}/libvmaf/include/libvmaf ${PROJECT_SOURCE_DIR}/libvmaf/src/feature")
    set(DOXYGEN_STRIP "${PROJECT_SOURCE_DIR}/libvmaf/include")
    set(DOXYGEN_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}")
    configure_file(Doxyfile.in
        ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
        @ONLY)
    add_custom_target(Name ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
        BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/html
        COMMENT "Building Docs"
        SOURCES ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
endif()