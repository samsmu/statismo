cmake_minimum_required(VERSION 2.8.10)
project(statismo)

if( POLICY CMP0042 )
  cmake_policy( SET CMP0042 NEW )
endif()

enable_testing()

set( STATISMO_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR} )

set(STATISMO_MAJOR_VERSION 0)
set(STATISMO_MINOR_VERSION 11)
set(STATISMO_PATCH_VERSION 0)
set(STATISMO_VERSION ${STATISMO_MAJOR_VERSION}.${STATISMO_MINOR_VERSION}.${STATISMO_PATCH_VERSION} )

set( statismo_LIB_VERSION ${STATISMO_VERSION} )
set( statismo_LIB_SOVERSION ${STATISMO_MAJOR_VERSION}.${STATISMO_MINOR_VERSION} )

# Setup build locations.
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
endif()

# set CMAKE_MODULE_PATH for cmake macro/function and modules
set( CMAKE_MODULE_PATH
  ${CMAKE_MODULE_PATH}
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake
)

if( APPLE )
  include( CMakeOSXVariables )
endif()

if( CMAKE_HOST_WIN32 )
  string( LENGTH "${CMAKE_CURRENT_SOURCE_DIR}" n )
  if( n GREATER 50 )
    message( FATAL_ERROR
      "Statismo source code directory path length is too long (${n} > 50)."
      "Please move the Statismo source code directory to a directory with a shorter path."
    )
  endif()

  string( LENGTH "${CMAKE_CURRENT_BINARY_DIR}" n )
  if( n GREATER 50 )
    message( FATAL_ERROR
      "Statismo build directory path length is too long (${n} > 50)."
      "Please move the Statismo build directory to a directory with a shorter path."
    )
  endif()
endif()

# installation
set(INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set(INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDE_DIR include CACHE PATH
  "Installation directory for header files")
set(INSTALL_DOC_DIR share/doc/statismo CACHE PATH
  "Installation directory for documentation")

if(WIN32 AND NOT CYGWIN)
  set(DEF_INSTALL_CMAKE_DIR CMake)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/cmake/statismo)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH
  "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE)
  set(var INSTALL_${p}_DIR)
  if(NOT IS_ABSOLUTE "${${var}}")
    set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
  endif()
endforeach()

include( CTest )

include( ModuleHeaderTest )

find_package( Eigen3 REQUIRED )
include_directories( ${EIGEN3_INCLUDE_DIR} )

set( REQUIRED_BOOST_COMPONENTS thread system date_time filesystem )

option( BUILD_CLI_TOOLS "Build command-line tools" ON )
#make the option to build the cli documentation available if and only if the cli tools will be built
include(CMakeDependentOption)
cmake_dependent_option( BUILD_CLI_TOOLS_DOC "Build documentation for the command-line tools" ON
  "BUILD_CLI_TOOLS" OFF
)
#it's only possible to enable the long running CLI tests if testing is enabled and the CLI tools will be built.
cmake_dependent_option( BUILD_LONG_RUNNING_CLI_TESTS "Run the cli examples(the execution of these tests can take some time)" OFF
  "BUILD_CLI_TOOLS;BUILD_TESTING" OFF
)
if( ${BUILD_CLI_TOOLS} MATCHES "ON" )
  if( ${BUILD_CLI_TOOLS_DOC} MATCHES "ON" )
    #search for pandoc to generate man pages for the cli tools
    find_program( PANDOC NAMES pandoc )
    mark_as_advanced( PANDOC )
  endif()

  #the cli tools require the program_options component from boost
  set( REQUIRED_BOOST_COMPONENTS ${REQUIRED_BOOST_COMPONENTS} program_options )
endif()

find_package( Boost 1.50 COMPONENTS ${REQUIRED_BOOST_COMPONENTS} REQUIRED )
include_directories( ${Boost_INCLUDE_DIRS} )

link_directories( ${Boost_LIBRARY_DIRS} )
add_definitions( -DBOOST_THREAD_VERSION=3 -DBOOST_FILESYSTEM_VERSION=3 )

if( WIN32 )
  # disable autolinking in boost
  add_definitions(-DBOOST_ALL_NO_LIB)
  add_definitions(-DBOOST_ALL_DYN_LINK)
endif()

option( ITK_SUPPORT "Build ITK Support" ON )

if( ${ITK_SUPPORT} MATCHES "ON" )
  find_package( ITK 4.13 REQUIRED )
  include( ${ITK_USE_FILE} )

  #set(HDF5_LIBRARIES ibitkhdf5_debugd itkhdf5_debugd)
  #set(HDF5_LIBRARIES itkhdf5 itkhdf5_cpp)

  add_definitions(-DSTATISMO_ITK_SUPPORT)

else()
  find_package( HDF5  COMPONENTS C CXX)
  include_directories( ${HDF5_INCLUDE_DIRS} )

  foreach( _library ${HDF5_LIBRARIES} )
    get_filename_component( _tempDir${_library} ${_library} PATH )
    list( APPEND HDF5_LIBRARY_DIR ${_tempDir${_library}} )
  endforeach()
endif()

option( VTK_SUPPORT "Build VTK Support" ON )

if( ${VTK_SUPPORT} MATCHES "ON" )
  find_package( VTK REQUIRED )
  include( ${VTK_USE_FILE} )
endif()

option( BUILD_SHARED_LIBS "Build shared libraries" ON )

option( BUILD_EXAMPLES "Build examples" ON )

option( BUILD_WRAPPING "Build Python Wrappers (experimental)" OFF )
mark_as_advanced(BUILD_WRAPPING)

if(BUILD_SHARED_LIBS)
  set(statismo_LIB_TYPE SHARED)
else()
  set(statismo_LIB_TYPE STATIC)
endif()

include_directories(${CMAKE_BINARY_DIR})

add_subdirectory( modules )
add_subdirectory( doc )
# -------------------------------------------------------------

configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/scripts/run_astyle.sh.in
  ${CMAKE_CURRENT_BINARY_DIR}/run_astyle.sh
  @ONLY
)

# -------------------------------------------------------------
include(GenerateExportHeader)

# Add all targets to the build-tree export set
set( _targets statismo_core )

generate_export_header(statismo_core EXPORT_FILE_NAME StatismoCoreExport.h)

install(FILES
  "${statismo_BINARY_DIR}/StatismoCoreExport.h"
  DESTINATION ${INSTALL_INCLUDE_DIR}/statismo/core
)

if( VTK_SUPPORT )
  set( _targets ${_targets} statismo_VTK )
  generate_export_header(statismo_VTK EXPORT_FILE_NAME StatismoVTKExport.h)

  install(FILES
    "${statismo_BINARY_DIR}/StatismoVTKExport.h"
    DESTINATION ${INSTALL_INCLUDE_DIR}/statismo/VTK
  )
endif()

export( TARGETS ${_targets}
  FILE "${statismo_BINARY_DIR}/statismoTargets.cmake"
)

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export(PACKAGE statismo)

# -------------------------------------------------------------
# for the build tree
set( CONF_INCLUDE_DIRS
  "${statismo_BINARY_DIR}"
  "${statismo_SOURCE_DIR}/modules/core/include"
)
set( statismo_LIBRARIES statismo_core )

if( ${VTK_SUPPORT} )
  set( CONF_INCLUDE_DIRS "${CONF_INCLUDE_DIRS}" "${statismo_SOURCE_DIR}/modules/VTK/include" )
  set( statismo_LIBRARIES ${statismo_LIBRARIES} statismo_VTK )
endif()

if( ${ITK_SUPPORT} )
  set( CONF_INCLUDE_DIRS "${CONF_INCLUDE_DIRS}"
    "${statismo_SOURCE_DIR}/modules/ITK/include"
    "${statismo_SOURCE_DIR}/modules/ITK/cli/utils" )
endif()

configure_file( ${statismo_SOURCE_DIR}/statismoConfig.cmake.in
  ${statismo_BINARY_DIR}/statismoConfig.cmake
  @ONLY
)

# for the install tree
file(RELATIVE_PATH REL_INCLUDE_DIR "${INSTALL_CMAKE_DIR}"
   "${INSTALL_INCLUDE_DIR}")

set( CONF_INCLUDE_DIRS
  "\${statismo_CMAKE_DIR}/${REL_INCLUDE_DIR}/statismo/core"
)
if( ${VTK_SUPPORT} )
  set( CONF_INCLUDE_DIRS "${CONF_INCLUDE_DIRS}" "\${statismo_CMAKE_DIR}/${REL_INCLUDE_DIR}/statismo/VTK" )
endif()

if( ${ITK_SUPPORT} )
  set( CONF_INCLUDE_DIRS "${CONF_INCLUDE_DIRS}" "\${statismo_CMAKE_DIR}/${REL_INCLUDE_DIR}/statismo/ITK" )
endif()

configure_file( ${statismo_SOURCE_DIR}/statismoConfig.cmake.in
  ${statismo_BINARY_DIR}/cmake/statismoConfig.cmake
  @ONLY
)
configure_file( ${statismo_SOURCE_DIR}/statismoConfigVersion.cmake.in
  ${statismo_BINARY_DIR}/statismoConfigVersion.cmake
  @ONLY
)
install( FILES
    "${statismo_BINARY_DIR}/cmake/statismoConfig.cmake"
    "${statismo_BINARY_DIR}/statismoConfigVersion.cmake"
  DESTINATION ${INSTALL_CMAKE_DIR} COMPONENT dev
)
install( EXPORT statismoTargets
  DESTINATION "${INSTALL_CMAKE_DIR}" COMPONENT dev
)
install( FILES ReadMe.md LICENSE
  DESTINATION ${INSTALL_DOC_DIR}
)
