cmake_minimum_required(VERSION 3.16)
project(duckstation C CXX)

# Policy settings.
cmake_policy(SET CMP0069 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0069 NEW)

message("CMake Version: ${CMAKE_VERSION}")
message("CMake System Name: ${CMAKE_SYSTEM_NAME}")
message("Build Type: ${CMAKE_BUILD_TYPE}")

# Pull in modules.
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeModules/")

# Platform detection.
if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
  set(LINUX TRUE)
  set(SUPPORTS_X11 TRUE)
  set(SUPPORTS_WAYLAND TRUE)
elseif(${CMAKE_SYSTEM_NAME} STREQUAL "FreeBSD")
  set(FREEBSD TRUE)
  set(SUPPORTS_X11 TRUE)
endif()

# Set minimum OS version for macOS. 10.14 should work.
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.14.0" CACHE STRING "")

# Global options.
if(NOT ANDROID)
  option(BUILD_NOGUI_FRONTEND "Build the NoGUI frontend" OFF)
  option(BUILD_QT_FRONTEND "Build the Qt frontend" ON)
  option(BUILD_REGTEST "Build regression test runner" OFF)
  option(ENABLE_CUBEB "Build with Cubeb audio output" ON)
  option(ENABLE_OPENGL "Build with OpenGL renderer" ON)
  option(ENABLE_VULKAN "Build with Vulkan renderer" ON)
  option(ENABLE_DISCORD_PRESENCE "Build with Discord Rich Presence support" ON)
  option(ENABLE_CHEEVOS "Build with RetroAchievements support" ON)
  option(USE_SDL2 "Link with SDL2 for controller support" ON)
endif()


# OpenGL context creation methods.
if(SUPPORTS_X11)
  option(USE_X11 "Support X11 window system" ON)
endif()
if(SUPPORTS_WAYLAND)
  option(USE_WAYLAND "Support Wayland window system" ON)
endif()
if((LINUX OR FREEBSD) OR ANDROID)
  option(USE_EGL "Support EGL OpenGL context creation" ON)
endif()
if((LINUX OR FREEBSD) AND NOT ANDROID)
  option(USE_DRMKMS "Support DRM/KMS OpenGL contexts" OFF)
  option(USE_FBDEV "Support FBDev OpenGL contexts" OFF)
  option(USE_EVDEV "Support EVDev controller interface" OFF)
  option(USE_DBUS "Enable DBus support for screensaver inhibiting" ON)
endif()

# Force EGL when using Wayland
if(USE_WAYLAND)
  set(USE_EGL ON)
endif()

if(ANDROID)
  if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
  endif()
  set(BUILD_NOGUI_FRONTEND OFF)
  set(BUILD_QT_FRONTEND OFF)
  set(BUILD_REGTEST OFF)
  set(ENABLE_CUBEB OFF)
  set(ENABLE_OPENGL ON)
  set(ENABLE_VULKAN ON)
  set(ENABLE_DISCORD_PRESENCE OFF)
  set(ENABLE_CHEEVOS ON)
  set(USE_SDL2 OFF)
  set(USE_X11 OFF)
  set(USE_WAYLAND OFF)
endif()


# Common include/library directories on Windows.
if(WIN32 AND USE_SDL2)
  set(SDL2_FOUND TRUE)
  set(SDL2_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/include")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(SDL2_LIBRARIES "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/lib64/SDL2.lib")
    set(SDL2MAIN_LIBRARIES "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/lib64/SDL2main.lib")
    set(SDL2_DLL_PATH "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/bin64/SDL2.dll")
    set(Qt6_DIR "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/qt/6.1.0/msvc2019_64/lib/cmake/Qt6")
  else()
    set(SDL2_LIBRARIES "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/lib32/SDL2.lib")
    set(SDL2MAIN_LIBRARIES "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/lib32/SDL2main.lib")
    set(SDL2_DLL_PATH "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/sdl2/bin32/SDL2.dll")
    set(Qt6_DIR "${CMAKE_CURRENT_SOURCE_DIR}/dep/msvc/qt/6.1.0/msvc2019_32/lib/cmake/Qt6")
  endif()
endif()


# Required libraries.
if(NOT ANDROID)
  if(NOT WIN32 AND USE_SDL2)
    find_package(SDL2 REQUIRED)
  endif()
  if(NOT WIN32)
    find_package(CURL REQUIRED)
  endif()
  if(BUILD_QT_FRONTEND)
    find_package(Qt6 COMPONENTS Core Gui Widgets Network LinguistTools REQUIRED)
  endif()
endif()

if(USE_EGL)
  find_package(EGL REQUIRED)
endif()
if(USE_X11)
  find_package(X11 REQUIRED)
  if (NOT X11_Xrandr_FOUND)
    message(FATAL_ERROR "XRandR extension is required")
  endif()
endif()
if(USE_WAYLAND)
  message(STATUS "Wayland support enabled")
endif()
if(USE_DRMKMS AND USE_FBDEV)
  message(FATAL_ERROR "Only one of DRM/KMS and FBDev can be enabled")
endif()
if(USE_DRMKMS)
  find_package(GBM REQUIRED)
  find_package(Libdrm REQUIRED)
  message(STATUS "DRM/KMS support enabled")
endif()
if(USE_FBDEV)
  message(STATUS "FBDev Support enabled")
endif()
if(USE_EVDEV)
  message(STATUS "EVDev Support enabled")
  find_package(LIBEVDEV REQUIRED)
endif()
if(ENABLE_CHEEVOS)
  message(STATUS "RetroAchievements support enabled")
endif()


# Set _DEBUG macro for Debug builds.
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -D_DEBUG")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -D_DEBUG")


# Release build optimizations for MSVC.
if(MSVC)
  add_definitions("/D_CRT_SECURE_NO_WARNINGS")
  foreach(config CMAKE_C_FLAGS CMAKE_CXX_FLAGS)
    # Set warning level 3 instead of 4.
    string(REPLACE "/W3" "/W4" ${config} "${${config}}")

    # Enable intrinsic functions, disable minimal rebuild, UTF-8 source, set __cplusplus version.
    set(${config} "${${config}} /Oi /Gm- /utf-8 /Zc:__cplusplus")
  endforeach()

  # RelWithDebInfo is set to Ob1 instead of Ob2.   
  string(REPLACE "/Ob1" "/Ob2" CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO}")
  string(REPLACE "/Ob1" "/Ob2" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")

  # Disable incremental linking in RelWithDebInfo.
  string(REPLACE "/INCREMENTAL" "/INCREMENTAL:NO" CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO}")

  # COMDAT folding/remove unused functions.
  set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /OPT:REF /OPT:ICF")
  set(CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO} /OPT:REF /OPT:ICF")

  # Enable LTO/LTCG on Release builds.
  if(${CMAKE_BUILD_TYPE} STREQUAL "Release")
    cmake_policy(SET CMP0069 NEW)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT IPO_IS_SUPPORTED)
    if(IPO_IS_SUPPORTED)
      message(STATUS "Enabling LTCG/IPO.")
      set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
    else()
      message(WARNING "LTCG/IPO is not supported, this will make the build slightly slower.")
    endif()
  endif()
endif()


# Default symbol visibility to hidden, that way we don't go through the PLT for intra-library calls.
if(ANDROID)
  cmake_policy(SET CMP0063 NEW)
  set(CMAKE_POLICY_DEFAULT_CMP0063 NEW)
  set(CMAKE_C_VISIBILITY_PRESET hidden)
  set(CMAKE_CXX_VISIBILITY_PRESET hidden)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-semantic-interposition")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-semantic-interposition")
endif()


# Detect C++ version support.
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    include(CheckCXXFlag)
    check_cxx_flag(-Wall COMPILER_SUPPORTS_WALL)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-switch")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-switch")
    if(NOT ANDROID)
      check_cxx_flag(-Wno-class-memaccess COMPILER_SUPPORTS_MEMACCESS)
      check_cxx_flag(-Wno-invalid-offsetof COMPILER_SUPPORTS_OFFSETOF)
    endif()
endif()


# Detect processor type.
if("${CMAKE_OSX_ARCHITECTURES}" STREQUAL "arm64")
  # Cross-compile on macos.
  set(CPU_ARCH "aarch64")
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "amd64" OR
   "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "AMD64")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(CPU_ARCH "x64")
  else()
    # Cross-compiling 32-bit build. 32-bit hosts are not supported.
    set(CPU_ARCH "x86")
  endif()
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "i386" OR
       "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "i686")
  set(CPU_ARCH "x86")
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "aarch64" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "arm64")
  set(CPU_ARCH "aarch64")
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "arm" OR "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7-a" OR
       "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7l")
  set(CPU_ARCH "aarch32")
  if(ANDROID)
    # Force ARM mode, since apparently ANDROID_ARM_MODE isn't working..
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -marm")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -marm")
  else()
    # Enable NEON.
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -marm -march=armv7-a")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -marm -march=armv7-a")
  endif()
else()
  message(FATAL_ERROR "Unknown system processor: ${CMAKE_SYSTEM_PROCESSOR}")
endif()


# We don't need exceptions, disable them to save a bit of code size.
# Except xbyak still uses them :(
if(NOT ("${CPU_ARCH}" STREQUAL "x86" OR "${CPU_ARCH}" STREQUAL "x64"))
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
endif()


# Write binaries to a seperate directory.
if(WIN32)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/bin/${CPU_ARCH}")
else()
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/bin")
endif()

# Needed for Linux - put shared libraries in the binary directory.
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")


# Enable threads everywhere.
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)


# Enable large file support on Linux 32-bit platforms.
# Android is deliberately ommitted here as it didn't support 64-bit ops on files until Android 7/N.
if((LINUX OR FREEBSD) AND (${CPU_ARCH} STREQUAL "x86" OR ${CPU_ARCH} STREQUAL "aarch32"))
  add_definitions("-D_FILE_OFFSET_BITS=64")
endif()


# Recursively include the source tree.
enable_testing()
add_subdirectory(dep)
add_subdirectory(src)

if(ANDROID)
  add_subdirectory(android/app/src/cpp)
endif()
