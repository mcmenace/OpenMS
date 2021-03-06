# --------------------------------------------------------------------------
#                   OpenMS -- Open-Source Mass Spectrometry
# --------------------------------------------------------------------------
# Copyright The OpenMS Team -- Eberhard Karls University Tuebingen,
# ETH Zurich, and Freie Universitaet Berlin 2002-2012.
#
# This software is released under a three-clause BSD license:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of any author or any participating institution
#    may be used to endorse or promote products derived from this software
#    without specific prior written permission.
# For a full list of authors, refer to the file AUTHORS.
# --------------------------------------------------------------------------
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL ANY OF THE AUTHORS OR THE CONTRIBUTING
# INSTITUTIONS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS;
# OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
# WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR
# OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF
# ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
# --------------------------------------------------------------------------
# $Maintainer: Stephan Aiche $
# $Authors: Stephan Aiche $
# --------------------------------------------------------------------------

cmake_minimum_required(VERSION 2.8.3 FATAL_ERROR)
project("OpenMS_host")

#------------------------------------------------------------------------------
# General CMake definitions & helper
#------------------------------------------------------------------------------
SET(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS true)

#------------------------------------------------------------------------------
## CMake sanity check: sometimes CMAKE_SIZEOF_VOID_P just vanishes when
## updating CMake.
if (NOT CMAKE_SIZEOF_VOID_P)
	message(FATAL_ERROR "'CMAKE_SIZEOF_VOID_P' is undefined. Thus you should"
                      " delete CMakeFiles (the directory) and the"
                      " CMakeCache.txt and rerun CMake again! This is some"
                      " weird CMake bug that seems to appear when updating the"
                      " CMake version.")
endif()

#------------------------------------------------------------------------------
# Include the standard Dart testing module
# for Nighlty Build log
enable_testing()
include(Dart)

#------------------------------------------------------------------------------
# entries meant to be configured using CMake cache
# - do NOT hardcode them here!
# - edit them within CMakeCache.txt using ccmake or cmake -D VAR=VALUE
#------------------------------------------------------------------------------
# CMAKE_FIND_ROOT_PATH
# CMAKE_BUILD_TYPE
# STL_DEBUG
# DB_TEST
# QT_DB_PLUGIN
# MT_CUDA_BUILD_TYPE
#------------------------------------------------------------------------------
option(MT_ENABLE_CUDA "Enable experimental CUDA support" OFF)
option(MT_ENABLE_TBB "Enable Intel TBB support" OFF)
option(MT_ENABLE_OPENMP "Enable OpenMP support" ON)
option(BOOST_USE_STATIC "Use Boost static libraries." ON)
option(HAS_XSERVER "Indicates if an XServer is available. If set to Off it will disable certain tests and the doc target." ON)
option(ENABLE_TUTORIALS "Indicates wether tutorials should be build. Note that this also depends on the availability of (pdf)latex." ON)

#------------------------------------------------------------------------------
# OpenMS Version Information
#------------------------------------------------------------------------------

set(OPENMS_PACKAGE_VERSION_MAJOR "1")
set(OPENMS_PACKAGE_VERSION_MINOR "12")
set(OPENMS_PACKAGE_VERSION_PATCH "0")

# add some informations used when building packages
include(cmake/package_definitions.cmake)

#------------------------------------------------------------------------------
# Unsupported systems
#------------------------------------------------------------------------------
if (MINGW OR MSYS)
  message(FATAL_ERROR "MSYS and/or MinGW are not supported! Please use a Visual Studio environment! See Windows build instructions for further information!")
endif()

#------------------------------------------------------------------------------
# MacOS X related library extensions
#------------------------------------------------------------------------------
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	# due to some awkward updates of cmake in the 2.8.x series we need to add the
  # following
  find_library(CARBON NAMES Carbon)
  find_package(OpenGL)
  set(APPLE_EXTRA_LIBS ${CARBON} ${OPENGL_LIBRARY})
else()
  set(APPLE_EXTRA_LIBS "")
endif()

#------------------------------------------------------------------------------
# All the multithreading stuff (OpenMP, CUDA, TBB)
#------------------------------------------------------------------------------
include(cmake/multithreading.cmake)

#------------------------------------------------------------------------------
# Check architecture 32bit vs. 64bit
#------------------------------------------------------------------------------

if (CMAKE_SIZEOF_VOID_P MATCHES "8")
	set(OPENMS_64BIT_ARCHITECTURE 1 CACHE INTERNAL "Architecture-bits")
	message(STATUS "Architecture: 64 bit")
else()
	set(OPENMS_64BIT_ARCHITECTURE 0 CACHE INTERNAL "Architecture-bits")
	message(STATUS "Architecture: 32 bit")
endif()

#------------------------------------------------------------------------------
# Handle build type
#------------------------------------------------------------------------------
# Set default build type (if not set by user on command line)
if (NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release)
endif()

# Force build type into the cache (needs to be set beforehand)
set(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)

#------------------------------------------------------------------------------
# Project specific compiler flags
#------------------------------------------------------------------------------
# Fill this with compile flags that external projects should use as well
# for OpenMS internal flags (not promoted to external compiler flags) append to CMAKE_CXX_FLAGS
# see OpenMS/cmake/OpenMSConfig.cmake.in to see how its configured and used (i.e. as OPENMS_ADDCXX_FLAGS)
set(CF_OPENMS_ADDCXX_FLAGS)
include(cmake/compiler_flags.cmake)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CF_OPENMS_ADDCXX_FLAGS}")

#------------------------------------------------------------------------------
# Enable STL debug mode (GCC only)
#------------------------------------------------------------------------------
option(STL_DEBUG OFF "[GCC only] Enable STL-DEBUG mode (very slow).")
if(STL_DEBUG)
  include(cmake/stl_debug.cmake)
endif()

#------------------------------------------------------------------------------
# we build shared libraries
set(BUILD_SHARED_LIBS true)

#------------------------------------------------------------------------------
# CMake Utitlities
#------------------------------------------------------------------------------
# include some OpenMS specific macros
include (${PROJECT_SOURCE_DIR}/cmake/OpenMSBuildSystem_macros.cmake)
# .. and some to ease the installation proces
include (${PROJECT_SOURCE_DIR}/cmake/install_macros.cmake)

#------------------------------------------------------------------------------
# Directories
#------------------------------------------------------------------------------

#------------------------------------------------------------------------------
# Host directory for referencing from subprojects
set(OPENMS_HOST_DIRECTORY "${PROJECT_SOURCE_DIR}")
set(OPENMS_HOST_BINARY_DIRECTORY "${PROJECT_BINARY_DIR}")

#------------------------------------------------------------------------------
# Output directory
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${OPENMS_HOST_BINARY_DIRECTORY}/lib")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${OPENMS_HOST_BINARY_DIRECTORY}/bin")
set(OPENMS_BINARY_DIR "${OPENMS_HOST_BINARY_DIRECTORY}/bin")
set(OPENMS_WIN32_DLL_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

#------------------------------------------------------------------------------
# define installation subdirectories to allow for custom installations
# note that all those directories are below CMAKE_INSTALL_PREFIX
set(INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDE_DIR include CACHE PATH "Installation directory for header files")
set(INSTALL_DOC_DIR share/doc CACHE PATH "Installation directory for documentation")
set(INSTALL_SHARE_DIR share/OpenMS CACHE PATH "Installation directory for shared data")

if(WIN32)
  set(DEF_INSTALL_CMAKE_DIR CMake)
  set(DEF_INSTALL_LIB_DIR bin)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/CMake/FooBar)
  set(DEF_INSTALL_LIB_DIR lib)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH "Installation directory for CMake files")
set(INSTALL_LIB_DIR ${DEF_INSTALL_LIB_DIR} CACHE PATH "Installation directory for libraries")

# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE DOC SHARE)
  set(var INSTALL_${p}_DIR)
  if(NOT IS_ABSOLUTE "${${var}}")
    set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
  endif()
endforeach()

#------------------------------------------------------------------------------
# (create an empty list if CMAKE_MODULE_PATH does not exist)
if(NOT CMAKE_MODULE_PATH)
  set(CMAKE_MODULE_PATH)
endif()
list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/modules")

#------------------------------------------------------------------------------
# lib naming when building with msvc
if (MSVC)
	## use OpenMSd.dll in debug mode
	SET(CMAKE_DEBUG_POSTFIX d)
endif()

########################################################
### setup path for external libs (contrib or system) ###
########################################################
include(${OPENMS_HOST_DIRECTORY}/cmake/OpenMSBuildSystem_setup_lib_find_paths.cmake)

#------------------------------------------------------------------------------
# Initialize package building environment
set(VALID_PACKAGE_TYPES "none" # the classical build type make/make install
                        "rpm"  # rpm package
                        "deb"  # deb package
                        "dmg") # a drag&drop dmg package for macosx

set(PACKAGE_TYPE "none" CACHE STRING "Package type (internal): ${VALID_PACKAGE_TYPES}")
is_valid_package(${PACKAGE_TYPE})

#------------------------------------------------------------------------------
# The actual openms code
#------------------------------------------------------------------------------
add_subdirectory(src)

#------------------------------------------------------------------------------
# Documentation
#------------------------------------------------------------------------------
add_subdirectory(doc)

#------------------------------------------------------------------------------
# GKN Package building
#------------------------------------------------------------------------------
# We will use this path for the knime packages and the osx installer
set(SEARCH_ENGINES_DIRECTORY "" CACHE PATH "Directory containing the search engine executables that should be shipped with OpenMS. Note: We expect the layout from the SVN.")

set(ENABLE_PREPARE_KNIME_PACKAGE OFF CACHE BOOL "If enabled, targets to prepare KNIME packages will be generated. Main target will be 'prepare_knime_package'.")
if (ENABLE_PREPARE_KNIME_PACKAGE)
	include(${OPENMS_HOST_DIRECTORY}/cmake/OpenMSBuildSystem_ctdSupport.cmake)
endif()

######################################################
#### build settings and configs for external code ####
######################################################
if ("${PACKAGE_TYPE}" STREQUAL "none")
	## grep definitions (-D<XXX>) that were set using 'add_definitions()'
	get_property(OPENMS_CMP_DEFS DIRECTORY . PROPERTY COMPILE_DEFINITIONS)
	## convert to CXX flags
	foreach(i ${OPENMS_CMP_DEFS})
		set(CF_OPENMS_ADDCXX_FLAGS "${CF_OPENMS_ADDCXX_FLAGS} -D${i}")
	endforeach(i)
	## for add_definition commands (currently empty, but who knows...)
	set(CF_OPENMS_ADD_DEFINITIONS)
	## the filename for library settings (this file will be included in OpenMSConfig.cmake)
	set(CF_LibOpenMSExport ${PROJECT_BINARY_DIR}/cmake/OpenMSLibraryExport.cmake)

	## create code that allows to call find_package()
	## (must come after the above code that sets the variables!)
	get_directory_property(OPENMS_INCLUDE_DIRS  INCLUDE_DIRECTORIES)
	configure_file(
		"${OPENMS_HOST_DIRECTORY}/cmake/OpenMSConfig.cmake.in"
		"${PROJECT_BINARY_DIR}/cmake/OpenMSConfig.cmake"
		@ONLY
	)

  #------------------------------------------------------------------------------
  # REMOVE with the fix of #676
	## Create OpenMSLibraryExport.cmake:
	## Note: export_build_settings and
	##       export_library_dependencies are deprecated and should not be used
	if ("${INSTALL_PREFIX}" STREQUAL ".")
		set(CVERSION "${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}")
		if (CVERSION VERSION_LESS "2.8.0")
			message(STATUS "Info: Cannot register OpenMS with CMake! For external code, set the path to OpenMS during find_package() manually.")
		else()
			## register OpenMS with CMake so that it can be found easily
			export(PACKAGE OpenMS)
		endif()
		## export our lib
    if(NOT DISABLE_OPENSWATH)
      export(TARGETS OpenMS OpenSwathAlgo FILE ${CF_LibOpenMSExport})
    else()
      export(TARGETS OpenMS FILE ${CF_LibOpenMSExport})
    endif(NOT DISABLE_OPENSWATH)
	else()
		# include(cmake/OpenMSBuildSystem_install.cmake)
	endif()
  #------------------------------------------------------------------------------
else()
	# install routines for MacOSX
	if("${PACKAGE_TYPE}" STREQUAL "dmg")
		include(cmake/package_dragndrop_dmg.cmake)
	endif()

  # .. rpms
	if("${PACKAGE_TYPE}" STREQUAL "rpm")
		include(cmake/package_rpm.cmake)
	endif()

  # .. deps
	if("${PACKAGE_TYPE}" STREQUAL "deb")
		include(cmake/package_deb.cmake)
	endif()
endif()

#------------------------------------------------------------------------------
# Some final messages for the users
#------------------------------------------------------------------------------

message(STATUS "")
message(STATUS "-----------------------------------------------------------------")
message(STATUS "")
message(STATUS "You have successfully configured OpenMS.")
message(STATUS "")
if (MSVC)
  message(STATUS "Execute the 'targets' project to see prominent targets!")
  message(STATUS "For faster loading see the individual solution files located in")
  message(STATUS "the src/ and doc/ folder and their subfolders")
  message(STATUS "  e.g., src/openms/OpenMS.sln")
  message(STATUS "for the solution containing the OpenMS library.")
else()
  message(STATUS "For a full list of make targets execute:")
  message(STATUS "'make targets'")
endif()
message(STATUS "")
message(STATUS "-----------------------------------------------------------------")
message(STATUS "")
