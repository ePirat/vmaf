

add_executable(vmafossexec
    main.cpp
    read_frame.c)

target_include_directories(vmafossexec PRIVATE
    "${libvmaf_inc}"
    ${stdatomic_dependency})

target_add_vmaf_common_flags(vmafossexec CCXX)

target_link_libraries(vmafossexec vmaf)
install(TARGETS vmafossexec
    RUNTIME DESTINATION ${CMAKE_INSTALL_FULL_BINDIR})

add_executable(vmaf_rc_exec
    vmaf.c
    cli_parse.c
    y4m_input.c
    vidinput.c
    yuv_input.c)
target_include_directories(vmaf_rc_exec PRIVATE
    "${libvmaf_inc}"
    "${vmaf_include}"
    ${stdatomic_dependency})
target_add_vmaf_common_flags(vmaf_rc_exec CCXX)
target_link_libraries(vmaf_rc_exec vmaf_rc)
set_target_properties(vmaf_rc_exec PROPERTIES OUTPUT_NAME vmaf_rc)

add_executable(vmaf_feature
    vmaf_feature_main.c
    read_frame.c)
target_include_directories(vmaf_feature PRIVATE
    "${libvmaf_inc}"
    ${stdatomic_dependency})
target_add_vmaf_common_flags(vmaf_feature CCXX)
target_link_libraries(vmaf_feature vmaf)