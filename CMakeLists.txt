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

find_package(Threads REQUIRED)
if(WIN32)
    find_package(WebP REQUIRED)
    message(STATUS "libwebp version ${WEBP_VERSION}")
    list(APPEND KWEBP_INCLUDE_DIRS ${WEBP_INCLUDE_DIRS})
    list(APPEND KWEBP_LIBS windowscodecs WebP::webp)
    list(APPEND KWEBP_SOURCE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/win_encode.c")
else()
    if(NOT LIBWEBP_DIR
       OR NOT EXISTS "${LIBWEBP_DIR}/CMakeLists.txt"
       OR NOT EXISTS "${LIBWEBP_DIR}/imageio/image_dec.h")
        message(FATAL_ERROR "LIBWEBP_DIR must point to the libwebp source tree")
    endif()

    find_package(JPEG REQUIRED)
    find_package(PNG REQUIRED)
    find_package(TIFF REQUIRED)
    find_package(GIF)

    set(BUILD_SHARED_LIBS OFF)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    # Keep the image decoder enabled and let it use shared image libraries.
    # libwebp disables TIFF when WEBP_LINK_STATIC is enabled.
    set(WEBP_LINK_STATIC OFF CACHE INTERNAL "Allow shared image libraries")
    set(WEBP_BUILD_CWEBP ON CACHE INTERNAL "Enable libwebp's image decoder target")
    set(WEBP_BUILD_LIBWEBPMUX ON CACHE INTERNAL "Enable libwebp's animation muxer")
    # Exclude libwebp's own tools and install rules from the module build.
    add_subdirectory("${LIBWEBP_DIR}" libwebp EXCLUDE_FROM_ALL)

    list(APPEND KWEBP_SOURCE_FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/src/gifdec.c"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/gif2webp.c")
    list(APPEND KWEBP_INCLUDE_DIRS "${LIBWEBP_DIR}")
    list(APPEND KWEBP_LIBS imagedec webp)

    if(GIF_FOUND)
        add_compile_definitions(WEBP_HAVE_GIF)
        list(APPEND KWEBP_INCLUDE_DIRS ${GIF_INCLUDE_DIR})
        list(APPEND KWEBP_LIBS ${GIF_LIBRARIES} libwebpmux)
    endif()
endif()

add_library(kwebp MODULE ${KWEBP_SOURCE_FILES})
set_target_properties(kwebp PROPERTIES PREFIX "" C_STANDARD 99 C_STANDARD_REQUIRED ON)
target_include_directories(kwebp PRIVATE ${KWEBP_INCLUDE_DIRS})
target_link_libraries(kwebp PRIVATE ${KWEBP_LIBS})

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    # Keep the private codec symbols local even if Kangle has loaded system libwebp.
    set_property(TARGET kwebp APPEND_STRING PROPERTY LINK_FLAGS " -Wl,--exclude-libs,ALL")
endif()

set(CMAKE_INSTALL_PREFIX "${KANGLE_DIR}" CACHE PATH "Install path prefix" FORCE)
install(TARGETS kwebp
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION bin)
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/ext/" DESTINATION ext)

set(CPACK_PACKAGE_NAME "kwebp")
include(CPack)
