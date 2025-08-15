cmake_minimum_required(VERSION 3.15)
project(Yojimbo LANGUAGES C CXX)

# Configuration types
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()

# Global settings
set(CMAKE_CXX_RTTI OFF)
if(NOT WIN32)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
endif()

# Compiler flags
if(MSVC)
    add_compile_options(/W4 /WX)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /fp:fast")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /fp:fast")
else()
    add_compile_options(-Wall -Wextra -Werror)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -ffast-math")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -ffast-math")
endif()

# Include directories
include_directories(. include sodium tlsf netcode reliable serialize)

# Configuration-specific definitions
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-DYOJIMBO_DEBUG -DNETCODE_DEBUG -DRELIABLE_DEBUG)
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions(-DYOJIMBO_RELEASE -DNETCODE_RELEASE -DRELIABLE_RELEASE)
endif()

# sodium-builtin static library
add_library(sodium-builtin STATIC)
target_sources(sodium-builtin PRIVATE sodium/dummy.c)

if(WIN32)
    file(GLOB_RECURSE SODIUM_SOURCES "sodium/*.c")
    file(GLOB_RECURSE SODIUM_HEADERS "sodium/*.h")
    target_sources(sodium-builtin PRIVATE ${SODIUM_SOURCES} ${SODIUM_HEADERS})
else()
    # Check if we're on x64, avx, or avx2 platforms
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|amd64|AMD64")
        file(GLOB_RECURSE SODIUM_ASM "sodium/*.S")
        target_sources(sodium-builtin PRIVATE ${SODIUM_ASM})
    endif()

    # Add compiler flags for non-MSVC builds
    target_compile_options(sodium-builtin PRIVATE
            -Wno-unused-parameter
            -Wno-unused-function
            -Wno-unknown-pragmas
            -Wno-unused-variable
            -Wno-type-limits
    )
endif()

# netcode static library
add_library(netcode STATIC)
target_sources(netcode PRIVATE netcode/netcode.c netcode/netcode.h)
target_compile_definitions(netcode PRIVATE NETCODE_ENABLE_TESTS=1)

# reliable static library
add_library(reliable STATIC)
target_sources(reliable PRIVATE reliable/reliable.c reliable/reliable.h)
target_compile_definitions(reliable PRIVATE RELIABLE_ENABLE_TESTS=1)

# tlsf static library
add_library(tlsf STATIC)
target_sources(tlsf PRIVATE tlsf/tlsf.c tlsf/tlsf.h)

# yojimbo static library
add_library(yojimbo STATIC)
add_library(yojimbo::yojimbo ALIAS yojimbo)

file(GLOB YOJIMBO_HEADERS "include/*.h")
file(GLOB YOJIMBO_SOURCES "source/*.cpp")
target_sources(yojimbo PRIVATE ${YOJIMBO_HEADERS} ${YOJIMBO_SOURCES})

set_target_properties(yojimbo
        PROPERTIES
        CMAKE_CXX_STANDARD 11
        CMAKE_CXX_STANDARD_REQUIRED ON
)

# Helper function to create executables with proper linking
function(add_yojimbo_executable target_name source_file)
    add_executable(${target_name} ${source_file} shared.h)

    if(WIN32)
        target_link_libraries(${target_name}
                yojimbo
                sodium-builtin
                tlsf
                netcode
                reliable
        )
    else()
        target_link_libraries(${target_name}
                yojimbo
                sodium
                tlsf
                netcode
                reliable
        )
        # Add homebrew lib directory for macOS
        if(APPLE)
            target_link_directories(${target_name} PRIVATE /opt/homebrew/lib)
        endif()
    endif()
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

if(WIN32)
    target_link_libraries(test
            yojimbo
            sodium-builtin
            tlsf
            netcode
            reliable
    )
else()
    target_link_libraries(test
            yojimbo
            sodium
            tlsf
            netcode
            reliable
    )
    if(APPLE)
        target_link_directories(test PRIVATE /opt/homebrew/lib)
    endif()
endif()

# Custom clean target (CMake has built-in clean, but this matches the original)
add_custom_target(clean-all
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/obj
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/ipch
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/bin
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/.vs
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/Debug
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/Release
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/release
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/cov-int
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_SOURCE_DIR}/docker/yojimbo
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_SOURCE_DIR}/valgrind/yojimbo
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_SOURCE_DIR}/docs
        COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_SOURCE_DIR}/xml
        COMMENT "Cleaning all build files and output"
)