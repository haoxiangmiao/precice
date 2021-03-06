cmake_minimum_required (VERSION 3.10.2)
include(${CMAKE_CURRENT_LIST_DIR}/cmake/XSDKCompilerDetect.cmake)

project(preCICE VERSION 1.6.1 LANGUAGES CXX)

#
# Overview of this configuration
# 
# PREAMBLE
# Setup Options
# Find Mandatory Dependencies
# Find Configurable Dependencies
# Configuration of Target precice
# Configuration of Target binprecice
# Configuration of Target testprecice
# Install Targets for precice
# CPack
#


#
# PREAMBLE
#

# Make our cmake modules visible to CMake
list (APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")
list (APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/modules")

include(CheckCXX11Library)
include(CopyTargetProperty)
include(XSDKMacros)
include(Validation)

# CMake Policies

# CMP0074: find_package() uses <PackageName>_ROOT variables.
if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()
# CMP0075 Include file check macros honor CMAKE_REQUIRED_LIBRARIES
if(POLICY CMP0075)
  cmake_policy(SET CMP0075 NEW)
endif()

#
# Setup Options
#

option(MPI "Enables MPI-based communication and running coupling tests." ON)
option(PETSC "Enable use of the PETSc linear algebra library." ON)
option(PYTHON "Python support" ON)
option(PRECICE_Packages "Configure package generation." ON)
option(PRECICE_InstallTest "Add test binary and necessary files to install target." OFF)
option(BUILD_SHARED_LIBS "Build shared libraries by default" OFF)
option(PRECICE_ALWAYS_VALIDATE_LIBS "Validate libraries even after the validatation succeeded." OFF)
xsdk_tpl_option_override(MPI TPL_ENABLE_MPI)
xsdk_tpl_option_override(PETSC TPL_ENABLE_PETSC)
xsdk_tpl_option_override(PYTHON TPL_ENABLE_PYTHON)

if(PETSC AND NOT MPI)
  message(FATAL_ERROR "Please enable MPI to use PETSC.")
endif()

set(PRECICE_CTEST_MPI_FLAGS "" CACHE STRING "Add additional flags to mpiexec for running tests via CTest.")

include(XSDKOptions)

# Print information about this configuration
include(PrintHelper)
print_configuration(
  ADDITIONAL
  "MPI;Build with MPI"
  "PETSC;Build with PETSC"
  "PYTHON;Build with PYTHON"
  "PRECICE_Packages;Configure package generation"
  "PRECICE_InstallTest;Install tests/testfiles"
  "PRECICE_CTEST_MPI_FLAGS;Additional CTest MPI Flags"
  )


#
# Find Mandatory Dependencies
#

find_package (Threads REQUIRED)

if(TPL_ENABLE_BOOST)
  xsdk_tpl_require(BOOST BOOST_ROOT)
  # Use BOOST_ROOT to set the directory
  set(Boost_NO_SYSTEM_PATHS ON CACHE BOOL "" FORCE)
  unset(ENV{BOOST_ROOT})
endif()
find_package(Boost 1.65.1 REQUIRED
  COMPONENTS filesystem log log_setup program_options system thread unit_test_framework
  )

# Eigen
if(TPL_ENABLE_EIGEN3)
  # Use EIGEN3_ROOT to set the directory
  xsdk_tpl_require(EIGEN3 EIGEN3_INCLUDE_DIR)
endif()
find_package(Eigen3 3.2 REQUIRED)
precice_validate_eigen()

# LibXML2
if(TPL_ENABLE_LIBXML2)
  xsdk_tpl_require(LIBXML2 LIBXML2_LIBRARIES LIBXML2_INCLUDE_DIRS)
endif()
find_package(LibXml2 REQUIRED)
precice_validate_libxml2()

# nlohmann/JSON
if(TPL_ENABLE_JSON)
  xsdk_tpl_require(JSON JSON_INCLUDE_DIR)
  add_library(JSON INTERFACE IMPORTED)
  set_property(TARGET JSON PROPERTY INTERFACE_INCLUDE_DIRECTORIES "${JSON_INCLUDE_DIR}")
else()
  add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/json)
endif()
precice_validate_json()

# prettyprint
if(TPL_ENABLE_PRETTYPRINT)
  xsdk_tpl_require(PRETTYPRINT PRETTYPRINT_INCLUDE_DIR)
  add_library(prettyprint INTERFACE IMPORTED)
  set_property(TARGET prettyprint PROPERTY INTERFACE_INCLUDE_DIRECTORIES "${PRETTYPRINT_INCLUDE_DIR}")
else()
  add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/prettyprint)
endif()
precice_validate_prettyprint()


#
# Find Configurable Dependencies
#

# Option: MPI
if (MPI)
  find_package(MPI REQUIRED)
endif()

# Option: PETSC
if (PETSC)
  if (TPL_ENABLE_PETSC)
    xsdk_tpl_require(PETSC PETSC_DIR PETSC_ARCH)
    # PETSc detection uses primarily these ENVs
    unset(ENV{PETSC_DIR})
    unset(ENV{PETSC_ARCH})
  endif()
  find_package(PETSc 3.6 REQUIRED)
  # No validation required as PETSc does this internally

  set(PETSC_VERSIONS "")
  set(PETSC_VERSION_MAJOR "")
  set(PETSC_VERSION_MINOR "")
  string(REGEX MATCHALL "[0-9]+" PETSC_VERSIONS ${PETSC_VERSION})
  list(GET PETSC_VERSIONS 0 PETSC_VERSION_MAJOR)
  list(GET PETSC_VERSIONS 1 PETSC_VERSION_MINOR)
else()
  message(STATUS "PETSc support disabled")
endif()

# Option Python
if (PYTHON)
  if (TPL_ENABLE_PYTHON)
    xsdk_tpl_require(PYTHON PYTHON_LIBRARY PYTHON_INCLUDE_DIR NumPy_INCLUDE_DIR)
    find_package(PythonLibs REQUIRED)

    # Override NumPy
    # TODO: Incorporate into the FindNumPy module
    set(NumPy_FOUND True CACHE BOOL "NumPy found?" FORCE)
    if(NOT TARGET NumPy::NumPy)
      add_library(NumPy::NumPy INTERFACE IMPORTED)
      set_property(TARGET NumPy::NumPy PROPERTY INTERFACE_INCLUDE_DIRECTORIES "${NumPy_INCLUDE_DIR}")
    endif()
  else()
    find_package(PythonLibs REQUIRED)
    find_package(NumPy REQUIRED)
  endif()
  if(PYTHONLIBS_VERSION_STRING VERSION_LESS 2.7)
    message(FATAL_ERROR "The selected Python Version is too old: ${PYTHONLIBS_VERSION_STRING} (minimum 2.7)")
  endif()
  precice_validate_libpython()
  precice_validate_numpy()
else()
  message(STATUS "Python support disabled")
endif()


#
# Configuration of Target precice
#

# Add a dummy to silence add_library warning for cmake < 3.11.
if(CMAKE_VERSION VERSION_LESS "3.11")
  set(preCICE_DUMMY "${CMAKE_CURRENT_BINARY_DIR}/dummy.cpp")
  if(NOT EXISTS "${preCICE_DUMMY}")
    file(WRITE "${preCICE_DUMMY}" "")
  endif()
endif()

# Add precice as an empty target 
add_library(precice ${preCICE_DUMMY})
set_target_properties(precice PROPERTIES
  # precice is a C++11 project
  CXX_STANDARD 11
  SOVERSION ${preCICE_VERSION}
  )

# Setup Boost
target_compile_definitions(precice PRIVATE BOOST_ALL_DYN_LINK BOOST_ASIO_ENABLE_OLD_SERVICES)
target_link_libraries(precice PRIVATE
  Boost::boost
  Boost::filesystem
  Boost::log
  Boost::log_setup
  Boost::program_options
  Boost::system
  Boost::thread
  Boost::unit_test_framework
  )
if(UNIX OR APPLE OR MINGW)
  target_link_libraries(precice PRIVATE ${CMAKE_DL_LIBS})
endif()

# Setup Eigen3
target_link_libraries(precice PRIVATE Eigen3::Eigen)
target_compile_definitions(precice PRIVATE "$<$<CONFIG:DEBUG>:EIGEN_INITIALIZE_MATRICES_BY_NAN>")

# Setup LIBXML2
target_include_directories(precice PRIVATE ${LIBXML2_INCLUDE_DIR})
target_link_libraries(precice PRIVATE ${LIBXML2_LIBRARIES})

# Setup Prettyprint
target_link_libraries(precice PRIVATE prettyprint)

# Setup JSON
target_link_libraries(precice PRIVATE JSON)

# Setup MPI
if (MPI)
  target_link_libraries(precice PRIVATE MPI::MPI_CXX)
else()
  target_compile_definitions(precice PRIVATE PRECICE_NO_MPI)
endif()

# Setup PETSC
if (PETSC AND MPI)
  target_include_directories(precice PRIVATE ${PETSC_INCLUDES})
  target_link_libraries(precice PRIVATE ${PETSC_LIBRARIES})
else()
  target_compile_definitions(precice PRIVATE PRECICE_NO_PETSC)
endif()

# Option Python
if (PYTHON)
  target_link_libraries(precice PRIVATE NumPy::NumPy)
  target_include_directories(precice PRIVATE ${PYTHON_INCLUDE_DIRS})
  target_compile_definitions(precice PRIVATE NPY_NO_DEPRECATED_API=NPY_1_7_API_VERSION)
  target_link_libraries(precice PRIVATE ${PYTHON_LIBRARIES})
else()
  target_compile_definitions(precice PRIVATE PRECICE_NO_PYTHON)
endif()


# File Configuration
include(GenerateVersionInformation)
include(${CMAKE_CURRENT_LIST_DIR}/cmake/DetectGitRevision.cmake)
configure_file("${PROJECT_SOURCE_DIR}/src/precice/impl/versions.hpp.in" "${PROJECT_BINARY_DIR}/src/precice/impl/versions.hpp" @ONLY)

# Includes Configuration
target_include_directories(precice PUBLIC 
  $<BUILD_INTERFACE:${preCICE_SOURCE_DIR}/src>
  $<BUILD_INTERFACE:${preCICE_BINARY_DIR}/src>
  $<INSTALL_INTERFACE:include>
  )

# Sources Configuration
include(${CMAKE_CURRENT_LIST_DIR}/src/sources.cmake)


#
# Configuration of Target binprecice
#

add_executable(binprecice "src/drivers/main.cpp")
target_link_libraries(binprecice 
  PRIVATE
  Threads::Threads
  precice
  prettyprint
  Eigen3::Eigen
  Boost::boost
  Boost::filesystem
  Boost::log
  Boost::log_setup
  Boost::program_options
  Boost::system
  Boost::thread
  Boost::unit_test_framework
  )
set_target_properties(binprecice PROPERTIES
  # precice is a C++11 project
  CXX_STANDARD 11
  )
# Copy needed properties from the lib to the executatble. This is necessary as
# this executable uses the library source, not only the interface.
copy_target_property(precice binprecice COMPILE_DEFINITIONS)
copy_target_property(precice binprecice COMPILE_OPTIONS)
if(MPI)
  target_link_libraries(binprecice PRIVATE MPI::MPI_CXX)
endif()
if(MPI AND PETSC)
  target_include_directories(binprecice PRIVATE ${PETSC_INCLUDES})
  target_link_libraries(binprecice PRIVATE ${PETSC_LIBRARIES})
endif()


#
# Configuration of Target testprecice
#

add_executable(testprecice "src/testing/main.cpp")
target_link_libraries(testprecice
  PRIVATE
  Threads::Threads
  precice
  Eigen3::Eigen
  prettyprint
  Boost::boost
  Boost::filesystem
  Boost::log
  Boost::log_setup
  Boost::program_options
  Boost::system
  Boost::thread
  Boost::unit_test_framework
  )
set_target_properties(testprecice PROPERTIES
  # precice is a C++11 project
  CXX_STANDARD 11
  )
# Copy needed properties from the lib to the executatble. This is necessary as
# this executable uses the library source, not only the interface.
copy_target_property(precice testprecice COMPILE_DEFINITIONS)
copy_target_property(precice testprecice COMPILE_OPTIONS)

# Testprecice fully depends on MPI and PETSc.
if(MPI)
  target_link_libraries(testprecice PRIVATE MPI::MPI_CXX)
endif()
if(MPI AND PETSC)
  target_include_directories(testprecice PRIVATE ${PETSC_INCLUDES})
  target_link_libraries(testprecice PRIVATE ${PETSC_LIBRARIES})
endif()

# Test Sources Configuration
include(${CMAKE_CURRENT_LIST_DIR}/src/tests.cmake)


#
# Install Targets for precice
#

# Setup General Install for:
# precice - the library
# binprecice - the precice binary
install(TARGETS precice binprecice
  EXPORT preciceTargets
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  RUNTIME DESTINATION bin
  PUBLIC_HEADER DESTINATION include/precice
  INCLUDES DESTINATION include/precice
  )

if(PRECICE_InstallTest)
  # Install the testprecice target
  install(TARGETS testprecice
    EXPORT preciceTargets
    RUNTIME DESTINATION bin
    )

  # Install the resources necessary for the tests
  install(DIRECTORY src
    DESTINATION share/precice
    FILES_MATCHING
    PATTERN "*.xml"
    )
  install(DIRECTORY src/action/tests/
    DESTINATION share/precice/src/action/tests
    FILES_MATCHING
    PATTERN "*.py"
    )
endif()

# Export the Targets to install
install(EXPORT preciceTargets
  FILE preciceTargets.cmake
  NAMESPACE precice::
  DESTINATION lib/cmake/precice
  )

# Generate a Package Config File for precice
include(CMakePackageConfigHelpers)
write_basic_package_version_file("preciceConfigVersion.cmake"
  VERSION ${preCICE_VERSION}
  COMPATIBILITY SameMajorVersion
  )

# Install the Config and the ConfigVersion files
install(FILES "cmake/preciceConfig.cmake" "${preCICE_BINARY_DIR}/preciceConfigVersion.cmake"
  DESTINATION lib/cmake/precice
  )

# Setup the config in the build directory
export(EXPORT preciceTargets
  NAMESPACE precice::
  FILE "preciceTargets.cmake")
file(COPY "cmake/preciceConfig.cmake"
  DESTINATION "${preCICE_BINARY_DIR}")

# Add an alias to allow subprojects to use the namespaced name
add_library(precice::precice ALIAS precice)


# Set the directory used to prepare files for packaging
set(PRECICE_PACKAGING_DIR "${CMAKE_CURRENT_BINARY_DIR}/packaging")
mark_as_advanced(PRECICE_PACKAGING_DIR)

# Compress and install changelog and manpages
find_program(GZIP_EXE gzip DOC "The gzip executable")
if(GZIP_EXE)
  # Process the changelog
  message(STATUS "Compressing changelog")
  file(COPY CHANGELOG.md DESTINATION ${PRECICE_PACKAGING_DIR})
  execute_process(COMMAND "${GZIP_EXE}" "-9nf" "${PRECICE_PACKAGING_DIR}/CHANGELOG.md")
  # Install compressed changelog
  install(FILES ${PRECICE_PACKAGING_DIR}/CHANGELOG.md.gz
    DESTINATION share/doc/libprecice${preCICE_VERSION}
    RENAME changelog.gz
    )

  # Process manpages for binaries
  file(COPY docs/man/man1/binprecice.1 DESTINATION packaging/man1)
  if(PRECICE_InstallTest)
    file(COPY docs/man/man1/testprecice.1 DESTINATION packaging/man1)
  endif()
  file(GLOB PRECICE_MAN_PAGES "${PRECICE_PACKAGING_DIR}/man1/*.1")
  foreach(manpage ${PRECICE_MAN_PAGES})
    message(STATUS "Compressing manpage: ${manpage}")
    execute_process(COMMAND "${GZIP_EXE}" "-9nf" "${manpage}")
  endforeach()
  # Install compressed manpages
  install(DIRECTORY ${PRECICE_PACKAGING_DIR}/man1
    DESTINATION share/man
    )
else()
  message(WARNING "Installing uncompressed changelog and manpages")
  # Install uncompressed changelog
  install(FILES CHANGELOG.md
    DESTINATION share/doc/libprecice${preCICE_VERSION}
    RENAME changelog
    )
  # Install uncompressed manpages
  install(FILES docs/man/man1/binprecice.1
    DESTINATION share/man
    )
  if(PRECICE_InstallTest)
    install(FILES docs/man/man1/testprecice.1
      DESTINATION share/man
      )
  endif()
endif()

# Configure a pkg-config file
configure_file(
  "${PROJECT_SOURCE_DIR}/tools/packaging/debian/precice.pc.in"
  "lib/pkgconfig/libprecice.pc"
  @ONLY
  )
install(DIRECTORY "${preCICE_BINARY_DIR}/lib/pkgconfig" 
  DESTINATION lib
)

#
# Add uninstall
#

include(${CMAKE_CURRENT_LIST_DIR}/cmake/Uninstall.cmake)


#
# CTest
#

include(${CMAKE_CURRENT_LIST_DIR}/cmake/CTestConfig.cmake)


#
# Add test_install
#

include(${CMAKE_CURRENT_LIST_DIR}/cmake/TestInstall.cmake)


#
# CPack
#

if (PRECICE_Packages)
  include(${CMAKE_CURRENT_LIST_DIR}/cmake/CPackConfig.cmake)
endif()
