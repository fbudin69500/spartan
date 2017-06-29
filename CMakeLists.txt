cmake_minimum_required(VERSION 3.5)
project(spartan)

enable_testing()

# Set default build type
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING
    "The type of build. Options are: Debug Release RelWithDebInfo MinSizeRel."
    FORCE)
endif()

# Build shared libraries by default
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

# Set default install prefix
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE STRING
    "Prefix for installation of sub-packages (note: required during build!)"
    FORCE)
endif()

# Options
option(WITH_PERCEPTION "Build with perception libraries and tools." ON)

option(WITH_IIWA_DRIVER "Build drake-iiwa-driver." OFF)
option(WITH_SCHUNK_DRIVER "Build drake-schunk-driver." OFF)

option(BUILD_GYGES "Build the GYGES external project" OFF)

# Setup external projects
include(ExternalProject)
set_property(DIRECTORY PROPERTY EP_PREFIX ${CMAKE_BINARY_DIR})
set(source_dir ${CMAKE_SOURCE_DIR})
set(build_dir ${CMAKE_BINARY_DIR})
include(cmake/set-python-args.cmake)

foreach(proj drake director signal-scope)
  if(NOT EXISTS ${source_dir}/${proj}/.git)
    message(FATAL_ERROR "\nThe ${proj} submodule is not available.\nPlease run: git submodule update --init")
  endif()
endforeach()

set(CMAKE_ARGS_FOR_EXTERNALS
  "-DCMAKE_PREFIX_PATH:PATH=${CMAKE_INSTALL_PREFIX}"
  "-DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}"
  "-DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}"
  "-DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}"
  "-DCMAKE_C_FLAGS:STRING=${CMAKE_C_FLAGS}"
)

# Add external projects
# NOTE: For bazel < 0.5.2, two external projects that use Bazel must not build simultaneously.
# One can depend on another one, which will force the order in which they are built.
set(bazel_args --config=unsupported_crosstool --cxxopt=-std=c++1y --color=no -s)
set(spartan_bazel_args "${bazel_args}")
string(REPLACE ";" " " spartan_bazel_args "${spartan_bazel_args}")
ExternalProject_Add(drake
    SOURCE_DIR ${source_dir}/drake
    BUILD_ALWAYS 1
    BUILD_IN_SOURCE 1
    USES_TERMINAL_BUILD 1
    CONFIGURE_COMMAND ""
    BUILD_COMMAND CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} bazel build //:install ${bazel_args}
    INSTALL_COMMAND CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} bazel run install ${CMAKE_INSTALL_PREFIX} ${bazel_args}
)
# This is based on drake's VTK/Qt dependency (see `tools/vtk.bzl` in drake).
# On MacOS, Drake uses homebrew VTK compiled with Qt5 and brew Python
# On Linux, checks distribution
# Ubuntu 14.04: Drake downloads a precompiled version of VTK compiled with Qt4
# Ubuntu 16.04: Drake downloads a precompiled version of VTK compiled with Qt5
# Otherwise, assume Qt5
set(USED_QTVERSION 5)
if(APPLE)
  # VTK version used by drake
  set(VTK_INSTALL_DIR /usr/local/Cellar/vtk@8.0/8.0.0)
  set(VTK_ARGS
  -DVTK_DIR:PATH=${VTK_INSTALL_DIR}/lib/cmake/vtk-8.0
  -DPYTHON_CONFIG_EXECUTABLE:FILEPATH=/usr/local/bin/python2.7-config
  )
  set(QT_ARGS -DQt5_DIR:PATH=/usr/local/opt/qt/lib/cmake/Qt5)
  set(VTK_PYTHON_DIR ${VTK_INSTALL_DIR}/lib/python2.7/site-packages)
elseif(UNIX)
  find_program(SED sed)
  execute_process(COMMAND ${SED} --version
    RESULT_VARIABLE SED_RESULT
    OUTPUT_VARIABLE SED_OUTPUT
  )
  if(SED_RESULT)
    message(FATAL_ERROR "Could NOT determine Linux distribution information ('sed' is missing?!)")
  endif()
  execute_process(COMMAND /bin/sed -n "/^\\(NAME\\|VERSION_ID\\)=/{s/[^=]*=//;s/\"//g;p}" /etc/os-release
    RESULT_VARIABLE SED_RESULT
    OUTPUT_VARIABLE SED_OUTPUT
  )
  if(SED_RESULT)
    message(FATAL_ERROR "Could NOT determine Linux distribution information")
  endif()
 string(REGEX REPLACE "\n" " " distro "${SED_OUTPUT}")
 string(STRIP ${distro} distro)
 if("${distro}" STREQUAL "Ubuntu 14.04")
   set(USED_QTVERSION 4)
 endif()
endif()

ExternalProject_Add(director
  SOURCE_DIR ${source_dir}/director/distro/superbuild
  BINARY_DIR ${build_dir}/director
  BUILD_ALWAYS 1
  CMAKE_CACHE_ARGS
    -DUSE_EXTERNAL_INSTALL:BOOL=ON
    -DDD_QT_VERSION:STRING=${USED_QTVERSION}
    -DUSE_DRAKE:BOOL=ON
    -DUSE_LCM:BOOL=ON
    -DUSE_LIBBOT:BOOL=ON
    -DUSE_SYSTEM_EIGEN:BOOL=ON
    -DUSE_SYSTEM_LCM:BOOL=ON
    -DUSE_SYSTEM_LIBBOT:BOOL=ON
    -DUSE_SYSTEM_VTK:BOOL=ON
    ${VTK_ARGS}
    -DUSE_PCL:BOOL=${WITH_PERCEPTION}
    -DUSE_APRILTAGS:BOOL=${WITH_PERCEPTION}
    -DUSE_KINECT:BOOL=${WITH_PERCEPTION}
    -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_RPATH:STRING=@loader_path/../lib
    ${CMAKE_ARGS_FOR_EXTERNALS}
  INSTALL_COMMAND ""
  DEPENDS drake
)

ExternalProject_Add(signal-scope
  SOURCE_DIR ${source_dir}/signal-scope
  BINARY_DIR ${build_dir}/signal-scope
  BUILD_ALWAYS 1
  CMAKE_CACHE_ARGS
    ${python_args}
    -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DUSED_QTVERSION:STRING=${USED_QTVERSION}
    ${QT_ARGS}
    ${CMAKE_ARGS_FOR_EXTERNALS}
  DEPENDS director
)

ExternalProject_Add(ContactParticleFilter
  SOURCE_DIR ${source_dir}/src/ContactParticleFilter
  BINARY_DIR ${build_dir}/ContactParticleFilter
  BUILD_ALWAYS 1
  CMAKE_CACHE_ARGS
    -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    ${CMAKE_ARGS_FOR_EXTERNALS}
  DEPENDS drake
)

ExternalProject_Add(spartan_lcmtypes
  SOURCE_DIR ${source_dir}/spartan_lcmtypes
  BINARY_DIR ${build_dir}/spartan_lcmtypes
  BUILD_ALWAYS 1
  CMAKE_CACHE_ARGS
    -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    ${CMAKE_ARGS_FOR_EXTERNALS}
  DEPENDS drake
)

ExternalProject_Add(RemoteTreeViewer
  SOURCE_DIR ${source_dir}/src/RemoteTreeViewer
  BINARY_DIR ${build_dir}/RemoteTreeViewer
  BUILD_ALWAYS 1
  CMAKE_CACHE_ARGS
    -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    ${CMAKE_ARGS_FOR_EXTERNALS}
  DEPENDS drake
)

if (BUILD_GYGES)
  ExternalProject_Add(GYGES
    SOURCE_DIR ${source_dir}/src/Gyges
    BINARY_DIR ${build_dir}/Gyges
    BUILD_ALWAYS 1
    CMAKE_CACHE_ARGS
      -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
      -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
      ${CMAKE_ARGS_FOR_EXTERNALS}
    DEPENDS drake spartan_lcmtypes
  )
endif()

if (OFF)
  ExternalProject_Add(ObjectDetection
    SOURCE_DIR ${source_dir}/src/ObjectDetection
    BINARY_DIR ${build_dir}/ObjectDetection
    BUILD_ALWAYS 1
    CMAKE_CACHE_ARGS
      -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
      -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
      ${CMAKE_ARGS_FOR_EXTERNALS}
    DEPENDS director RemoteTreeViewer
  )

endif()

if (WITH_IIWA_DRIVER)
  set(IIWA_DRIVER_GIT_REPOSITORY
	"git@github.com:RobotLocomotion/drake-iiwa-driver.git"
	CACHE STRING
	"Location of drake-iiwa-driver repo including kuka-fri sources")
  set(IIWA_DRIVER_GIT_TAG
	"3bc3424"
	CACHE STRING
	"Build tag of drake-iiwa-driver repo")

  ExternalProject_Add(drake-iiwa-driver
    SOURCE_DIR ${build_dir}/externals/drake-iiwa-driver
    BINARY_DIR ${build_dir}/drake-iiwa-driver
    GIT_REPOSITORY "${IIWA_DRIVER_GIT_REPOSITORY}"
    GIT_TAG "${IIWA_DRIVER_GIT_TAG}"
    BUILD_ALWAYS 1
    CMAKE_CACHE_ARGS
      -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
      -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
      ${CMAKE_ARGS_FOR_EXTERNALS}
    DEPENDS drake
  )

endif()

if (WITH_SCHUNK_DRIVER)

  ExternalProject_Add(drake-schunk-driver
    SOURCE_DIR ${build_dir}/externals/drake-schunk-driver
    GIT_REPOSITORY git@github.com:RobotLocomotion/drake-schunk-driver.git
    GIT_TAG a28d35c
    BUILD_ALWAYS 1
    BUILD_IN_SOURCE 1
    USES_TERMINAL_BUILD 1
    CONFIGURE_COMMAND ""
    BUILD_COMMAND CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER} bazel build //...
    INSTALL_COMMAND ""
    DEPENDS drake
  )

endif()

# Configure the environment file
configure_file(config/setup_environment.sh.in ${CMAKE_BINARY_DIR}/setup_environment.sh @ONLY)

# Add tests
add_subdirectory(tests)
