# Copyright 2015-2023 The Khronos Group Inc.
# Copyright 2022-2023 RasterGrid Kft.
# Copyright 2026 Binomial LLC - basisu v2.1 integration
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.22)

# N.B. Beware of using cmake_print_variables for function dependent
# variables, e.g. ARGN and CMAKE_CURRENT_FUNCTION_LIST_DIR.
include(CMakePrintHelpers)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/modules/")

include(cmake/version.cmake)

if(POLICY CMP0149)
    # Ignore CMAKE_SYSTEM_VERSION and select either latest available
    # Windows SDK or that specified in WindowsSDKVersion environment
    # variable  Needed because OLD policy picks SDK that matches
    # system version, CI uses Windows Server 2022 and its matching
    # SDK lacks the arm64 glu32.lib which causes builds to fail.
    # MUST be set before project() command.
    cmake_policy(SET CMP0149 NEW)
endif()

# N.B IOS and, in the Darwin case, CMAKE_SYSTEM_NAME are not set
# until after the project() command. The latter is most strange.
if(APPLE)
    if(CMAKE_SYSTEM_NAME STREQUAL "iOS" OR CMAKE_SYSTEM_NAME STREQUAL "tvOS" OR CMAKE_SYSTEM_NAME STREQUAL "visionOS")
        set( APPLE_LOCKED_OS ON )
    else()
        set( APPLE_MAC_OS ON )
    endif()
endif()

include(CMakeDependentOption)

include(cmake/codesign.cmake) # Needs APPLE_LOCKED_OS value.
include(cmake/cputypetest.cmake)

# OPTIONS

option( KTX_WERROR "Make all warnings in KTX code into errors." OFF)

if(POLICY CMP0127)
    cmake_policy(SET CMP0127 NEW)
endif()

CMAKE_DEPENDENT_OPTION( KTX_EMBED_BITCODE
    "Embed bitcode in binaries."
    OFF
    "APPLE AND APPLE_LOCKED_OS"
    OFF
)

# Platform specific settings

set(bitness 64)

if(APPLE)
    # Deployment targets.
    # MUST be set before project() else it is ignored.
    if(APPLE_MAC_OS)
        set(CMAKE_OSX_DEPLOYMENT_TARGET "13.3" CACHE STRING "macOS Deployment Target")
    elseif(CMAKE_SYSTEM_NAME STREQUAL "iOS" OR CMAKE_SYSTEM_NAME STREQUAL "tvOS")
        set(CMAKE_OSX_DEPLOYMENT_TARGET "12.0" CACHE STRING "iOS/tvOS Deployment Target")
        set(CMAKE_XCODE_ATTRIBUTE_ONLY_ACTIVE_ARCH NO)
    elseif(CMAKE_SYSTEM_NAME STREQUAL "visionOS")
        set(CMAKE_OSX_DEPLOYMENT_TARGET "1.0" CACHE STRING "visionOS Deployment Target")
    endif()
endif()

# After most option settings so settings can be used to affect vcpkg.
project(KTX-Software
    VERSION ${KTX_VERSION}
    DESCRIPTION "Libraries and tools to create and read KTX image texture files."
)

include(GNUInstallDirs) # Must be after project.

set_target_processor_type(CPU_ARCHITECTURE) # Must be after project.
if (CPU_ARCHITECTURE STREQUAL x86)
    message(FATAL_ERROR "This project cannot be built for x86 cpu.")
endif()

if(UNIX AND NOT APPLE AND NOT ANDROID)
    set(LINUX TRUE)
endif()

if(NOT BUILD_SHARED_LIBS)
    set(LIB_TYPE STATIC)
else()
    set(LIB_TYPE SHARED)
endif()

option(BUILD_SHARED_LIBS "Create shared libraries (static otherwise)." ON)

# Global compile & link options including optimization flags
if(MSVC)
    add_compile_options( /W4;$<$<BOOL:${KTX_WERROR}>:/WX> )
    add_compile_options( $<IF:$<CONFIG:Debug>,/Gz,/O2> )
    # Enable UTF-8 support
    add_compile_options( $<$<C_COMPILER_ID:MSVC>:/utf-8> )
    add_compile_options( $<$<CXX_COMPILER_ID:MSVC>:/utf-8> )
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU"
       OR ${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
    add_compile_options( -Wall -Wextra $<$<BOOL:${KTX_WERROR}>:-Werror>)
    add_compile_options( $<IF:$<CONFIG:Debug>,-O0$<SEMICOLON>-g,-O3> )
    add_link_options( $<IF:$<CONFIG:Debug>,-g,-O3> )
else()
    message(FATAL_ERROR "${CMAKE_CXX_COMPILER_ID} not yet supported.")
endif()

# To improve output determinism, enable precise floating point operations globally
include(cmake/fp-settings.cmake)
get_fp_compile_options(fp_options)
add_compile_options( ${fp_options} )

# Set output directories
set(KTX_BUILD_DIR "${CMAKE_BINARY_DIR}")

if(APPLE OR LINUX OR WIN32)
    # - Normalize the output directory so it is the same regardless of
    #   single or multi-config generator. The generator expression forces
    #   multi-configuration generators to take the exact path.
    # - Build all executables into a common output directory.
    # - On Windows, build dlls into the same directory as the executables.
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY $<1:${KTX_BUILD_DIR}/$<CONFIG>>)

    # Build shared libs into the same directory as executables so that
    # INSTALL RPATH is functional in the build directory as well.
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY $<1:${KTX_BUILD_DIR}/$<CONFIG>>)

    # For GNU/Linux and macOS BUILD_WITH_INSTALL_RPATH is necessary to have
    # the install rpath set during build.
    set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
endif()

# Ensure the following are set as KTX-Software needs and hide them from users.
set( LIBKTX_WERROR ${KTX_WERROR} )
set( LIBKTX_EMBED_BITCODE ${KTX_EMBED_BITCODE} )
set( LIBKTX_VERSION_FULL ON )
# Want to expose this one to users so must be a cache variable. Note
# that this is a different default from the one set in the libktx project.
set( LIBKTX_VERSION_READ_ONLY ON CACHE BOOL "Build the read-only library")
add_subdirectory(lib)

create_version_file()

add_library( objUtil STATIC
    utils/argparser.cpp
    utils/argparser.h
    utils/sbufstream.h
    utils/stdafx.h
    utils/platform_utils.h
    utils/unused.h
    )
target_include_directories(
    objUtil
PUBLIC
    utils
)
target_compile_features(
    objUtil
PUBLIC
    cxx_std_11
)

# In C++ apps that use statically linked libraries all compilation units must
# be compiled with matching symbol visibility settings to avoid warnings from
# clang. See original CMakeLists.txt for full rationale.
set(STATIC_APP_LIB_SYMBOL_VISIBILITY hidden)
set_target_properties(objUtil PROPERTIES
    CXX_VISIBILITY_PRESET ${STATIC_APP_LIB_SYMBOL_VISIBILITY}
)
if(NOT BUILD_SHARED_LIBS AND
   (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang"))
    set_source_files_properties(
        lib/astc_encode.cpp
        PROPERTIES COMPILE_OPTIONS "-fvisibility=hidden"
    )
endif()

add_subdirectory(interface/basisu_c_binding)

# fmt and cxxopts: always needed for the tools
if(NOT TARGET fmt::fmt)
    set(BUILD_SHARED_LIBS OFF)
    set(FMT_INSTALL OFF)
    set(FMT_SYSTEM_HEADERS ON)
    add_subdirectory(external/fmt)
    set(BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS_RESET})
endif()
if(NOT TARGET cxxopts::cxxopts)
    add_subdirectory(external/cxxopts)
endif()

# Tools (btx)
add_subdirectory(tools)

# CPack

include(CPackComponent)

set(CPACK_PACKAGE_NAME "KTX-Software")
set(CPACK_PACKAGE_VENDOR "Khronos Group")
set(CPACK_PACKAGE_HOMEPAGE_URL "https://github.khronos.org/KTX-Software")
set(CPACK_PACKAGE_CONTACT "khronos@callow.im" )

set(CPACK_RESOURCE_FILE_WELCOME "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Welcome.rtf")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/cmake/ReadMe.rtf")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/cmake/License.rtf")

# Custom package file name
if( APPLE AND CMAKE_OSX_ARCHITECTURES )
    list(LENGTH CMAKE_OSX_ARCHITECTURES archs_len)
    list(GET CMAKE_OSX_ARCHITECTURES 0 arch0)
    if( ${archs_len} GREATER 1 OR ${arch0} STREQUAL "$(ARCHS_STANDARD)" )
        set(processor_name "universal")
    else()
        set(processor_name ${arch0})
    endif()
elseif( CMAKE_CXX_COMPILER_ARCHITECTURE_ID )
    set(processor_name ${CMAKE_CXX_COMPILER_ARCHITECTURE_ID})
elseif( CMAKE_SYSTEM_PROCESSOR )
    if( ${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64" )
        set(processor_name "arm64")
    else()
        set(processor_name ${CMAKE_SYSTEM_PROCESSOR})
    endif()
elseif( APPLE_LOCKED_OS )
    set(processor_name "arm64")
endif()
string(TOLOWER "${processor_name}" processor_name)
set(CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${KTX_VERSION_FULL}-${CMAKE_SYSTEM_NAME}-${processor_name}")

if(APPLE)
    if(APPLE_MAC_OS)
        install(FILES tools/package/mac/ktx-uninstall
            DESTINATION ${CMAKE_INSTALL_BINDIR}
            PERMISSIONS OWNER_READ OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
            COMPONENT tools
        )
        set(CPACK_GENERATOR productbuild)
        set(CPACK_PACKAGE_NAME "ktx")
        set(CPACK_PACKAGE_VENDOR "khronos")
        set(CPACK_PACKAGING_INSTALL_PREFIX "/usr/local")
        set(CPACK_PRODUCTBUILD_IDENTITY_NAME ${PRODUCTBUILD_IDENTITY_NAME})
        set(CPACK_PRODUCTBUILD_KEYCHAIN_PATH ${PRODUCTBUILD_KEYCHAIN_PATH})
        set(CPACK_PRODUCTBUILD_RESOURCES_DIR "${CMAKE_CURRENT_SOURCE_DIR}/tools/package/mac/Resources")
        set(CPACK_PRODUCTBUILD_BACKGROUND "ktx_logo_190_xp.png")
        set(CPACK_PRODUCTBUILD_BACKGROUND_MIME_TYPE "image/png")
        set(CPACK_PRODUCTBUILD_BACKGROUND_ALIGNMENT "bottomleft")
    else()
        set(CPACK_GENERATOR ZIP)
        set(CPACK_ARCHIVE_KTX_FILE_NAME "${CPACK_PACKAGE_FILE_NAME}")
        set(CPACK_ARCHIVE_COMPONENT_INSTALL OFF)
        set(CPACK_PACKAGE_CHECKSUM SHA1)
    endif()
elseif(LINUX)
    set(CPACK_GENERATOR DEB RPM TBZ2)
    set(CPACK_PACKAGE_CHECKSUM SHA1)
    set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/cmake/ReadMe.txt")
    set(CPACK_RPM_DEFAULT_DIR_PERMISSIONS
        OWNER_READ OWNER_WRITE OWNER_EXECUTE
        GROUP_READ GROUP_EXECUTE
        WORLD_READ WORLD_EXECUTE)
elseif(WIN32)
    set(CPACK_GENERATOR "NSIS")
    set(CPACK_PACKAGE_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/win\\\\ktx_logo_200_bmp3.bmp")
    set(CPACK_NSIS_MUI_ICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/win/ktx_app.ico")
    set(CPACK_NSIS_MUI_UNIICON "${CMAKE_CURRENT_SOURCE_DIR}/icons/win/ktx_app.ico")
    set(CPACK_NSIS_INSTALLED_ICON_NAME uninstall.exe)
    set(CPACK_NSIS_MANIFEST_DPI_AWARE ON)
    set(CPACK_NSIS_URL_INFO_ABOUT ${CPACK_PACKAGE_HOMEPAGE_URL})
    set(CPACK_NSIS_CONTACT ${CPACK_PACKAGE_CONTACT})
    set(CPACK_NSIS_MODIFY_PATH ON)
    set(CPACK_NSIS_UNINSTALL_NAME "Uninstall")
    set(CPACK_PACKAGE_INSTALL_DIRECTORY "KTX-Software")
    if(${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.20")
        set(CPACK_NSIS_BRANDING_TEXT "KTX for Windows")
    endif()
    if (CODE_SIGN_KEY_VAULT)
        set_nsis_installer_codesign_cmd()
    else()
        set(CPACK_PACKAGE_CHECKSUM SHA1)
    endif()
else()
    set(CPACK_PACKAGE_CHECKSUM SHA1)
endif()

cpack_add_component(library
    DISPLAY_NAME "Library"
    DESCRIPTION "Main KTX library."
    REQUIRED
)
cpack_add_component(tools
    DISPLAY_NAME "Command line tools"
    DESCRIPTION "Command line tools for creating, converting and inspecting KTX files."
    DEPENDS library
)
cpack_add_component(dev
    DISPLAY_NAME "Development"
    DESCRIPTION "Additional resources for development (header files and documentation)."
    DEPENDS library
    DISABLED
)

set(CPACK_COMPONENTS_ALL
    library
    dev
    tools
)

include(CPack)

# vim:ai:ts=4:sts=2:sw=2:expandtab