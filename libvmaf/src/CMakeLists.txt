set(vmaf_soversion "${vmaf_api_version_major}")

add_library(libptools OBJECT
    ${ptools_path}/m2pythontools.cc
    ${ptools_path}/valpython.cc
    ${ptools_path}/midassocket.cc
    ${ptools_path}/valprotocol2.cc
    ${ptools_path}/m2ser.cc
    ${ptools_path}/m2streamdataenc.cc
    ${ptools_path}/m2convertrep.cc
    ${ptools_path}/timeconv.cc)

target_include_directories(libptools PRIVATE ${opencontainers_include})
target_compile_definitions(libptools PRIVATE OC_NEW_STYLE_INCLUDES _REENTRANT)

find_library(LIBM m)

if(MSVC)
    target_test_compiler_flags(libptools CXX -wd4028 -wd4996)
else()
    target_test_compiler_flags(libptools CXX -fno-strict-aliasing)
    target_compile_definitions(libptools PRIVATE LINUX_)
    if(LIBM)
        target_link_libraries(libptools ${LIBM})
    endif()
endif()

target_link_libraries(libptools Threads::Threads)

set(third_party_dir "${CMAKE_CURRENT_LIST_DIR}/third_party")
set(vmaf_base_include "${CMAKE_CURRENT_LIST_DIR}" "${feature_src_dir}/common")

add_library(convolution_and_psnr_avx OBJECT
    "${feature_src_dir}/common/convolution_avx.c"
    "${feature_src_dir}/psnr_tools.c")

if(MSVC)
    target_add_vmaf_common_flags(convolution_and_psnr_avx C /arch:AVX)
else()
    target_add_vmaf_common_flags(convolution_and_psnr_avx C -mavx)
endif()

add_library(libvmaf_feature OBJECT
    "${feature_src_dir}/common/alignment.c"
    "${feature_src_dir}/common/convolution.c"
    "${feature_src_dir}/common/cpu.c"
    "${feature_src_dir}/offset.c"
    "${feature_src_dir}/adm.c"
    "${feature_src_dir}/adm_tools.c"
    "${feature_src_dir}/ansnr.c"
    "${feature_src_dir}/ansnr_tools.c"
    "${feature_src_dir}/vif.c"
    "${feature_src_dir}/vif_tools.c"
    "${feature_src_dir}/motion.c"
    "${feature_src_dir}/psnr.c"
    "${feature_src_dir}/ssim.c"
    "${feature_src_dir}/ms_ssim.c"
    "${feature_src_dir}/moment.c"
    "${feature_src_dir}/all.c"
    "${feature_src_dir}/common/blur_array.c"
    "${feature_src_dir}/iqa/math_utils.c"
    "${feature_src_dir}/iqa/convolve.c"
    "${feature_src_dir}/iqa/decimate.c"
    "${feature_src_dir}/iqa/ssim_tools.c")

target_include_directories(libvmaf_feature PRIVATE
    ${vmaf_include}
    ${vmaf_base_include}
    ${stdatomic_dependency})

add_library(vmaf
    "${third_party_dir}/pugixml/pugixml.cpp"
    "${src_dir}/mem.c"
    "${src_dir}/combo.c"
    "${src_dir}/cpu_info.c"
    "${src_dir}/svm.cpp"
    "${src_dir}/darray.c"
    "${src_dir}/libvmaf.cpp"
    "${src_dir}/vmaf.cpp"
    $<TARGET_OBJECTS:convolution_and_psnr_avx>
    $<TARGET_OBJECTS:libptools>
    $<TARGET_OBJECTS:libvmaf_feature>)

target_add_vmaf_common_flags(vmaf CCXX)

target_include_directories(vmaf PRIVATE
    ${vmaf_include}
    ${stdatomic_dependency})
target_link_libraries(vmaf Threads::Threads)

set_target_properties(vmaf PROPERTIES SOVERSION ${vmaf_soversion})
set_target_properties(vmaf PROPERTIES VERSION ${vmaf_soname_version})
install(TARGETS vmaf
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/libvmaf)

add_library(libvmaf_rc_feature OBJECT
    "${feature_src_dir}/picture_copy.c"
    "${feature_src_dir}/integer_psnr.c"
    "${feature_src_dir}/feature_extractor.c"
    "${feature_src_dir}/alias.c"
    "${feature_src_dir}/float_adm.c"
    "${feature_src_dir}/feature_collector.c"
    "${feature_src_dir}/float_psnr.c"
    "${feature_src_dir}/float_motion.c"
    "${feature_src_dir}/float_ssim.c"
    "${feature_src_dir}/float_ms_ssim.c"
    "${feature_src_dir}/float_vif.c"
    "${feature_src_dir}/integer_ssim.c"
    "${feature_src_dir}/float_moment.c")

target_include_directories(libvmaf_rc_feature PRIVATE
    "${opencontainers_include}"
    "${libvmaf_inc}"
    "${src_dir}"
    "${feature_src_dir}"
    "${feature_src_dir}/common"
    "${ptools_path}"
    "${stdatomic_dependency}")

add_library(vmaf_rc
    "${src_dir}/libvmaf.rc.c"
    "${src_dir}/predict.c"
    "${src_dir}/model.c"
    "${src_dir}/unpickle.cpp"
    "${src_dir}/svm.cpp"
    "${src_dir}/picture.c"
    "${src_dir}/mem.c"
    "${src_dir}/picture.c"
    "${src_dir}/output.c"
    "${src_dir}/fex_ctx_vector.c"
    "${src_dir}/thread_pool.c"
    $<TARGET_OBJECTS:convolution_and_psnr_avx>
    $<TARGET_OBJECTS:libptools>
    $<TARGET_OBJECTS:libvmaf_feature>
    $<TARGET_OBJECTS:libvmaf_rc_feature>)

target_include_directories(vmaf_rc PRIVATE
    ${vmaf_include}
    "${libvmaf_inc}"
    ${stdatomic_dependency})

target_add_vmaf_common_flags(vmaf_rc CCXX)

target_link_libraries(vmaf_rc Threads::Threads)

if(LIBM)
    target_link_libraries(vmaf_rc ${LIBM})
endif()

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/libvmaf.pc
"prefix=${CMAKE_INSTALL_PREFIX}
libdir=${CMAKE_INSTALL_FULL_LIBDIR}
includedir=${CMAKE_INSTALL_FULL_INCLUDEDIR}/libvmaf

Name: libvmaf
Description: VMAF, Video Multimethod Assessment Fusion
Version: ${PROJECT_VERSION}
Libs: -L\${libdir} -lvmaf
Libs.private: -lstdc++ -pthread
Cflags: -I\${includedir}
")
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libvmaf.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)