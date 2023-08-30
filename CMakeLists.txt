cmake_minimum_required(VERSION 3.8...3.12 FATAL_ERROR)

if (NOT VTK_MODULE_SOURCE_DIR)
  message(FATAL_ERROR "Setting VTK_MODULE_SOURCE_DIR is required to lookup the VTK module sources.")
endif ()
message(STATUS "Setting VTK_MODULE_SOURCE_DIR to ${VTK_MODULE_SOURCE_DIR}")

if (NOT VTK_MODULE_NAME)
  message(FATAL_ERROR "Setting VTK_MODULE_NAME is required to configure the VTK module.")
endif ()
message(STATUS "Setting VTK_MODULE_NAME to ${VTK_MODULE_NAME}")

if(NOT DEFINED VTK_MODULE_SUPERBUILD)
  set(VTK_MODULE_SUPERBUILD OFF)
endif()
message(STATUS "Setting VTK_MODULE_SUPERBUILD to ${VTK_MODULE_SUPERBUILD}")

if(VTK_MODULE_SUPERBUILD)
  if(NOT DEFINED VTK_MODULE_EXTERNAL_PROJECT_DEPENDENCIES)
    message(FATAL_ERROR "Setting VTK_MODULE_EXTERNAL_PROJECT_DEPENDENCIES is required to configure the VTK module with VTK_MODULE_SUPERBUILD set to ON.")
  endif()
  message(STATUS "Setting VTK_MODULE_EXTERNAL_PROJECT_DEPENDENCIES to ${VTK_MODULE_EXTERNAL_PROJECT_DEPENDENCIES}")

  if(DEFINED VTK_MODULE_EXTERNAL_PROJECT_CMAKE_CACHE_ARGS)
    message(STATUS "Setting VTK_MODULE_EXTERNAL_PROJECT_CMAKE_CACHE_ARGS to ${VTK_MODULE_EXTERNAL_PROJECT_CMAKE_CACHE_ARGS}")
  endif()
endif()

# Update CMake module path
list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_CURRENT_SOURCE_DIR}/CMake")

# Superbuild
set(SUPERBUILD_TOPLEVEL_PROJECT vtk${VTK_MODULE_NAME})
set(EXTERNAL_PROJECT_DIR ${VTK_MODULE_SOURCE_DIR}/SuperBuild)
set(EXTERNAL_PROJECT_FILE_PREFIX "External_")
include(ExternalProject)
include(ExternalProjectDependency)

if(NOT DEFINED ${SUPERBUILD_TOPLEVEL_PROJECT}_SUPERBUILD)
  set(${SUPERBUILD_TOPLEVEL_PROJECT}_SUPERBUILD ${VTK_MODULE_SUPERBUILD})
endif()

if(VTK_MODULE_SUPERBUILD)
  message(STATUS "Setting SUPERBUILD_TOPLEVEL_PROJECT to ${SUPERBUILD_TOPLEVEL_PROJECT}")
  message(STATUS "Setting EXTERNAL_PROJECT_DIR to ${EXTERNAL_PROJECT_DIR}")
  message(STATUS "Setting EXTERNAL_PROJECT_FILE_PREFIX to ${EXTERNAL_PROJECT_FILE_PREFIX}")
endif()

mark_as_superbuild(
  VARS
    VTK_MODULE_NAME:STRING
    VTK_MODULE_SOURCE_DIR:PATH
  )

if (VTK_MODULE_CMAKE_MODULE_PATH)
  message(STATUS "Setting VTK_MODULE_CMAKE_MODULE_PATH to ${VTK_MODULE_CMAKE_MODULE_PATH}")
endif ()
mark_as_superbuild(VTK_MODULE_CMAKE_MODULE_PATH:PATH)

if (VTK_KIT_FILEPATH)
  message(STATUS "Setting VTK_KIT_FILEPATH to ${VTK_KIT_FILEPATH}")
endif ()
mark_as_superbuild(VTK_KIT_FILEPATH:FILEPATH)

project(${SUPERBUILD_TOPLEVEL_PROJECT} VERSION 0.1.0)

if (DEFINED VTK_MODULE_ENABLE_VTK_${VTK_MODULE_NAME} AND NOT "${CMAKE_PROJECT_NAME}" STREQUAL "${PROJECT_NAME}")
  # When building as a remote module for VTK 9 (or VTK 8.90+)
  message(FATAL_ERROR "${PROJECT_NAME}: Building as a Remote VTK Module should NOT be done using add_subdirectory()")
  return()
endif ()

if(DEFINED Python3_EXECUTABLE)
  mark_as_superbuild(Python3_EXECUTABLE:FILEPATH)
endif()

option(BUILD_PYTHON_WRAPPERS "Build python wrappers" ON)
option(VTK_WHEEL_BUILD "Build wheel version of libraries." OFF)
option(BUILD_JAVA_WRAPPERS "Build java wrappers" OFF)

# Adapted from "VTK/CMake/vtkCrossCompiling.cmake"
if(CMAKE_CROSSCOMPILING
    AND NOT VTKCompileTools_FOUND
    AND NOT DEFINED CMAKE_CROSSCOMPILING_EMULATOR
    AND (BUILD_JAVA_WRAPPERS OR BUILD_PYTHON_WRAPPERS))
  # if CMAKE_CROSSCOMPILING is true and crosscompiling emulator is not available, we need
  # to import build-tools targets.
  find_package(VTKCompileTools REQUIRED)
endif()

find_package(VTK REQUIRED)
if(DEFINED VTK_DIR)
  mark_as_superbuild(VTK_DIR:PATH)
endif()

if (VTK_VERSION VERSION_LESS 8.90)
  message(FATAL_ERROR "VTK 8.90 or later is required.")
endif ()

option(BUILD_SHARED_LIBS "Build shared libraries." ON)
mark_as_superbuild(BUILD_SHARED_LIBS)

if (VTK_MODULE_CMAKE_MODULE_PATH)
  list(INSERT CMAKE_MODULE_PATH 0 ${VTK_MODULE_CMAKE_MODULE_PATH})
endif ()

# Wrap Python option
if (NOT VTK_WRAP_PYTHON)
  unset(BUILD_PYTHON_WRAPPERS CACHE)
  unset(VTK_WHEEL_BUILD CACHE)
endif ()
mark_as_superbuild(
  VARS
    BUILD_PYTHON_WRAPPERS:BOOL
    VTK_WHEEL_BUILD:BOOL
  )

# Wrap Java option
if (NOT VTK_WRAP_JAVA)
  unset(BUILD_JAVA_WRAPPERS CACHE)
endif ()
mark_as_superbuild(BUILD_JAVA_WRAPPERS:BOOL)

# Defines CMAKE_INSTALL_LIBDIR etcetera
include(GNUInstallDirs)
if(DEFINED CMAKE_INSTALL_LIBDIR)
  mark_as_superbuild(CMAKE_INSTALL_LIBDIR:STRING)
endif()

# Set up our directory structure for output libraries and binaries
# (Note: these are the build locations, not the install locations)
if (NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR}")
endif ()
if (NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  if (UNIX)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
  else ()
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR}")
  endif ()
endif ()
if (NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR}")
endif ()

mark_as_superbuild(
  VARS
    CMAKE_RUNTIME_OUTPUT_DIRECTORY:PATH
    CMAKE_LIBRARY_OUTPUT_DIRECTORY:PATH
    CMAKE_ARCHIVE_OUTPUT_DIRECTORY:PATH
  )

mark_as_superbuild(
  VARS
    CMAKE_INSTALL_BINDIR:PATH
    CMAKE_INSTALL_LIBDIR:PATH
  )

if (DEFINED VTK_INSTALL_RUNTIME_DIR)
  message(AUTHOR_WARNING "Setting VTK_INSTALL_RUNTIME_DIR has no effect. Set CMAKE_INSTALL_BINDIR instead.")
endif ()
if (DEFINED VTK_INSTALL_LIBRARY_DIR)
  message(AUTHOR_WARNING "Setting VTK_INSTALL_LIBRARY_DIR has no effect. Set CMAKE_INSTALL_LIBDIR instead.")
endif ()

set(_vtk_module_wrap_python_additional_args)
set(_vtk_module_runtime_destination "${CMAKE_INSTALL_BINDIR}")
set(_vtk_module_library_destination "${CMAKE_INSTALL_LIBDIR}")
if (VTK_WRAP_PYTHON AND VTK_WHEEL_BUILD)
  # Following settings adapted from "VTK/CMake/vtkWheelPreparation.cmake"
  if (APPLE)
    # macOS loader settings.
    set(CMAKE_BUILD_WITH_INSTALL_NAME_DIR ON)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
    set(CMAKE_INSTALL_NAME_DIR "@rpath")
    list(APPEND CMAKE_INSTALL_RPATH
      "@loader_path")
  elseif (UNIX)
    # ELF loader settings.
    set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
    list(APPEND CMAKE_INSTALL_RPATH
      "$ORIGIN")
  endif ()
  set(VTK_PYTHON_OPTIONAL_LINK ON)
  # These need to be set so the libraries all end up in lib/vtkmodules/
  set(_vtk_module_library_destination "${CMAKE_INSTALL_LIBDIR}/vtkmodules")
  set(_vtk_module_wrap_python_additional_args MODULE_DESTINATION "${CMAKE_INSTALL_LIBDIR}")
  if (WIN32)
    # Ensure the dlls end up in lib/vtkmodules as well
    set(_vtk_module_runtime_destination "${CMAKE_INSTALL_LIBDIR}/vtkmodules")
  endif ()
endif()

# Testing
set(VTK_BUILD_TESTING "OFF"
  CACHE STRING "Build module testing directories")
set_property(CACHE VTK_BUILD_TESTING
  PROPERTY
    STRINGS "ON;OFF;WANT")
mark_as_superbuild(VTK_BUILD_TESTING:STRING)

if (VTK_BUILD_TESTING)
  include(CTest)
  set_property(CACHE BUILD_TESTING
    PROPERTY
      TYPE INTERNAL)
  set(BUILD_TESTING ON)
else ()
  set(BUILD_TESTING OFF)
endif ()

set(ExternalData_SOURCE_ROOT ${VTK_MODULE_SOURCE_DIR})

if(VTK_MODULE_SUPERBUILD)
  set(proj ${SUPERBUILD_TOPLEVEL_PROJECT})
  
  if(APPLE)
    mark_as_superbuild(
      VARS CMAKE_OSX_ARCHITECTURES:STRING CMAKE_OSX_SYSROOT:PATH CMAKE_OSX_DEPLOYMENT_TARGET:STRING
      ALL_PROJECTS
      )
  endif()

  foreach(varname IN ITEMS
    CMAKE_CXX_STANDARD
    CMAKE_CXX_STANDARD_REQUIRED
    CMAKE_CXX_EXTENSIONS
    SKBUILD
    VTKCompileTools_DIR
    )
    if(DEFINED ${varname})
      mark_as_superbuild(${varname})
    endif()
  endforeach()

  foreach(varname IN ITEMS
    CMAKE_CROSSCOMPILING
    CMAKE_CROSSCOMPILING_EMULATOR
    CMAKE_SYSTEM_NAME
    )
    if(DEFINED ${varname})
      mark_as_superbuild(VARS ${varname} ALL_PROJECTS)
    endif()
  endforeach()

  if(DEFINED VTK_MODULE_EXTERNAL_PROJECT_CMAKE_CACHE_ARGS)
    foreach(varname IN LISTS VTK_MODULE_EXTERNAL_PROJECT_CMAKE_CACHE_ARGS)
      if(DEFINED ${varname})
        mark_as_superbuild(${varname})
      endif()
    endforeach()
  endif()

  ExternalProject_Include_Dependencies(${proj} PROJECT_VAR proj DEPENDS_VAR VTK_MODULE_EXTERNAL_PROJECT_DEPENDENCIES)

  ExternalProject_Add(${proj}
    ${${proj}_EP_ARGS}
    DEPENDS ${VTK_MODULE_EXTERNAL_PROJECT_DEPENDENCIES}
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}
    BINARY_DIR ${CMAKE_BINARY_DIR}/inner-build
    DOWNLOAD_COMMAND ""
    UPDATE_COMMAND ""
    CMAKE_CACHE_ARGS
      -DCMAKE_CXX_COMPILER:FILEPATH=${CMAKE_CXX_COMPILER}
      -DCMAKE_CXX_FLAGS:STRING=${CMAKE_CXX_FLAGS}
      -DCMAKE_C_COMPILER:FILEPATH=${CMAKE_C_COMPILER}
      -DCMAKE_C_FLAGS:STRING=${CMAKE_C_FLAGS}
      -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_INSTALL_PREFIX}
      -DVTK_MODULE_SUPERBUILD:BOOL=OFF
      ${EXTERNAL_PROJECT_OPTIONAL_CMAKE_CACHE_ARGS}
    INSTALL_COMMAND ""
    )
  ExternalProject_AlwaysConfigure(${proj})
  
  # Install inner project
  install(CODE "
include(\"${CMAKE_BINARY_DIR}/inner-build/cmake_install.cmake\")
")
  return()
endif()

# Search for "vtk.module" files in all subdirectories.
vtk_module_find_modules(vtk_module_files ${VTK_MODULE_SOURCE_DIR})

set(_vtk_module_scan_kit_args)
if (VTK_KIT_FILEPATH)
  set(_vtk_module_scan_kit_args
    KIT_FILES ${VTK_KIT_FILEPATH}
    PROVIDES_KITS _vtk_kits
    )
endif ()

vtk_module_scan(
  MODULE_FILES     "${vtk_module_files}"
  ${_vtk_module_scan_kit_args}
  REQUEST_MODULES  "VTK::${VTK_MODULE_NAME}"
  PROVIDES_MODULES _modules
  ENABLE_TESTS     "${BUILD_TESTING}")

set(vtk_module_build_kit_args)
if (VTK_KIT_FILEPATH)
  set(vtk_module_build_kit_args
    KITS ${_vtk_kits}
    )
endif ()

foreach(_vtk_build_module IN LISTS _modules)
  get_property("_vtk_build_${_vtk_build_module}_depends" GLOBAL
    PROPERTY "_vtk_module_${_vtk_build_module}_depends")
  get_property("_vtk_build_${_vtk_build_module}_private_depends" GLOBAL
    PROPERTY "_vtk_module_${_vtk_build_module}_private_depends")

  foreach(_vtk_build_depend IN LISTS _vtk_build_${_vtk_build_module}_depends _vtk_build_${_vtk_build_module}_private_depends)
    if (NOT TARGET "${_vtk_build_depend}")
      # Attempt to import externally build module `vtk<ModuleName>`.
      # Considering `_vtk_build_depend` is formatted as `VTK::<ModuleName>`,
      # replace `^VTK::` with `vtk`.
      string(REGEX REPLACE "^VTK\:\:" "vtk" _vtk_build_depend_package ${_vtk_build_depend})
      find_package(${_vtk_build_depend_package} QUIET)
    endif()
  endforeach()
endforeach()

string(TOLOWER ${PROJECT_NAME} _project_name_lc)
vtk_module_build(
  MODULES               ${_modules}
  ${vtk_module_build_kit_args}
  INSTALL_EXPORT        ${PROJECT_NAME}
  ARCHIVE_DESTINATION   "lib"
  HEADERS_DESTINATION   "include/${_project_name_lc}-${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}"
  CMAKE_DESTINATION     "${CMAKE_INSTALL_LIBDIR}/cmake/vtk"
  LICENSE_DESTINATION   "${CMAKE_INSTALL_LIBDIR}/vtk"
  HIERARCHY_DESTINATION "${CMAKE_INSTALL_LIBDIR}/vtk/hierarchy/${CMAKE_PROJECT_NAME}"
  RUNTIME_DESTINATION   "${_vtk_module_runtime_destination}"
  LIBRARY_DESTINATION   "${_vtk_module_library_destination}"
  VERSION               "${PROJECT_VERSION}"
  SOVERSION             "1"
  TEST_DATA_TARGET      ${PROJECT_NAME}Data
  USE_EXTERNAL          "1"
  TEST_INPUT_DATA_DIRECTORY   "${_test_data_directory_input}"
  TEST_OUTPUT_DATA_DIRECTORY  "${_test_data_directory_output}")

if (BUILD_PYTHON_WRAPPERS)
  find_package(PythonInterp ${VTK_PYTHON_VERSION} QUIET)

  vtk_module_wrap_python(
    MODULES             ${_modules}
    INSTALL_EXPORT      ${PROJECT_NAME}Python
    PYTHON_PACKAGE      "vtkmodules"
    SOABI               "${Python${VTK_PYTHON_VERSION}_SOABI}"
    CMAKE_DESTINATION   "${CMAKE_INSTALL_LIBDIR}/cmake/vtk"
    LIBRARY_DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    BUILD_STATIC        OFF
    ${_vtk_module_wrap_python_additional_args})
endif ()

if (BUILD_JAVA_WRAPPERS)
  vtk_module_wrap_java(
    MODULES         ${_modules}
    WRAPPED_MODULES _java_wrapped_modules
    JAVA_OUTPUT     "${CMAKE_CURRENT_BINARY_DIR}/CMakeFiles")
endif ()

set(vtk_python_version "")
if (VTK_WRAP_PYTHON)
  set(vtk_python_version "${VTK_PYTHON_VERSION}")
endif ()

# Copied from vtkInstallCMakePackageHelpers.cmake
function (_vtk_package_append_variables)
  set(_vtk_package_variables)
  foreach (var IN LISTS ARGN)
    if (NOT DEFINED "${var}")
      continue ()
    endif ()

    get_property(type_is_set CACHE "${var}"
      PROPERTY TYPE SET)
    if (type_is_set)
      get_property(type CACHE "${var}"
        PROPERTY TYPE)
    else ()
      set(type UNINITIALIZED)
    endif ()

    string(APPEND _vtk_package_variables
      "if (NOT DEFINED \"${var}\" OR NOT ${var})
  set(\"${var}\" \"${${var}}\" CACHE ${type} \"Third-party helper setting from \${CMAKE_FIND_PACKAGE_NAME}\")
endif ()
")
  endforeach ()

  set(vtk_find_package_code
    "${vtk_find_package_code}${_vtk_package_variables}"
    PARENT_SCOPE)
endfunction ()

# Collect external VTK package dependencies
set(_vtk_external_package_depends)
foreach (_vtk_build_module IN LISTS _modules)
  get_property("_depends" GLOBAL
    PROPERTY "_vtk_module_${_vtk_build_module}_depends")
  get_property("_private_depends" GLOBAL
    PROPERTY "_vtk_module_${_vtk_build_module}_private_depends")
  foreach (_vtk_build_depend IN LISTS _depends _private_depends)
    # Considering `_vtk_build_depend` is formatted as `VTK::<ModuleName>`,
    # replace `^VTK::` with `vtk`.
    string(REGEX REPLACE "^VTK\:\:" "vtk" _vtk_build_depend_package ${_vtk_build_depend})
    if (DEFINED ${_vtk_build_depend_package}_DIR OR ${_vtk_build_depend_package}_ROOT)
      list(APPEND _vtk_external_package_depends ${_vtk_build_depend_package})
    endif ()
  endforeach ()
endforeach ()

# Collect packages searched using vtk_module_find_package()
get_property(_vtk_module_find_packages GLOBAL
  PROPERTY _vtk_module_find_packages_${PROJECT_NAME})

# Set code for configuring '<projectname>-vtk-find-package-helpers.cmake'
set(vtk_find_package_code)
foreach (_vtk_package IN LISTS _vtk_external_package_depends _vtk_module_find_packages)
  _vtk_package_append_variables(
    # Standard CMake `find_package` mechanisms.
    "${_vtk_package}_DIR"
    "${_vtk_package}_ROOT"

    # Per-package custom variables.
    ${${_vtk_package}_FIND_PACKAGE_VARS}
  )
endforeach ()

# Configure '<projectname>-config.cmake' for a build tree
set(_cmake_dir
    "${CMAKE_CURRENT_SOURCE_DIR}/CMake")
set(_cmake_destination
    "${CMAKE_INSTALL_LIBDIR}/cmake/vtk")
set(_cmake_build_dir
    "${CMAKE_CURRENT_BINARY_DIR}/${_cmake_destination}")

string(TOLOWER ${PROJECT_NAME} _project_name_lc)
configure_file(
  "${_cmake_dir}/vtkmodule-config.cmake.in"
  "${_cmake_build_dir}/${_project_name_lc}-config.cmake"
  @ONLY)

# Configure '<projectname>-vtk-find-package-helpers.cmake' for a build tree
file(GENERATE
  OUTPUT  "${_cmake_build_dir}/${PROJECT_NAME}-vtk-find-package-helpers.cmake"
  CONTENT "${vtk_find_package_code}")

# For convenience, a package is written to the top of the build tree.
file(GENERATE
  OUTPUT  "${CMAKE_BINARY_DIR}/${_project_name_lc}-config.cmake"
  CONTENT "include(\"${_cmake_build_dir}/${_project_name_lc}-config.cmake\")\n")

# Configure '<projectname>-config.cmake' for a install tree

# TODO

vtk_module_export_find_packages(
  CMAKE_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/vtk"
  FILE_NAME         "${PROJECT_NAME}-vtk-module-find-packages.cmake"
  MODULES           ${_modules})

