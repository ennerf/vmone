cmake_minimum_required(VERSION 3.10)

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
        set(TARGET "macosx")
    elseif (WIN32)
        set(TARGET "windows")
    endif ()
endif ()

# Create the vmone static library
file(GLOB SOURCES "src/*.c")
file(GLOB DARWIN_SOURCES "src/darwin/*.m")
add_library(vmone STATIC ${SOURCES})

# Set target specific options
if (${TARGET} STREQUAL "linux")
    target_include_directories(vmone PRIVATE ${JAVA}/include ${JAVA}/include/linux)
    target_compile_definitions(vmone PRIVATE _GNU_SOURCE)

elseif(${TARGET} STREQUAL "macosx")
    target_include_directories(vmone PRIVATE ${JAVA}/include ${JAVA}/include/darwin)
    target_compile_definitions(vmone PRIVATE DARWIN)
    target_sources(vmone PRIVATE ${DARWIN_SOURCES})

elseif(${TARGET} STREQUAL "windows")
    target_include_directories(vmone PRIVATE ${JAVA}/include ${JAVA}/include/win32)
    target_compile_definitions(vmone PRIVATE WIN32)
    target_link_libraries(vmone PRIVATE
            ws2_3)

elseif(${TARGET} STREQUAL "android")
    target_include_directories(vmone PRIVATE $ENV{ANDROID_NDK_HOME}/include)
    set(CMAKE_C_COMPILER "$ENV{ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/linux-x86_64/bin/clang")
    target_compile_definitions(vmone PRIVATE ANDROID)
    target_compile_options(vmone PRIVATE
            "-target aarch64-linux-android")

elseif(${TARGET} STREQUAL "ios")
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
