cmake_minimum_required(VERSION 3.24)

project(volmaris-forge
    VERSION 0.0.1
    DESCRIPTION "Open-source 2D HVAC schematic editor"
    LANGUAGES CXX
)

# --------------------------------------------------------------------------
# Global C++ settings
# --------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# --------------------------------------------------------------------------
# Qt setup (auto-MOC / -UIC / -RCC for any target that links Qt)
# --------------------------------------------------------------------------
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets OpenGL OpenGLWidgets)

qt_standard_project_setup()

# --------------------------------------------------------------------------
# Third-party deps
# --------------------------------------------------------------------------
# Boost.Geometry and SQLite are installed by the vcpkg manifest but are not
# consumed by any target in Phase 0. They will be wired up by the modules
# that need them (core/spatial -> Boost.Geometry, io -> SQLite) in Phase 1.

# --------------------------------------------------------------------------
# Warnings
# --------------------------------------------------------------------------
if(MSVC)
    add_compile_options(/W4 /permissive- /utf-8)
else()
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# --------------------------------------------------------------------------
# Subdirectories
# --------------------------------------------------------------------------
add_subdirectory(src)

# --------------------------------------------------------------------------
# Tests
# --------------------------------------------------------------------------
include(CTest)
if(BUILD_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif()
