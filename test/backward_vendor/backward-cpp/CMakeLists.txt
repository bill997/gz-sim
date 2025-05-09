#
# CMakeLists.txt
# Copyright 2013 Google Inc. All Rights Reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.14)
project(backward CXX)

# Introduce variables:
# * CMAKE_INSTALL_LIBDIR
# * CMAKE_INSTALL_BINDIR
# * CMAKE_INSTALL_INCLUDEDIR
include(GNUInstallDirs)

include(BackwardConfig.cmake)

###############################################################################
# OPTIONS
###############################################################################

option(BACKWARD_SHARED "Build backward as a shared library" OFF)
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR AND NOT DEFINED BACKWARD_TESTS)
	# If this is a top level CMake project, we most lixely want the tests
	set(BACKWARD_TESTS ON CACHE BOOL "Enable tests")
else()
	set(BACKWARD_TESTS OFF CACHE BOOL "Enable tests")
endif()

###############################################################################
# COMPILER FLAGS
###############################################################################

# check if compiler is nvcc or nvcc_wrapper
set(COMPILER_IS_NVCC false)
get_filename_component(COMPILER_NAME ${CMAKE_CXX_COMPILER} NAME)
if (COMPILER_NAME MATCHES "^nvcc")
  set(COMPILER_IS_NVCC true)
endif()

if (DEFINED ENV{OMPI_CXX} OR DEFINED ENV{MPICH_CXX})
   if ( ($ENV{OMPI_CXX} MATCHES "nvcc") OR ($ENV{MPICH_CXX} MATCHES "nvcc") )
     set(COMPILER_IS_NVCC true)
   endif()
endif()

# set CXX standard
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_CXX_STANDARD 11)
if (${COMPILER_IS_NVCC})
  # GNU CXX extensions are not supported by nvcc
  set(CMAKE_CXX_EXTENSIONS OFF)
endif()

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_COMPILER_IS_GNUCXX)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
	if (NOT ${COMPILER_IS_NVCC})
	  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic-errors")
	endif()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
endif()

###############################################################################
# BACKWARD INTERFACE
###############################################################################

add_library(backward_interface INTERFACE)
set_target_properties(backward_interface PROPERTIES EXPORT_NAME Interface)
target_compile_definitions(backward_interface INTERFACE ${BACKWARD_DEFINITIONS})
target_include_directories(backward_interface INTERFACE ${BACKWARD_INCLUDE_DIRS})
if(BACKWARD_HAS_EXTERNAL_LIBRARIES)
    target_link_libraries(backward_interface INTERFACE ${BACKWARD_LIBRARIES})
endif()
add_library(Backward::Interface ALIAS backward_interface)

###############################################################################
# BACKWARD OBJECT (Includes backward.cpp)
# (Note that this target is not exported, since CMake currently does not allow
# exporting an OBJECT library.)
###############################################################################

add_library(backward_object OBJECT backward.cpp)
set_target_properties(backward_object PROPERTIES EXPORT_NAME Object)
target_link_libraries(backward_object PUBLIC Backward::Interface)
add_library(Backward::Object ALIAS backward_object)

###############################################################################
# BACKWARD LIBRARY (Includes backward.cpp)
# (Note that the linker will not include unused objects from a static library,
# unless the -Wl,--whole-archive option (or similar) is used.)
###############################################################################

set(libtype STATIC)
if(BACKWARD_SHARED)
    set(libtype SHARED)
endif()
add_library(backward ${libtype} backward.cpp)
set_target_properties(backward PROPERTIES EXPORT_NAME Backward)
target_link_libraries(backward PUBLIC Backward::Interface)
add_library(Backward::Backward ALIAS backward)

# check if Backward is being used as a top-level project or included as a subproject
if(CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR)
    install(
        FILES "backward.hpp"
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
    install(
        FILES "BackwardConfig.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
    )
    # export the targets (note that exporting backward_object does not make sense)
    install(TARGETS backward_interface backward EXPORT BackwardTargets)
    # install a CMake file for the exported targets
    install(EXPORT BackwardTargets
            NAMESPACE Backward::
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
endif()
