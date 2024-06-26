#[[
This file generates a build system that, when invoked, creates a dynamic library
    containing fTetWild.

This file was created because the cmake system provided with fTetWild only builds
    a static library that contains object files from fTetWild and all the libraries
    that fTetWild depends upon, such as igl, geogram, etc. Using this static
    library is undesirable because it happens to include object files from
    libraries (e.g. igl and geogram) that we would probably like our application
    to link against. But it links against pretty old versions of these libraries.
    We will probably want to link against newer versions of these libraries.

    Using the static library as-provided by fTetWild:

                     app.out
                      /   \
            fTetWild_v0.a  igl_v3.a/.so, geogram_v3.a/.so, etc.
                  |
        igl_v0.a, geogram_v0.a, etc.

    This almost certainly leads to link-time errors due to symbol redefinition. 
   
    Using the dynamic library provided by this file:

                     app.out
                      /   \
             fTetWild.so  igl_v3.so, geogram_v3.so, etc.
                  |
    igl_v0.a/.so, geogram_v0.a/.so, etc.

    This avoids symbol redefinition errors at link time.

    However, it turns out I was missing a critical detail: The API exposed
        by fTetWild.so includes objects from its dependencies. For example,
        the tetrahedralize() function requires objects from the Eigen
        library to be passed to it. This is quite annoying. It means that
        linking against newer versions of the libraries (e.g., igl_v3.so,
        geogram_v3.so, etc.) may yield errors due to ABI incompatability -
        if the content of an Eigen Vec3 changed between eigen_v3.so and
        eigen_c0.so, then hard to debug runtime errors will result.
    There is no good general solution to this problem. In this case, I've chosen
        to modify the interface exposed by the subset of fTetWild that we want
        to use so that it only depends on stdlib++. If the API provided by the
        library was larger, I'm not sure what the best appraoch would be....

The content of this file was mostly sourced from the CMakeLists.txt file that
    accompanies the fTetWild repository.

The fTetWild code relies upon macros to determine when special functionality, such
    such as thread building blocks, should be used. When this file is used
    to generate the build system that is then used to build the dynamic library,
    the dynamic library is built with the following macros defined at compile
    time:
        FLOAT_TETWILD_USE_TBB

The dependency tree is:

    (DL = Dynamic Library)
    (HO = Header Only Library)
    (Anything without listed dependencies only depends on system libraries)

    DL: fTetWild 
        DL: Geogram                        
        DL: GMP 
        DL: Wenzel Jakob's tbb fork 
        HO: IGL
            HO: Eigen
        HO: Jeremie Dumas' Json fork 
        HO: spdlog
        DL: System
        DL: stdc++ 

    The full dependency tree, with version numbers, is pictured in notes/dependency_tree.xopp.
]]

cmake_minimum_required(VERSION 3.26)
project(FloatTetwild LANGUAGES CXX C)

set(CMAKE_WARN_DEPRECATED OFF)

# Using a more recent standard causes errors at build time. At the very least,
#     the format library (which gets built but isn't actually included in the
#     dynamic library) appears to use features that were available in C++11 
#     but were then deprecated in C++17, etc.
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

# Must come before dependencies are downloaded and compiled. 
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
                      tbb::tbb
                     )

target_compile_definitions(${PROJECT_NAME} PUBLIC FLOAT_TETWILD_USE_TBB)

add_subdirectory(src)

# ******************************************************************************
