cmake_minimum_required(VERSION 3.12)
project(kwebp LANGUAGES C)

set(KANGLE_DIR "" CACHE PATH "Path to the installed kangle directory")
set(KANGLE_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../kangle" CACHE PATH
    "Path to the kangle source tree (used when installed headers are unavailable)")
set(LIBWEBP_DIR "" CACHE PATH "Path to the libwebp source tree")

if(NOT KANGLE_DIR)
    message(FATAL_ERROR
        "KANGLE_DIR is required and must point to the installed kangle directory")
endif()
if(NOT IS_DIRECTORY "${KANGLE_DIR}")
    message(FATAL_ERROR "KANGLE_DIR does not exist: ${KANGLE_DIR}")
endif()

set(KANGLE_INCLUDE_DIRS)
if(EXISTS "${KANGLE_DIR}/include/ksapi.h"
   AND EXISTS "${KANGLE_DIR}/include/kasync_config.h"
   AND EXISTS "${KANGLE_DIR}/include/khttp.h")
    list(APPEND KANGLE_INCLUDE_DIRS "${KANGLE_DIR}/include")
else()
    if(NOT EXISTS "${KANGLE_SOURCE_DIR}/include/ksapi.h"
       OR NOT EXISTS "${KANGLE_SOURCE_DIR}/kasync/include/kasync_config.h.in"
       OR NOT EXISTS "${KANGLE_SOURCE_DIR}/khttpd/include/khttp.h.in")
        message(FATAL_ERROR
            "Cannot find kangle headers. Install the current kangle version with its "
            "development headers, or set KANGLE_SOURCE_DIR to the kangle source tree.")
    endif()

    set(GENERATED_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
    file(MAKE_DIRECTORY "${GENERATED_INCLUDE_DIR}")
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set(LINUX ON)
    elseif(CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
        set(FREEBSD ON)
    elseif(APPLE)
        set(DARWIN ON)
    endif()
    configure_file(
        "${KANGLE_SOURCE_DIR}/kasync/include/kasync_config.h.in"
        "${GENERATED_INCLUDE_DIR}/kasync_config.h")
    configure_file(
        "${KANGLE_SOURCE_DIR}/khttpd/include/khttp.h.in"
        "${GENERATED_INCLUDE_DIR}/khttp.h")
    list(APPEND KANGLE_INCLUDE_DIRS
        "${GENERATED_INCLUDE_DIR}"
        "${KANGLE_SOURCE_DIR}/include"
        "${KANGLE_SOURCE_DIR}/kasync/include"
        "${KANGLE_SOURCE_DIR}/khttpd/include")
endif()

set(KWEBP_SOURCE_FILES
    "${CMAKE_CURRENT_SOURCE_DIR}/src/access.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/main.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/push_gate.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/upstream.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/vary.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/webp_encode.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/unix_encode.c")
set(KWEBP_INCLUDE_DIRS ${KANGLE_INCLUDE_DIRS})
set(KWEBP_LIBS)

if(WIN32)
    list(APPEND KWEBP_LIBS windowscodecs)
    list(APPEND KWEBP_SOURCE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/win_encode.c")
else()
    if(NOT LIBWEBP_DIR
       OR NOT EXISTS "${LIBWEBP_DIR}/src/mux/animi.h"
       OR NOT EXISTS "${LIBWEBP_DIR}/imageio/image_dec.h")
        message(FATAL_ERROR
            "LIBWEBP_DIR must point to the libwebp source tree used to build libwebp")
    endif()
    if(NOT IS_DIRECTORY "${LIBWEBP_DIR}/build")
        message(FATAL_ERROR
            "Cannot find ${LIBWEBP_DIR}/build. Build libwebp with CMake first.")
    endif()

    list(APPEND KWEBP_SOURCE_FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/src/gifdec.c"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/gif2webp.c")
    list(APPEND KWEBP_INCLUDE_DIRS "${LIBWEBP_DIR}")
    list(APPEND KWEBP_LIBS imagedec jpeg png tiff webpdemux imageioutil)
    link_directories("${LIBWEBP_DIR}/build")

    find_package(GIF)
    if(GIF_FOUND)
        add_compile_definitions(WEBP_HAVE_GIF)
        list(APPEND KWEBP_INCLUDE_DIRS ${GIF_INCLUDE_DIR})
        list(APPEND KWEBP_LIBS ${GIF_LIBRARIES} webpmux)
    endif()
endif()

find_package(Threads REQUIRED)
find_package(WebP REQUIRED)
message(STATUS "libwebp version ${WEBP_VERSION}")
list(APPEND KWEBP_LIBS WebP::webp)

add_library(kwebp MODULE ${KWEBP_SOURCE_FILES})
set_target_properties(kwebp PROPERTIES PREFIX "" C_STANDARD 99 C_STANDARD_REQUIRED ON)
target_include_directories(kwebp PRIVATE ${KWEBP_INCLUDE_DIRS} ${WEBP_INCLUDE_DIRS})
target_link_libraries(kwebp PRIVATE ${KWEBP_LIBS})

set(CMAKE_INSTALL_PREFIX "${KANGLE_DIR}" CACHE PATH "Install path prefix" FORCE)
install(TARGETS kwebp
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION bin)
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/ext/" DESTINATION ext)

set(CPACK_PACKAGE_NAME "kwebp")
include(CPack)
