cmake_minimum_required(VERSION 3.10)

# =================================================================
# Target setup
# =================================================================

# Project languages
if(APPLE)
    project(vmone LANGUAGES C OBJC)
else()
    project(vmone LANGUAGES C)
endif()

# Set the Java directory. Default to $JAVA_HOME
if (NOT DEFINED JAVA)
    if (DEFINED ENV{JAVA_HOME})
        set(JAVA $ENV{JAVA_HOME})
    else ()
        message(FATAL_ERROR "JAVA or JAVA_HOME is not defined.")
    endif ()
endif ()

# Set target. Default to current OS.
if (NOT DEFINED TARGET)
    if (UNIX AND NOT APPLE)
        set(TARGET "linux")
    elseif (APPLE)
        set(TARGET "macos")
    elseif (WIN32)
        set(TARGET "windows")
    endif ()
endif ()

# Set the architecture. Default to current one
if (NOT DEFINED TARGET_ARCH)
    if(${TARGET} STREQUAL "android" OR ${TARGET} STREQUAL "ios")
        set(TARGET_ARCH "aarch64")
    else()
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64")
            set(TARGET_ARCH "x64")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "ARM64")
            set(TARGET_ARCH "aarch64")
        else()
            message(FATAL_ERROR "Unsupported system processor: ${CMAKE_SYSTEM_PROCESSOR}")
        endif()
    endif()
endif()

# =================================================================
# Static library
# =================================================================

# Create the vmone static library
file(GLOB SOURCES "src/*.c")
file(GLOB DARWIN_SOURCES "src/darwin/*.m")
add_library(vmone STATIC ${SOURCES})

# Set target specific options
if (${TARGET} STREQUAL "linux")
    target_include_directories(vmone PRIVATE ${JAVA}/include ${JAVA}/include/linux)
    target_compile_definitions(vmone PRIVATE _GNU_SOURCE)

elseif(${TARGET} STREQUAL "macos")
    target_include_directories(vmone PRIVATE ${JAVA}/include ${JAVA}/include/darwin)
    target_compile_definitions(vmone PRIVATE DARWIN)
    target_sources(vmone PRIVATE ${DARWIN_SOURCES})

elseif(${TARGET} STREQUAL "windows")
    target_include_directories(vmone PRIVATE ${JAVA}/include ${JAVA}/include/win32)
    target_compile_definitions(vmone PRIVATE WIN32)
    target_link_libraries(vmone PRIVATE
            ws2_32)

elseif(${TARGET} STREQUAL "android")
    set(DISTRIBUTION_NAME "android-linux-aarch64")
    target_include_directories(vmone PRIVATE $ENV{ANDROID_NDK_HOME}/include)
    set(CMAKE_C_COMPILER "$ENV{ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/bin/clang")
    target_compile_definitions(vmone PRIVATE ANDROID)
    target_compile_options(vmone PRIVATE
            "-target aarch64-linux-android")

elseif(${TARGET} STREQUAL "ios")
    set(DISTRIBUTION_NAME "ios-macos-aarch64")
    set(CMAKE_C_COMPILER "clang")
    target_compile_options(vmone PRIVATE
            "-arch arm64"
            "-isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk"
            "-miphoneos-version-min=12.0"
            "-fobjc-arc"
            "-O2"
            "-Wall -Wextra"
            "-g")
    target_sources(vmone PRIVATE ${DARWIN_SOURCES})
    target_link_libraries(vmone PRIVATE
            "-framework Foundation")

endif()

# =================================================================
# Release/Distribution setup (combine static lib with libjdk)
# =================================================================

# Download the corresponding static jdk lib
if (NOT DEFINED DISTRIBUTION_NAME)
    set(DISTRIBUTION_NAME "${TARGET}-${TARGET_ARCH}")
endif ()

set(LIBJDK_DIR "${CMAKE_CURRENT_BINARY_DIR}/libjdk")
file(DOWNLOAD # download
        "https://github.com/gluonhq/mobile/releases/latest/download/${DISTRIBUTION_NAME}.zip"
        "${LIBJDK_DIR}/${DISTRIBUTION_NAME}.zip"
        SHOW_PROGRESS
)
execute_process( # unzip
        COMMAND ${CMAKE_COMMAND} -E tar xzf "${LIBJDK_DIR}/${DISTRIBUTION_NAME}.zip"
        WORKING_DIRECTORY "${LIBJDK_DIR}"
)

# Create a single static library that combines the vmone output with the static jdk lib
set(DIST_DIR "${CMAKE_CURRENT_BINARY_DIR}/lib")
file(REMOVE_RECURSE ${DIST_DIR})
file(MAKE_DIRECTORY ${DIST_DIR})
if (UNIX)
    file(REMOVE_RECURSE ${CMAKE_CURRENT_BINARY_DIR}/tmp)
    file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tmp)
    add_custom_command(TARGET vmone POST_BUILD
            COMMAND ar -x ${LIBJDK_DIR}/${DISTRIBUTION_NAME}/libjdk.a && # extract libjdk
            ar -x $<TARGET_FILE:vmone> && # extract vmone
            ar -r ${DIST_DIR}/libvmone.a *.o # combine the extracted .o files into a new library
            COMMENT "Creating combined static library for Unix at ${DIST_DIR}/libvmone.a"
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tmp
    )
elseif (WIN32) # Note: for MSVC. MSYS would use 'ar' instead

    if(${CMAKE_HOST_SYSTEM_NAME} STREQUAL "MSYS")
        message(FATAL_ERROR "Windows builds assume MSVC. MSYS would need to change to use 'ar', similar to unix.")
    endif()
    add_custom_command(TARGET vmone POST_BUILD
            COMMAND lib /OUT:${DIST_DIR}/vmone.lib
            ${LIBJDK_DIR}/${DISTRIBUTION_NAME}/libjdk.lib
            $<TARGET_FILE:vmone>
            COMMENT "Creating combined static library for Windows at ${DIST_DIR}/libvmone.lib"
    )
endif()

# Zip up the release file
add_custom_command(
    TARGET vmone POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E tar "cfv" "${CMAKE_CURRENT_BINARY_DIR}/vmone-${DISTRIBUTION_NAME}.zip" --format=zip
    "${DIST_DIR}"
    COMMENT "Archiving binary files into 'vmone-${DISTRIBUTION_NAME}.zip'"
)