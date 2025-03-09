cmake_minimum_required(VERSION 3.31)
project(ffmpeg)



set(FFMPEG_PATH ${CMAKE_BINARY_DIR}/ffmpeg)
make_directory(${FFMPEG_PATH})

add_custom_command(
    OUTPUT ${FFMPEG_PATH}/config.h
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/FFmpeg/configure
        --disable-libdrm
        --disable-vaapi
        --disable-vdpau
        --disable-zlib
        --disable-lzma
        --enable-pic
        --disable-asm
        --extra-cflags="--target=${CMAKE_SYSTEM_PROCESSOR}-none-linux-android${ANDROID_NATIVE_API_LEVEL} -fpic"
        --extra-ldflags="--target=${CMAKE_SYSTEM_PROCESSOR}-none-linux-android${ANDROID_NATIVE_API_LEVEL}"
        --cc="${CMAKE_C_COMPILER}"
        --arch="${CMAKE_SYSTEM_PROCESSOR}"
        --ld="${CMAKE_C_COMPILER}"
        --target_os=android
        --enable-cross-compile
    COMMENT "Configuring FFmpeg..."
    WORKING_DIRECTORY ${FFMPEG_PATH}
)
add_custom_target(ffmpeg-configure DEPENDS ${FFMPEG_PATH}/config.h)
add_custom_target(ffmpeg-build-all)

function(import_ffmpeg_library name)
    if (${CMAKE_GENERATOR} STREQUAL "Unix Makefiles")
        set(MAKE_COMMAND $(MAKE) -j$(nproc))
    elseif (${CMAKE_GENERATOR} STREQUAL "Ninja")
        set(MAKE_COMMAND make -j$$(nproc))
    else()
        set(MAKE_COMMAND make)
    endif()

    add_custom_command(
        OUTPUT "${FFMPEG_PATH}/lib${name}/lib${name}.a"
        COMMAND ${MAKE_COMMAND} -C ${FFMPEG_PATH} "lib${name}/lib${name}.a"
        COMMENT "Building lib${name}/lib${name}.a"
        DEPENDS ffmpeg-configure
        WORKING_DIRECTORY ${FFMPEG_PATH}
    )

    add_custom_target(ffmpeg-build-${name} DEPENDS "${FFMPEG_PATH}/lib${name}/lib${name}.a")

    add_library(ffmpeg::${name} STATIC IMPORTED GLOBAL)
    set_property(TARGET ffmpeg::${name} PROPERTY IMPORTED_LOCATION "${FFMPEG_PATH}/lib${name}/lib${name}.a")
    set_property(TARGET ffmpeg::${name} PROPERTY INTERFACE_INCLUDE_DIRECTORIES "${FFMPEG_PATH}")
    add_dependencies(ffmpeg::${name} ffmpeg-build-${name})

    add_dependencies(ffmpeg-build-all ffmpeg-build-${name})
endfunction()

import_ffmpeg_library(avcodec)
import_ffmpeg_library(avformat)
import_ffmpeg_library(avfilter)
import_ffmpeg_library(avdevice)
import_ffmpeg_library(avutil)
import_ffmpeg_library(swscale)
import_ffmpeg_library(swresample)
# import_ffmpeg_library(postproc)

