# Building Yojimbo

This document provides comprehensive instructions for building and installing the Yojimbo network library.

## Prerequisites

### Required Tools
- **CMake** 3.15 or later
- **C++ Compiler** with C++11 support:
    - GCC 4.8+
    - Clang 3.3+
    - MSVC 2015+ (Visual Studio 2015 or later)
- **Git** (for cloning the repository)

### Platform-Specific Requirements

#### Windows
- Visual Studio 2015 or later (Community edition is enough)
- Windows SDK

#### macOS
- Xcode Command Line Tools
- Homebrew (recommended for dependency management)

#### Linux
- Build essentials package:
  ```bash
  # Ubuntu/Debian
  sudo apt-get install build-essential cmake git
  
  # CentOS/RHEL/Fedora
  sudo yum install gcc gcc-c++ cmake git make
  # or (newer versions)
  sudo dnf install gcc gcc-c++ cmake git make
  ```

## Quick Start

### Clone the Repository
```bash
git clone https://github.com/mas-bandwidth/yojimbo.git
cd yojimbo
```

### Basic Build (Static Libraries)
```bash
# Create build directory
mkdir build && cd build

# Configure with CMake
cmake .. -DYOJIMBO_TESTS=ON

# Build
cmake --build .

# Run tests (optional)
./test
```

## Build Options

### CMake Configuration Options

| Option             | Default | Description                         |
|--------------------|---------|-------------------------------------|
| `YOJIMBO_TESTS`    | `OFF`   | Build test programs and examples    |
| `YOJIMBO_INSTALL`  | `ON`    | Enable installation targets         |
| `CMAKE_BUILD_TYPE` | `Debug` | Build configuration (Debug/Release) |

### Examples

#### Static Libraries (Default)
```bash
cmake -B build -DYOJIMBO_TESTS=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

#### Debug Build with Tests
```bash
cmake -B build-debug \
    -DCMAKE_BUILD_TYPE=Debug \
    -DYOJIMBO_TESTS=ON
cmake --build build-debug
```

## Platform-Specific Building

### Windows (Visual Studio)

#### Using Visual Studio IDE
```cmd
# Generate Visual Studio solution
cmake -B build -G "Visual Studio 16 2019" -A x64 -DYOJIMBO_TESTS=ON

# Open the solution
start build\Yojimbo.sln
```

#### Using Command Line
```cmd
# Configure
cmake -B build -DYOJIMBO_TESTS=ON -DCMAKE_BUILD_TYPE=Release

# Build
cmake --build build --config Release

# Or use MSBuild directly
MSBuild.exe build\Yojimbo.sln /p:Configuration=Release
```

### macOS

#### Using Xcode
```bash
# Generate Xcode project
cmake -B build -G Xcode -DYOJIMBO_TESTS=ON

# Open in Xcode
open build/Yojimbo.xcodeproj
```

#### Using Make
```bash
# Configure with Make generator
cmake -B build -G "Unix Makefiles" -DYOJIMBO_TESTS=ON -DCMAKE_BUILD_TYPE=Release

# Build
cmake --build build -j$(sysctl -n hw.ncpu)
```

### Linux

#### Standard Build
```bash
# Configure
cmake -B build -DYOJIMBO_TESTS=ON -DCMAKE_BUILD_TYPE=Release

# Build with all CPU cores
cmake --build build -j$(nproc)
```

#### Using Ninja (faster builds)
```bash
# Install ninja first
sudo apt-get install ninja-build  # Ubuntu/Debian
# or
sudo dnf install ninja-build      # Fedora

# Configure with Ninja
cmake -B build -G Ninja -DYOJIMBO_TESTS=ON -DCMAKE_BUILD_TYPE=Release

# Build
cmake --build build
```

## Installation

### System-wide Installation

#### Linux/macOS
```bash
# Configure with installation prefix
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr/local

# Build and install
cmake --build build
sudo cmake --install build
```

#### Windows
```cmd
# Configure (as Administrator)
cmake -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=C:\yojimbo

# Build and install
cmake --build build --config Release
cmake --install build --config Release
```

### Custom Installation Directory
```bash
# Install to custom location
cmake -B build -DCMAKE_INSTALL_PREFIX=$HOME/yojimbo
cmake --build build
cmake --install build
```

### Installation Contents
After installation, you'll have:
```
${CMAKE_INSTALL_PREFIX}/
├── lib/
│   ├── libyojimbo.a (or .so/.dll)
│   ├── libnetcode.a
│   ├── libreliable.a
│   ├── libtlsf.a
│   ├── libsodium-builtin.a
│   ├── pkgconfig/
│   │   └── yojimbo.pc
│   └── cmake/yojimbo/
│       ├── yojimboConfig.cmake
│       ├── yojimboConfigVersion.cmake
│       └── yojimboTargets.cmake
├── include/yojimbo/
│   ├── *.h (main headers)
│   ├── netcode/
│   ├── reliable/
│   ├── tlsf/
│   ├── sodium/
│   └── serialize/
└── bin/ (if tests enabled)
    ├── client
    ├── server
    ├── loopback
    ├── soak
    └── test
```

## Using Yojimbo in Your Project

### CMake Integration
```cmake
# Find and link yojimbo
find_package(yojimbo REQUIRED)

add_executable(my_game src/main.cpp)
target_link_libraries(my_game yojimbo::yojimbo)
```

### pkg-config Integration
```bash
# Compile with pkg-config
g++ -o my_game main.cpp $(pkg-config --cflags --libs yojimbo)
```

### Manual Integration
```cmake
# If not installed system-wide
find_library(YOJIMBO_LIBRARY yojimbo PATHS /path/to/yojimbo/lib)
find_path(YOJIMBO_INCLUDE yojimbo.h PATHS /path/to/yojimbo/include/yojimbo)

target_include_directories(my_target PRIVATE ${YOJIMBO_INCLUDE})
target_link_libraries(my_target ${YOJIMBO_LIBRARY})
```

## Testing

### Running Built-in Tests
```bash
# After building with -DYOJIMBO_TESTS=ON
cd build

# Run the main test suite
./test

# Run example programs
./client
./server
./loopback
./soak
```

### Cross-compilation Testing
```bash
# Example for ARM64
cmake -B build-arm64 \
    -DCMAKE_TOOLCHAIN_FILE=path/to/arm64-toolchain.cmake \
    -DYOJIMBO_TESTS=ON

cmake --build build-arm64
```

## Troubleshooting

### Common Issues

#### CMake Version Too Old
```bash
# Error: CMake 3.15 or higher is required
# Solution: Update CMake or use a newer version
cmake --version
```

#### Missing Compiler
```bash
# Linux: Install build tools
sudo apt-get install build-essential

# macOS: Install Xcode Command Line Tools
xcode-select --install
```

#### Windows: MSVC Not Found
- Install Visual Studio with C++ development tools
- Or install Build Tools for Visual Studio

#### Link Errors on Linux
```bash
# If you get undefined symbol errors, ensure proper linking order
# The libraries have dependencies: yojimbo -> netcode/reliable/tlsf -> sodium-builtin
```

### Debug Build Issues
```bash
# If debug builds are too slow, try RelWithDebInfo
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo -DYOJIMBO_TESTS=ON
```

### Clean Build
```bash
# Remove build directory and start fresh
rm -rf build
mkdir build && cd build
cmake .. -DYOJIMBO_TESTS=ON
```

## Advanced Building

### Cross-compilation
```bash
# Example for Raspberry Pi
cmake -B build-rpi \
    -DCMAKE_TOOLCHAIN_FILE=cmake/rpi-toolchain.cmake \
    -DYOJIMBO_TESTS=ON

cmake --build build-rpi
```

### Static Analysis
```bash
# Using clang-tidy
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
clang-tidy src/*.cpp -p build/
```

### Packaging
```bash
# Create distribution packages
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
cpack --config build/CPackConfig.cmake
```

## Performance Notes

- **Release builds** are significantly faster than Debug builds
- **Fast math** optimizations are enabled in Release mode
- Consider **LTO (Link Time Optimization)** for maximum performance:
  ```bash
  cmake -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON
  ```

## Getting Help

If you encounter issues:

1. Check the [GitHub Issues](https://github.com/mas-bandwidth/yojimbo/issues)
2. Review the build output for specific error messages
3. Ensure all prerequisites are met
4. Try a clean build
5. Check platform-specific requirements

For more information, visit the [Yojimbo Documentation](https://github.com/mas-bandwidth/yojimbo).