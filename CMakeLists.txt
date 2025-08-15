cmake_minimum_required(VERSION 3.15)
project(Yojimbo
        VERSION 1.2.3
        LANGUAGES C CXX
        DESCRIPTION "A network library for client/server games with dedicated servers"
)

# Configuration types
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif ()

# Options
option(YOJIMBO_TESTS "Compile various test programs" OFF)
option(YOJIMBO_INSTALL "Install yojimbo files" ON)

# Global settings
set(CMAKE_CXX_RTTI OFF)

# Set position independent code for shared libraries
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Include GNUInstallDirs for standard installation directories
include(GNUInstallDirs)

# Compiler flags
if (MSVC)
    add_compile_options(/W4 /WX)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /fp:fast")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /fp:fast")
else ()
    add_compile_options(-Wall -Wextra -Werror)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -ffast-math")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -ffast-math")
endif ()

# Configuration-specific definitions
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-DYOJIMBO_DEBUG -DNETCODE_DEBUG -DRELIABLE_DEBUG)
elseif (CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions(-DYOJIMBO_RELEASE -DNETCODE_RELEASE -DRELIABLE_RELEASE)
endif ()

# sodium-builtin library
add_library(sodium-builtin)
target_sources(sodium-builtin PRIVATE sodium/dummy.c)
target_include_directories(sodium-builtin PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/sodium/>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/sodium>
)

if (WIN32)
    file(GLOB_RECURSE SODIUM_SOURCES "sodium/*.c")
    file(GLOB_RECURSE SODIUM_HEADERS "sodium/*.h")
    target_sources(sodium-builtin PRIVATE ${SODIUM_SOURCES} ${SODIUM_HEADERS})
else ()
    # Check if we're on x64, avx, or avx2 platforms
    if (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|amd64|AMD64")
        file(GLOB_RECURSE SODIUM_ASM "sodium/*.S")
        target_sources(sodium-builtin PRIVATE ${SODIUM_ASM})
    endif ()

    # Add compiler flags for non-MSVC builds
    target_compile_options(sodium-builtin PRIVATE
            -Wno-unused-parameter
            -Wno-unused-function
            -Wno-unknown-pragmas
            -Wno-unused-variable
            -Wno-type-limits
    )
endif ()

# netcode library
add_library(netcode)
target_sources(netcode PRIVATE netcode/netcode.c netcode/netcode.h)
target_compile_definitions(netcode PRIVATE NETCODE_ENABLE_TESTS=1)
target_link_libraries(netcode PRIVATE sodium-builtin)
target_include_directories(netcode PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/netcode/>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/netcode>
)

# reliable library
add_library(reliable)
target_sources(reliable PRIVATE reliable/reliable.c reliable/reliable.h)
target_compile_definitions(reliable PRIVATE RELIABLE_ENABLE_TESTS=1)
target_include_directories(reliable PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/reliable/>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/reliable>
)

# tlsf library
add_library(tlsf)
target_sources(tlsf PRIVATE tlsf/tlsf.c tlsf/tlsf.h)
target_include_directories(tlsf PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/tlsf/>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/tlsf>
)

# yojimbo main library
add_library(yojimbo STATIC)
add_library(yojimbo::yojimbo ALIAS yojimbo)

file(GLOB YOJIMBO_HEADERS "include/*.h")
file(GLOB YOJIMBO_SOURCES "source/*.cpp")

target_include_directories(yojimbo PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/serialize/>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/yojimbo>
)
target_sources(yojimbo PRIVATE ${YOJIMBO_HEADERS} ${YOJIMBO_SOURCES})
target_link_libraries(yojimbo PUBLIC
        sodium-builtin
        netcode
        reliable
        tlsf
)

set_target_properties(yojimbo
        PROPERTIES
        CMAKE_CXX_STANDARD 11
        CMAKE_CXX_STANDARD_REQUIRED ON
        VERSION ${PROJECT_VERSION}
        SOVERSION ${PROJECT_VERSION_MAJOR}
        EXPORT_NAME yojimbo
)

# Tests (if enabled)
if (YOJIMBO_TESTS)
    # Helper function to create executables with proper linking
    function(add_yojimbo_executable target_name source_file)
        add_executable(${target_name} ${source_file} shared.h)

        if (WIN32)
            target_link_libraries(${target_name}
                    yojimbo::yojimbo
            )
        else ()
            target_link_libraries(${target_name}
                    yojimbo::yojimbo
            )
            # Add homebrew lib directory for macOS
            if (APPLE)
                target_link_directories(${target_name} PRIVATE /opt/homebrew/lib)
            endif ()
        endif ()
    endfunction()

    # client executable
    add_yojimbo_executable(client client.cpp)

    # server executable
    add_yojimbo_executable(server server.cpp)

    # loopback executable
    add_yojimbo_executable(loopback loopback.cpp)

    # soak executable
    add_yojimbo_executable(soak soak.cpp)

    # test executable
    add_executable(test test.cpp)
    target_compile_definitions(test PRIVATE SERIALIZE_ENABLE_TESTS=1)

    if (WIN32)
        target_link_libraries(test
                yojimbo::yojimbo
        )
    else ()
        target_link_libraries(test
                yojimbo::yojimbo
        )
        if (APPLE)
            target_link_directories(test PRIVATE /opt/homebrew/lib)
        endif ()
    endif ()
endif ()

# Installation
if (YOJIMBO_INSTALL)
    # Install libraries
    install(TARGETS yojimbo sodium-builtin netcode reliable tlsf
            EXPORT yojimboTargets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo
    )

    # Install headers
    install(FILES ${YOJIMBO_HEADERS}
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo
    )

    # Install dependency headers
    install(DIRECTORY netcode/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/netcode
            FILES_MATCHING PATTERN "*.h"
    )

    install(DIRECTORY reliable/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/reliable
            FILES_MATCHING PATTERN "*.h"
    )

    install(DIRECTORY tlsf/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/tlsf
            FILES_MATCHING PATTERN "*.h"
    )

    install(DIRECTORY sodium/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/sodium
            FILES_MATCHING PATTERN "*.h"
    )

    install(DIRECTORY serialize/
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/yojimbo/serialize
            FILES_MATCHING PATTERN "*.h"
    )

    # Install test executables (if built)
    if (YOJIMBO_TESTS)
        install(TARGETS client server loopback soak test
                DESTINATION ${CMAKE_INSTALL_BINDIR}
        )
    endif ()

    # Create and install CMake config files
    include(CMakePackageConfigHelpers)

    # Generate the config file
    configure_package_config_file(
            "${CMAKE_CURRENT_SOURCE_DIR}/cmake/yojimboConfig.cmake.in"
            "${CMAKE_CURRENT_BINARY_DIR}/yojimboConfig.cmake"
            INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/yojimbo
            PATH_VARS CMAKE_INSTALL_INCLUDEDIR CMAKE_INSTALL_LIBDIR
    )

    # Generate the version file
    write_basic_package_version_file(
            "${CMAKE_CURRENT_BINARY_DIR}/yojimboConfigVersion.cmake"
            VERSION ${PROJECT_VERSION}
            COMPATIBILITY SameMajorVersion
    )

    # Install the config files
    install(FILES
            "${CMAKE_CURRENT_BINARY_DIR}/yojimboConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/yojimboConfigVersion.cmake"
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/yojimbo
    )

    # Install the targets file
    install(EXPORT yojimboTargets
            FILE yojimboTargets.cmake
            NAMESPACE yojimbo::
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/yojimbo
    )

    # Create pkg-config file
    configure_file(
            "${CMAKE_CURRENT_SOURCE_DIR}/cmake/yojimbo.pc.in"
            "${CMAKE_CURRENT_BINARY_DIR}/yojimbo.pc"
            @ONLY
    )

    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/yojimbo.pc"
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif ()

# CPack configuration for creating packages
set(CPACK_PACKAGE_NAME "yojimbo")
set(CPACK_PACKAGE_VENDOR "Network Next")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "A network library for client/server games with dedicated servers")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_PACKAGE_INSTALL_DIRECTORY "yojimbo")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENCE")
include(CPack)