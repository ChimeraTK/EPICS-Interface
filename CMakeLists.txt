cmake_minimum_required(VERSION 3.16)
if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
    cmake_policy(SET CMP0135 OLD)
endif()
project(ChimeraTK-EPICS)

# Change project name, based on EPICS version
OPTION(EPICS_VERSION "Select major version of EPICS to build (3 or 7)")

if(NOT EPICS_VERSION)
  set(EPICS_VERSION "7")
endif()

if((EPICS_VERSION STREQUAL "3"))
  set(EPICS_FULLVER "3.15.9")
elseif(EPICS_VERSION STREQUAL "7")
  set(EPICS_FULLVER "7.0.8")
else()
  message(FATAL_ERROR "Unknown EPICS version specified: ${EPICS_VERSION}!\n"
    "Please pass to cmake command line one of the following:\n"
    "-DEPICS_VERSION=3 to build against EPICS ver. 3.15.9\n"
    "-DEPICS_VERSION=7 to build against EPICS ver. 7.0.8.1")
endif()

set(${PROJECT_NAME}_MAJOR_VERSION 01)
set(${PROJECT_NAME}_MINOR_VERSION 01)
set(${PROJECT_NAME}_PATCH_VERSION 00)
include(cmake/set_version_numbers.cmake)

find_program(PERL perl REQUIRED)
include(FindPkgConfig)
if (NOT PKG_CONFIG_FOUND)
    message(FATAL_ERROR "EPICS needs pkg-config to work")
endif()
configure_file(configEpics.in configEpics @ONLY)

# determine EPICS host architecture string
execute_process(COMMAND perl ${CMAKE_SOURCE_DIR}/epics${EPICS_VERSION}-base/src/tools/EpicsHostArch.pl
  OUTPUT_VARIABLE EPICS_ARCH_TEMP RESULT_VARIABLE RESULT)

if(NOT ${RESULT} EQUAL 0)
  message(FATAL_ERROR "Cannot determine EPICS host architecture. Is perl installed?")
endif()

string(STRIP ${EPICS_ARCH_TEMP} EPICS_ARCH)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules)

find_package(readline REQUIRED)

# EPICS 7 requires the Yet Another JSON Library
if(EPICS_VERSION STREQUAL "7")
  find_package(libYajl REQUIRED)
endif()

find_program(PATCHELF patchelf)

if (PATCHELF)
  execute_process(COMMAND ${PATCHELF} "--version" OUTPUT_VARIABLE PATCHELF_OUTPUT)
  string(REPLACE " " ";" PATCHELF_VERSION ${PATCHELF_OUTPUT})
  list(POP_FRONT PATCHELF_VERSION)
  if (PATCHELF_VERSION VERSION_LESS "0.18.0")
      set(PATCHELF "PATCHELF-NOTFOUND")
  else()
      message("${PATCHELF} is >= 0.18, using system version")
  endif()
endif()

if (NOT PATCHELF)
    include(ExternalProject)
    ExternalProject_Add(patchelf-external
        SOURCE_DIR ${PROJECT_BINARY_DIR}/patchelf
        URL https://github.com/NixOS/patchelf/releases/download/0.18.0/patchelf-0.18.0.tar.gz
        BUILD_BYPRODUCTS patchelf-external-prefix/bin/patchelf
        PATCH ${CMAKE_CURRENT_SOURCE_DIR}/drop-unsupported-elf-tag.patch
        CONFIGURE_COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/patchelf/configure --prefix=<INSTALL_DIR> CFLAGS=-w CPPFLAGS=-w CXXFLAGS=-w
        BUILD_COMMAND ${MAKE}
    )
    ExternalProject_Get_property(patchelf-external install_dir)
    set(PATCHELF ${install_dir}/bin/patchelf)
    add_custom_target(patchelf-build DEPENDS patchelf-external)
else()
    add_custom_target(patchelf-build COMMAND echo "Nothing to do for patchelf.")
endif()

message("Patchelf is ${PATCHELF}")

# set directory to gather all resulting libraries
set(TARGET_DIR ${CMAKE_INSTALL_PREFIX}/share/ChimeraTK-EPICS${EPICS_VERSION})
set(TARGET_LIB_DIR ${TARGET_DIR}/lib/${EPICS_ARCH})
set(TEMP_TARGET_DIR ${CMAKE_BINARY_DIR}/epics-temp)
set(TEMP_TARGET_LIB_DIR ${TEMP_TARGET_DIR}/lib/${EPICS_ARCH})

# generate a list of all epics-base libraries, depending on epics version
# Note: The order of libraries is important, otherwise linkage will break in downstream projects! Libraries which
# depend on others must come before their dependencies.
if(EPICS_VERSION STREQUAL "3")
  set(EPICS_BASE_LIBS_RELATIVE libdbRecStd.so
    libdbCore.so
    libca.so
    libcas.so
    libCom.so
    libgdd.so)
else()
  set(EPICS_BASE_LIBS_RELATIVE libdbRecStd.so
    libdbCore.so
    libca.so
    libCom.so
    libnt.so
    libqsrv.so
    libpvAccessIOC.so
    libpvAccessCA.so
    libpvAccess.so
    libpvaClient.so
    libpvData.so
    libpvDatabase.so
  )
endif()

set(EPICS_BASE_LIBS ${EPICS_BASE_LIBS_RELATIVE})
list(TRANSFORM EPICS_BASE_LIBS PREPEND "${TARGET_LIB_DIR}/")
list(TRANSFORM TEMP_EPICS_BASE_LIBS PREPEND "${TEMP_TARGET_LIB_DIR}/")

# configure EPICS base
# The ulgy part is that we are going to install a few files into the CMAKE_INSTALL_PREFIX already at configuration time.
# There seems to be no other way to get to the pkg-config .pc file which contains valuable
# information for us (include directories).
execute_process(
  COMMAND ${PROJECT_BINARY_DIR}/configEpics "${CMAKE_CURRENT_SOURCE_DIR}/epics${EPICS_VERSION}-base" "${TARGET_DIR}" "${EPICS_ARCH}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR} RESULT_VARIABLE RESULT)

if(NOT ${RESULT} EQUAL 0)
  message(FATAL_ERROR "Cannot configure EPICS.")
endif()

file(READ ${PROJECT_BINARY_DIR}/include-dirs.txt INC_DIRS)

# build EPICS base
add_custom_target(${PROJECT_NAME}-base ALL
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildEpics "${TARGET_DIR}" "${EPICS_ARCH}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  BYPRODUCTS ${TEMP_EPICS_BASE_LIBS})

# build autosave
add_custom_target(${PROJECT_NAME}-autosave ALL
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildAutosave ${CMAKE_CURRENT_SOURCE_DIR}/plugins/autosave "${TEMP_TARGET_DIR}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  DEPENDS ${PROJECT_NAME}-base
  BYPRODUCTS ${TEMP_TARGET_LIB_DIR}/libautosave.so)

# build calc
add_custom_target(${PROJECT_NAME}-calc ALL
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildCalc ${CMAKE_CURRENT_SOURCE_DIR}/plugins/calc "${TEMP_TARGET_DIR}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  DEPENDS ${PROJECT_NAME}-base
  BYPRODUCTS ${TEMP_TARGET_LIB_DIR}/libcalc.so)

# fix shared library names and dependencies
configure_file(fixSharedLibs.in fixSharedLibs @ONLY)
add_custom_target(${PROJECT_NAME}-sharedlibs ALL
  COMMAND ${PROJECT_BINARY_DIR}/fixSharedLibs "${TARGET_DIR}" "${EPICS_ARCH}" "${EPICS_FULLVER}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  DEPENDS ${PROJECT_NAME}-base ${PROJECT_NAME}-autosave ${PROJECT_NAME}-calc patchelf-build)

# create interface library
add_library(EPICS INTERFACE)

# Note: order of libraries is important!
target_link_libraries(EPICS INTERFACE ${TARGET_LIB_DIR}/libautosave.so ${TARGET_LIB_DIR}/libcalc.so ${EPICS_BASE_LIBS})
target_link_libraries(EPICS INTERFACE readline)

if(EPICS_VERSION STREQUAL "7")
  # NOTICE: Due to https://github.com/lloyd/yajl/issues/209 the correct linker flag is -lyajl_s.
  # TODO: If this issue is resolved, the library might have to be changed from yail_s to yajl!
  target_link_libraries(EPICS INTERFACE yajl_s)
endif()

target_include_directories(EPICS INTERFACE ${INC_DIRS})

# This only works for newer cmake versions. Enable once we got rid of Ubunutu 20.04! Workaround is below...
# define_property(TARGET PROPERTY INTERFACE_EPICS_BASE)
# define_property(TARGET PROPERTY INTERFACE_EPICS_ARCH)
# set_property(TARGET EPICS APPEND PROPERTY EXPORT_PROPERTIES INTERFACE_EPICS_BASE INTERFACE_EPICS_ARCH)
# set_property(TARGET EPICS PROPERTY INTERFACE_EPICS_BASE ${TARGET_DIR})
# set_property(TARGET EPICS PROPERTY INTERFACE_EPICS_ARCH ${EPICS_ARCH})
add_library(ChimeraTK::EPICS ALIAS EPICS)

install(TARGETS EPICS EXPORT EPICS)

# this defines architecture-dependent ${CMAKE_INSTALL_LIBDIR}
include(GNUInstallDirs)

#
install(DIRECTORY ${TEMP_TARGET_DIR}/lib/${EPICS_ARCH}
  DESTINATION ${TARGET_DIR}/lib
  COMPONENT Runtime
  FILES_MATCHING PATTERN "*.so.*-ChimeraTK")

install(DIRECTORY ${TEMP_TARGET_DIR}/lib/${EPICS_ARCH}
  DESTINATION ${TARGET_DIR}/lib
  COMPONENT Development
  FILES_MATCHING PATTERN "*.so")

install(DIRECTORY ${TEMP_TARGET_DIR}/lib/perl ${TEMP_TARGET_DIR}/lib/pkgconfig
  DESTINATION ${TARGET_DIR}/lib
  COMPONENT Development)

install(DIRECTORY ${TEMP_TARGET_DIR}/cfg ${TEMP_TARGET_DIR}/db
  ${TEMP_TARGET_DIR}/dbd ${TEMP_TARGET_DIR}/include ${TEMP_TARGET_DIR}/templates
  DESTINATION ${TARGET_DIR}
  COMPONENT Development)

install(DIRECTORY ${TEMP_TARGET_DIR}/configure-patched/
  DESTINATION ${TARGET_DIR}/configure
  COMPONENT Development)

install(DIRECTORY ${TEMP_TARGET_DIR}/bin/${EPICS_ARCH}
  DESTINATION ${TARGET_DIR}/bin
  COMPONENT Development
  PATTERN "*"
  PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ GROUP_EXECUTE GROUP_READ WORLD_READ WORLD_EXECUTE)

install(DIRECTORY ${TEMP_TARGET_DIR}/html
  DESTINATION ${TARGET_DIR}
  COMPONENT Documentation)

# generate and install export file
install(EXPORT EPICS
  FILE ${PROJECT_NAME}Config.cmake
  NAMESPACE ChimeraTK::
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}${EPICS_VERSION}"
)

# This is the workaround for the missing support for additional properties at exported targets in older cmake verions.
configure_file(${CMAKE_SOURCE_DIR}/additionalExports.cmake.in
  ${CMAKE_BINARY_DIR}/${PROJECT_NAME}Config-properties.cmake)
install(FILES ${CMAKE_BINARY_DIR}/${PROJECT_NAME}Config-properties.cmake
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}${EPICS_VERSION}")
