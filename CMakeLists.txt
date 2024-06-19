#[[
This file generates a build system that, when invoked, creates a dynamic library
    containing fTetWild.
This file was created because the cmake system provided with fTetWild only builds
    a static library that contains object files from fTetWild and all the libraries
    that fTetWild depends upon, such as igl, geogram, etc. Using this static
    library is undesirable because it happens to include object files from
    libraries (e.g. igl and geogram) that we would probably like out application
    to link against. This almost certainly would lead to symbol redefinition
    errors.

    Using the static library as-provided by fTetWild:

                     app.out
                      /   \
             fTetWild.a  igl.a, geogram.a, etc.
                  |
        igl.a, geogram.a, etc.

    This almost certainly leads to link-time errors due to symbol redefinition. 
        I think that there is a convoluted way to get around this problem by
        omitting the object files from igl.a, geogram.a, etc. that are already
        embedded in fTetWild.a. This seems very messy.
   
    Using the dynamic library provided by the generated build system:

                     app.out
                      /   \
             fTetWild.so  igl.so, geogram.so, etc.
                  |
        igl.so, geogram.so, etc.

    This avoids symbol redefinition errors at link time. In fact, it also makes
        it possible for the application to link against versions of the dynamic
        libraries that are newer than those that fTetWild.so links against.

The content of this file was mostly sourced from the CMakeLists.txt file that
    accompanies the fTetWild repository.
The fTetWild code relies upon macros to determine when special functionality, such
    such as Intel thread building blocks, should be used. When this file is used
    to generate the build system that is then used to build the dynamic library,
    the dynamic library is built with the following macros defined at compile
    time:
        FLOAT_TETWILD_USE_TBB

The dynamic library produced is linked against the following dynamic libraries 
    and header-only libraries: 

    ****
    TODO
    ****

]]

cmake_minimum_required(VERSION 3.26)
project(FloatTetwild LANGUAGES CXX C)

# Using a more recent standard causes build-time errors.
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

set(FLOAT_TETWILD_TOPLEVEL_PROJECT ON)
set(FLOAT_TETWILD_EXTERNAL ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

include(PrependCurrentPath)
include(FloatTetwildUtils)
include(Warnings)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# ******************************************************************************
#                           Options For Dependencies
# ******************************************************************************

option(LIBIGL_USE_STATIC_LIBRARY     "Use libigl as static library" OFF)
option(LIBIGL_WITH_COMISO            "Use CoMiso"                   OFF)
option(LIBIGL_WITH_EMBREE            "Use Embree"                   OFF)
option(LIBIGL_WITH_OPENGL            "Use OpenGL"                   OFF)
option(LIBIGL_WITH_OPENGL_GLFW       "Use GLFW"                     OFF)
option(LIBIGL_WITH_OPENGL_GLFW_IMGUI "Use ImGui"                    OFF)
option(LIBIGL_WITH_PNG               "Use PNG"                      OFF)
option(LIBIGL_WITH_TETGEN            "Use Tetgen"                   OFF)
option(LIBIGL_WITH_TRIANGLE          "Use Triangle"                 OFF)
option(LIBIGL_WITH_PREDICATES        "Use exact predicates"         ON)
option(LIBIGL_WITH_XML               "Use XML"                      OFF)

# ******************************************************************************

# Must come before dependencies are dealt with.
# This ensures that TBB is downloaded and a target is created.
option(FLOAT_TETWILD_ENABLE_TBB "Enable TBB" ON)

include(FloatTetwildDependencies)

# GMP is special for some reason. The expectation is that it exists on the system
#     already and it just needs to be found.
find_package(GMPfTetWild)
IF(NOT ${GMP_FOUND})
    MESSAGE(FATAL_ERROR "Cannot find GMP")
ENDIF()

# ******************************************************************************
#                      Constructing the Dynamic Library 
# ******************************************************************************

# The dynamic library.
add_library(${PROJECT_NAME} SHARED src/Logger.cpp src/external/WindingNumber.h)

target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_BINARY_DIR}/include)
target_link_libraries(${PROJECT_NAME} PRIVATE warnings::all)
target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC ${GMP_INCLUDE_DIRS})

target_link_libraries(${PROJECT_NAME}
	              PUBLIC
		      igl::core
		      igl::predicates
		      geogram::geogram
		      spdlog::spdlog
		      Threads::Threads
		      json
		      ${GMP_LIBRARIES}
                     )

target_link_libraries(${PROJECT_NAME} PUBLIC tbb::tbb)
target_compile_definitions(${PROJECT_NAME} PUBLIC FLOAT_TETWILD_USE_TBB)

# Add more source files to the dynamic library.
add_subdirectory(src)

# ******************************************************************************
