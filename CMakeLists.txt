cmake_minimum_required(VERSION 3.16)

project(ChimeraTK-EPICS)

# Change project name, based on EPICS version
OPTION(EPICS_VERSION "Select major version of EPICS to build (3 or 7)")
if(NOT EPICS_VERSION)
  set(EPICS_VERSION "3")
endif()
if((EPICS_VERSION STREQUAL "3"))
  set(EPICS_VER "epics3")
  set(EPICS_FULLVER "3.15.9")
  set(${PROJECT_NAME}_MAJOR_VERSION 03)
  set(${PROJECT_NAME}_MINOR_VERSION 15)
  set(${PROJECT_NAME}_PATCH_VERSION 09)
elseif(EPICS_VERSION STREQUAL "7")
  set(EPICS_VER "epics7")
  set(EPICS_FULLVER "7.0.8")
  set(${PROJECT_NAME}_MAJOR_VERSION 07)
  set(${PROJECT_NAME}_MINOR_VERSION 00)
  set(${PROJECT_NAME}_PATCH_VERSION 08)
else()
  message(FATAL_ERROR "Unknown EPICS version specified: ${EPICS_VERSION}!\n"
                      "Please pass to cmake command line one of the following:\n"
                      "-DEPICS_VERSION=3 to build against EPICS ver. 3.15.9\n"
                      "-DEPICS_VERSION=7 to build against EPICS ver. 7.0.8.1")
endif()

string(TOUPPER ${EPICS_VER} EPICS_VER_UPPER)

# determine EPICS host architecture string
execute_process(COMMAND perl ${CMAKE_SOURCE_DIR}/${EPICS_VER}-base/src/tools/EpicsHostArch.pl
                OUTPUT_VARIABLE EPICS_ARCH_TEMP RESULT_VARIABLE RESULT)
if(NOT ${RESULT} EQUAL 0)
  message(FATAL_ERROR "Cannot determine EPICS host architecture. Is perl installed?")
endif()
string(STRIP ${EPICS_ARCH_TEMP} EPICS_ARCH)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules)

include(cmake/set_version_numbers.cmake)

find_package(readline REQUIRED)
# EPICS 7 requires the Yet Another JSON Library
if(EPICS_VER STREQUAL "epics7")
  find_package(libYajl REQUIRED)
endif()

# set directory to gather all resulting libraries
set(TARGET_DIR ${CMAKE_INSTALL_PREFIX}/share/ChimeraTK-${EPICS_VER_UPPER})
set(TARGET_LIB_DIR ${TARGET_DIR}/lib/${EPICS_ARCH})

# generate a list of all epics-base libraries, depending on epics version
if(EPICS_VER STREQUAL "epics3")
  set(EPICS_BASE_LIBS ${TARGET_LIB_DIR}/libdbRecStd.a
    ${TARGET_LIB_DIR}/libdbCore.a
    ${TARGET_LIB_DIR}/libca.a
    ${TARGET_LIB_DIR}/libcas.a
    ${TARGET_LIB_DIR}/libCom.a
    ${TARGET_LIB_DIR}/libgdd.a)
else()
  set(EPICS_BASE_LIBS ${TARGET_LIB_DIR}/libdbRecStd.a
    ${TARGET_LIB_DIR}/libdbCore.a
    ${TARGET_LIB_DIR}/libca.a
    ${TARGET_LIB_DIR}/libCom.a
    ${TARGET_LIB_DIR}/libnt.a
    ${TARGET_LIB_DIR}/libpvAccess.a
    ${TARGET_LIB_DIR}/libpvAccessCA.a
    ${TARGET_LIB_DIR}/libpvAccessIOC.a
    ${TARGET_LIB_DIR}/libpvaClient.a
    ${TARGET_LIB_DIR}/libpvData.a
    ${TARGET_LIB_DIR}/libpvDatabase.a
    ${TARGET_LIB_DIR}/libqsrv.a)
endif()

# configure EPICS base
# The ulgy part is that we are going to install a few files into the CMAKE_INSTALL_PREFIX already at configuration time.
# There seems to be no other way to get to the pkg-config .pc file which contains valuable
# information for us (include directories).
execute_process(
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/configEpics "${CMAKE_CURRENT_SOURCE_DIR}/${EPICS_VER}-base" "${TARGET_DIR}" "${EPICS_ARCH}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR} RESULT_VARIABLE RESULT)

if(NOT ${RESULT} EQUAL 0)
  message(FATAL_ERROR "Cannot configure EPICS.")
endif()

file(READ ${PROJECT_BINARY_DIR}/include-dirs.txt INC_DIRS)


# build EPICS base
add_custom_target(${PROJECT_NAME}-base ALL
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildEpics
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  BYPRODUCTS ${EPICS_BASE_LIBS})

# build autosave
add_custom_target(${PROJECT_NAME}-autosave ALL
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildAutosave ${CMAKE_CURRENT_SOURCE_DIR}/plugins/autosave "${TARGET_DIR}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  DEPENDS ${PROJECT_NAME}-base
  BYPRODUCTS ${TARGET_LIB_DIR}/libautosave.a)

# build calc
add_custom_target(${PROJECT_NAME}-calc ALL
COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/buildCalc ${CMAKE_CURRENT_SOURCE_DIR}/plugins/calc "${TARGET_DIR}"
  WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
  DEPENDS ${PROJECT_NAME}-base
  BYPRODUCTS ${TARGET_LIB_DIR}/libcalc.a)

# create interface library
set(LIB_NAME EPICS${EPICS_VERSION})
add_library(${LIB_NAME} INTERFACE)
target_link_libraries(${LIB_NAME} INTERFACE ${EPICS_BASE_LIBS} ${TARGET_LIB_DIR}/libautosave.a ${TARGET_LIB_DIR}/libcalc.a)
target_include_directories(${LIB_NAME} INTERFACE ${INC_DIRS})

define_property(TARGET PROPERTY EPICS_BASE)
define_property(TARGET PROPERTY EPICS_ARCH)
set_property(TARGET ${LIB_NAME} PROPERTY EPICS_BASE ${TARGET_DIR})
set_property(TARGET ${LIB_NAME} PROPERTY EPICS_ARCH ${TARGET_DIR})
set_property(TARGET ${LIB_NAME} APPEND PROPERTY EXPORT_PROPERTIES EPICS_BASE EPICS_ARCH)

add_library(ChimeraTK::${LIB_NAME} ALIAS ${LIB_NAME})

install(TARGETS ${LIB_NAME} EXPORT ${LIB_NAME})

# this defines architecture-dependent ${CMAKE_INSTALL_LIBDIR}
include(GNUInstallDirs)
# generate and install export file
install(EXPORT ${LIB_NAME}
  FILE ${PROJECT_NAME}${EPICS_VERSION}Config.cmake
  NAMESPACE ChimeraTK::
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}${EPICS_VERSION}"
)