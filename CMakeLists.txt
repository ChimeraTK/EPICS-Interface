cmake_minimum_required(VERSION 3.16)

# Change project name, based on EPICS version
OPTION(EPICS_VERSION "Select major version of EPICS to build (3 or 7)")
if((EPICS_VERSION STREQUAL "3") OR (NOT EPICS_VERSION))
  project(ChimeraTK-EPICS3)
  set(EPICS_VER "epics3")
  set(EPICS_FULLVER "3.15.9")
elseif(EPICS_VERSION STREQUAL "7")
  project(ChimeraTK-EPICS7)
  set(EPICS_VER "epics7")
  set(EPICS_FULLVER "7.0.8")
else()
  message(FATAL_ERROR "Unknown EPICS version specified: ${EPICS_VERSION}!\n"
                      "Please pass to cmake command line one of the following:\n"
                      "-DEPICS_VERSION=3 to build against EPICS ver. 3.15.9\n"
                      "-DEPICS_VERSION=7 to build against EPICS ver. 7.0.8.1")
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules)

set(${PROJECT_NAME}_MAJOR_VERSION 02)
set(${PROJECT_NAME}_MINOR_VERSION 01)
set(${PROJECT_NAME}_PATCH_VERSION 01)
include(cmake/set_version_numbers.cmake)

find_package(readline REQUIRED)
# EPICS 7 requires the Yet Another JSON Library
if(EPICS_VER STREQUAL "epics7")
  find_package(libYajl REQUIRED)
endif()

# set directory to gather all resulting libraries
set(PROJECT_LIB_DIR ${PROJECT_BINARY_DIR}/lib)

# generate a list of all epics-base libraries, depending on epics version
set(epics3_BASE_LIBS ${PROJECT_LIB_DIR}/libdbRecStd.a
  ${PROJECT_LIB_DIR}/libdbCore.a
  ${PROJECT_LIB_DIR}/libca.a
  ${PROJECT_LIB_DIR}/libcas.a
  ${PROJECT_LIB_DIR}/libCom.a
  ${PROJECT_LIB_DIR}/libgdd.a)

set(epics7_BASE_LIBS ${PROJECT_LIB_DIR}/libdbRecStd.a
  ${PROJECT_LIB_DIR}/libdbCore.a
  ${PROJECT_LIB_DIR}/libca.a
  ${PROJECT_LIB_DIR}/libCom.a
  ${PROJECT_LIB_DIR}/libnt.a
  ${PROJECT_LIB_DIR}/libpvAccess.a
  ${PROJECT_LIB_DIR}/libpvAccessCA.a
  ${PROJECT_LIB_DIR}/libpvAccessIOC.a
  ${PROJECT_LIB_DIR}/libpvaClient.a
  ${PROJECT_LIB_DIR}/libpvData.a
  ${PROJECT_LIB_DIR}/libpvDatabase.a
  ${PROJECT_LIB_DIR}/libqsrv.a)


# build EPICS base
add_custom_target(${PROJECT_NAME}-base ALL COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildEpics "${CMAKE_CURRENT_SOURCE_DIR}/${EPICS_VER}-base"
                                           WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                                           BYPRODUCTS ${${EPICS_VER}_BASE_LIBS})

# build autosave
add_custom_target(${PROJECT_NAME}-autosave ALL COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildAutosave ${CMAKE_CURRENT_SOURCE_DIR}/plugins/autosave
                                               WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                                               DEPENDS ${PROJECT_NAME}-base
                                               BYPRODUCTS ${PROJECT_LIB_DIR}/libautosave.a)

# build calc
add_custom_target(${PROJECT_NAME}-calc ALL COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildCalc ${CMAKE_CURRENT_SOURCE_DIR}/plugins/calc
                                           WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
                                           DEPENDS ${PROJECT_NAME}-base
                                           BYPRODUCTS ${PROJECT_LIB_DIR}/libcalc.a)

# create combined shared library containing all EPICS libraries
add_library(${PROJECT_NAME} SHARED src/empty.cc)
target_link_libraries(${PROJECT_NAME} PUBLIC
  "-Wl,--whole-archive" ${${EPICS_VER}_BASE_LIBS} ${PROJECT_LIB_DIR}/libautosave.a ${PROJECT_LIB_DIR}/libcalc.a)
set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS "-Wl,--allow-multiple-definition -Wl,--no-allow-shlib-undefined")

# Install the combined library and export as cmake target
include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

set(PROVIDES_EXPORTED_TARGETS 1)
list(APPEND ${PROJECT_NAME}_PUBLIC_DEPENDENCIES)
include(${CMAKE_SOURCE_DIR}/cmake/create_cmake_config_files.cmake)
