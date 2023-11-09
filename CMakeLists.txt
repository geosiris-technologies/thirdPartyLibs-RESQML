cmake_minimum_required(VERSION 3.23.1)

# ###############################
# GEOS_TPL_RESQML
# ###############################
project(geosx_tpl_resqml LANGUAGES C CXX)

set(GEOSX_TPL_DIR "$ENV{GEOSX_TPL_DIR}" CACHE PATH "" FORCE)
if (NOT DEFINED GEOSX_TPL_DIR)
  message(FATAL_ERROR "You must set GEOSX_TPL_DIR with -D GEOSX_TPL_DIR=")
endif ()

# ###############################
# BLT
# ###############################
set(BLT_CXX_STD c++17 CACHE STRING "" FORCE)

if(DEFINED BLT_SOURCE_DIR)
    # Support having a shared BLT outside of the repository if given a BLT_SOURCE_DIR
    if(NOT EXISTS ${BLT_SOURCE_DIR}/SetupBLT.cmake)
        message(FATAL_ERROR "Given BLT_SOURCE_DIR does not contain SetupBLT.cmake")
    endif()
else()
    # Use internal BLT if no BLT_SOURCE_DIR is given
    set(BLT_SOURCE_DIR "${PROJECT_SOURCE_DIR}/cmake/blt" CACHE PATH "")

    if(NOT EXISTS ${BLT_SOURCE_DIR}/SetupBLT.cmake)
        message(FATAL_ERROR
            "The BLT submodule is not present. "
            "Run the following two commands in your git repository: \n"
            "    git submodule init\n"
            "    git submodule update" )
        
    endif()
endif()

include(${BLT_SOURCE_DIR}/SetupBLT.cmake)

# ###############################
# General GEOSX TPL Info
# ###############################
include(ExternalProject)

option(BUILD_SHARED_LIBS "" OFF)

include(ProcessorCount)
ProcessorCount(NUM_CORES)

set(NUM_PROC ${NUM_CORES} CACHE STRING "")
message("Running with NUM_PROC = ${NUM_PROC}")

set(TPL_MIRROR_DIR "${CMAKE_SOURCE_DIR}/tplMirror")
set(build_list)

set(CMAKE_VERBOSE_MAKEFILE BOOL OFF "")

# Set up C flags
string(REPLACE "-Wall" "" C_FLAGS_NO_WARNINGS ${CMAKE_C_FLAGS})
string(REPLACE "-Wextra" "" C_FLAGS_NO_WARNINGS ${C_FLAGS_NO_WARNINGS})
set(C_FLAGS_NO_WARNINGS "${C_FLAGS_NO_WARNINGS} -w")
string(STRIP ${C_FLAGS_NO_WARNINGS} C_FLAGS_NO_WARNINGS)

# Set up CXX flags
string(REPLACE "-Wall" "" CXX_FLAGS_NO_WARNINGS ${CMAKE_CXX_FLAGS})
string(REPLACE "-Wextra" "" CXX_FLAGS_NO_WARNINGS ${CXX_FLAGS_NO_WARNINGS})
set(CXX_FLAGS_NO_WARNINGS "${CXX_FLAGS_NO_WARNINGS} -w")
string(STRIP ${CXX_FLAGS_NO_WARNINGS} CXX_FLAGS_NO_WARNINGS)

string(TOUPPER ${CMAKE_BUILD_TYPE} BUILD_TYPE_UPPER)

# Setup Ninja support
if(ENABLE_NINJA)
    message(STATUS "Ninja is enabled")
    set(TPL_BUILD_COMMAND ninja)
    set(TPL_INSTALL_COMMAND ninja install)
    set(TPL_GENERATOR Ninja)
else()
    set(TPL_BUILD_COMMAND make -j ${NUM_PROC})
    set(TPL_INSTALL_COMMAND make install)
    set(TPL_GENERATOR "Unix Makefiles")
endif()

message(STATUS "Build command = ${TPL_BUILD_COMMAND}")

include(GNUInstallDirs)

set(HDF5_DIR ${GEOSX_TPL_DIR}/hdf5)

# ###############################
# Fesapi
# ###############################

#
# ENABLE Boost
#
set(BOOST_DIR "${CMAKE_INSTALL_PREFIX}/boost")
set(BOOST_URL "${TPL_MIRROR_DIR}/boost_1_67_0.tar.gz")
message(STATUS "Extract Boost found at ${BOOST_URL}")

set(BOOST_ROOT ${CMAKE_CURRENT_BINARY_DIR}/boost/src/boost/ CACHE STRING "Set the value of BOOST_ROOT to point to the root folder of your boost install." FORCE)

ExternalProject_Add(boost
    URL ${BOOST_URL}
    PREFIX ${PROJECT_BINARY_DIR}/boost
    INSTALL_DIR ${BOOST_DIR}
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE 1
)

list(APPEND build_list boost)

#
# ENABLE Minizip
#

# Depending on the platform, the install directory could be 'lib' or 'lib64'.
# Fesapi doesn't use CMake Targets to import Minizip but needs an explicit
# path thus it has to be fixed regardless of the platform
set(MINIZIP_CMAKE_INSTALL_LIBDIR "lib")

set(MINIZIP_DIR "${CMAKE_INSTALL_PREFIX}/minizip")
set(MINIZIP_URL "${TPL_MIRROR_DIR}/Minizip-1.1_cmake0.1.tar.gz")
message(STATUS "Building Minizip found at ${MINIZIP_URL}")

ExternalProject_Add(minizip
    URL ${MINIZIP_URL}
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/minizip
    INSTALL_DIR ${MINIZIP_DIR}
    BUILD_COMMAND ${TPL_BUILD_COMMAND}
    INSTALL_COMMAND ${TPL_INSTALL_COMMAND}
    CMAKE_ARGS -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
    -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    -DCMAKE_CXX_FLAGS=${TPL_CXX_STANDARD}
    -DCMAKE_PREFIX_PATH:PATH=<INSTALL_DIR>
    -DCMAKE_INSTALL_LIBDIR=${MINIZIP_CMAKE_INSTALL_LIBDIR}
    -DCMAKE_CXX_STANDARD=17
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
)

list(APPEND build_list minizip)

if(ENABLE_MPI)
    set(FESAPI_C_COMPILER ${MPI_C_COMPILER})
    set(FESAPI_CXX_COMPILER ${MPI_CXX_COMPILER})
else()
    set(FESAPI_C_COMPILER ${CMAKE_C_COMPILER})
    set(FESAPI_CXX_COMPILER ${CMAKE_CXX_COMPILER})
endif(ENABLE_MPI)

#
# Then Fesapi
#

# Depending on the platform, the install directory could be 'lib' or 'lib64'.
# geos doesn't use CMake Targets to import Fesapi but needs an explicit
# path thus it has to be fixed regardless of the platform
set(FESAPI_CMAKE_INSTALL_LIBDIR "lib")

set(FESAPI_DIR "${CMAKE_INSTALL_PREFIX}/fesapi")
set(FESAPI_URL "${TPL_MIRROR_DIR}/fesapi-2.8.0.0.tar.gz")
message(STATUS "Building Fesapi found at ${FESAPI_URL}")

ExternalProject_Add(fesapi
    URL ${FESAPI_URL}
    PREFIX ${PROJECT_BINARY_DIR}/fesapi
    INSTALL_DIR ${FESAPI_DIR}
    BUILD_COMMAND ${TPL_BUILD_COMMAND}
    INSTALL_COMMAND ${TPL_INSTALL_COMMAND}
    DEPENDS minizip boost
    CMAKE_ARGS     
    -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>    
    -DCMAKE_C_COMPILER=${FESAPI_C_COMPILER}
    -DCMAKE_CXX_COMPILER=${FESAPI_CXX_COMPILER}
    -DCMAKE_CXX_FLAGS=${TPL_CXX_STANDARD}
    -DCMAKE_INSTALL_RPATH_USE_LINK_PATH:BOOL=TRUE
    -DCMAKE_INSTALL_LIBDIR=${FESAPI_CMAKE_INSTALL_LIBDIR}
    -DCMAKE_CXX_STANDARD=17
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DHDF5_ROOT:PATH=${HDF5_DIR}
    -DHDF5_PREFER_PARALLEL:BOOL=TRUE
    -DMINIZIP_INCLUDE_DIR:PATH=${MINIZIP_DIR}/include
    -DMINIZIP_LIBRARY_RELEASE:PATH=${MINIZIP_DIR}/lib/libminizip.a
    -DBoost_NO_SYSTEM_PATHS:BOOL=TRUE
    -DBoost_NO_BOOST_CMAKE:BOOL=TRUE
    -DBoost_INCLUDE_DIR:PATH=${BOOST_ROOT}
    -DWITH_EXAMPLE:BOOL=OFF
    -DWITH_DOTNET_WRAPPING:BOOL=OFF
    -DWITH_JAVA_WRAPPING:BOOL=OFF
    -DWITH_PYTHON_WRAPPING:BOOL=OFF
    -DWITH_RESQML2_2:BOOL=ON
    -DWITH_TEST:BOOL=OFF)

list(APPEND build_list fesapi)

# ###############################
# Create target that builds all dependencies
# ###############################
message(STATUS "Building = ${build_list}")

blt_add_executable(NAME tpl
    SOURCES tpl.cpp)

add_dependencies(tpl ${build_list})

# Clean up
add_custom_command(TARGET tpl POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E remove -f ${PROJECT_BINARY_DIR}/config_suitesparse_for_geosx
    COMMAND ${CMAKE_COMMAND} -E remove -f ${PROJECT_BINARY_DIR}/build_suitesparse_for_geosx
    COMMAND ${CMAKE_COMMAND} -E remove -f ${PROJECT_BINARY_DIR}/install_suitesparse_for_geosx)
