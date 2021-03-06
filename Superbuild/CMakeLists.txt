cmake_minimum_required(VERSION 2.8.8)

project(VeloViewSuperBuild)
set (SuperBuild_CMAKE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/CMake")
set (SuperBuild_PROJECTS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/Projects")
set (SuperBuild_BINARY_DIR ${VeloViewSuperBuild_BINARY_DIR})

include(CMakeDependentOption)

#------------------------------------------------------------------------------
# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'RelWithDebInfo' as none was specified.")
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

#-----------------------------------------------------------------------------
# Collect information about the build platform.
include(CheckTypeSize)
check_type_size(void* VOID_PTR_SIZE BUILTIN_TYPES_ONLY)
if (VOID_PTR_SIZE EQUAL 8)
  set(VV_BUILD_ARCHITECTURE "64")
else ()
  set(VV_BUILD_ARCHITECTURE "32")
endif()

set (platform)
if (APPLE)
  set (platform "apple")
  option(APPLY_MOUNTAIN_LION_FIX "Apply Mountain Lion bundle fix" OFF)
  mark_as_advanced(APPLY_MOUNTAIN_LION_FIX)
  # Fix stdlic being wrongly used instead of libc++
  #   (as it conflicts with boost which always uses libc++)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libc++")
elseif (UNIX)
  set (platform "unix")
elseif (WIN32)
  set (platform "win32")
endif()

# Set suffix to be used for generating archives. This ensures that the package
# files have decent names that we can directly upload to the website.
set (package_suffix "${CMAKE_SYSTEM_NAME}-${VV_BUILD_ARCHITECTURE}bit")

#-----------------------------------------------------------------------------
# Setup CMAKE_MODULE_PATH so that platform specific configurations are processed
# before the generic ones.
set (CMAKE_MODULE_PATH
  "${SuperBuild_PROJECTS_DIR}/${platform}"
  "${SuperBuild_PROJECTS_DIR}/common"
  "${SuperBuild_PROJECTS_DIR}"
  "${SuperBuild_CMAKE_DIR}"
  "${CMAKE_CURRENT_SOURCE_DIR}"
  "${CMAKE_CURRENT_SOURCE_DIR}/../CMake"
  ${CMAKE_MODULE_PATH})

#-----------------------------------------------------------------------------
# Setup some standard variables that control various locations and flags.
set (install_location "${CMAKE_CURRENT_BINARY_DIR}/install")
set (download_location "${CMAKE_CURRENT_BINARY_DIR}/downloads"
     CACHE PATH "Location where source tar-balls are (to be) downloaded.")
mark_as_advanced(download_location)

if (WIN32)
  set (ldflags)
  set (cppflags)
  set (cxxflags)
  set (ld_library_path)
else ()
  # FIXME: currently, if any of these are empty, we have build issues on Unix.
  set (ldflags "$ENV{LDFLAGS} -L${install_location}/lib")
  set (cppflags "$ENV{CPPFLAGS} -fPIC")
  set (cxxflags "$ENV{CXXFLAGS} -fPIC")
  set (cflags "$ENV{CFLAGS} -fPIC")
  set (ld_library_path "${install_location}/lib:$ENV{LD_LIBRARY_PATH}")
endif()

set (prefix_path "${install_location}")

# merge in default/user-specified CMake flags.
set (ldflags "${CMAKE_SHARED_LINKER_FLAGS} ${ldflags}")
set (cppflags "${CMAKE_CXX_FLAGS} ${cppflags}")
set (cxxflags "${CMAKE_CXX_FLAGS} ${cxxflags}")
set (cflags "${CMAKE_C_FLAGS} ${cflags}")

string (STRIP "${ldflags}" ldflags)
string (STRIP "${cppflags}" cppflags)
string (STRIP "${cxxflags}" cxxflags)
string (STRIP "${cflags}"  cflags)
#-----------------------------------------------------------------------------
## Top-level options. These must be limited to a commonly used subset with
## default values being used for building VeloView binaries.
#option(PV_COMMAND_LINE_TOOLS_ONLY
#  "Turn this ON to build command lines tools alone." OFF)

if (WIN32)
  # Since we only support superbuild compilation with ninja, let's provide it
  file(COPY ${SuperBuild_PROJECTS_DIR}/win32/ninja.exe
       DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
  if ( (NOT "${CMAKE_GENERATOR}" MATCHES "^NMake.*$") OR
       (NOT "${CMAKE_GENERATOR}" MATCHES "^Visual Studio.*$"))
    # not use VS environment. We need to be pointed to nmake and devenv paths
    # since they are needed when building tools (qt, python, etc.)
    find_program(DEVENV_PATH devenv)
    find_program(NMAKE_PATH nmake)
    mark_as_advanced(DEVENV_PATH NMAKE_PATH)
  endif()
endif()

#-----------------------------------------------------------------------------
include (ParaViewModules)
include (versions)

#-----------------------------------------------------------------------------
set (CM_PROJECTS_ALL)

# Include all optional projects. These are the ones user can pick and choose.
# Some of these allow using system libraries too.
include(qt)
include(python)
include(pythonqt)
include(paraview)
include(pcap)
include(boost)
include(eigen)
include(liblas)
include(veloview)

process_dependencies()

if (veloview_ENABLED)
  include(veloview.bundle OPTIONAL)
endif()
