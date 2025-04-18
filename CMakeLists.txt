cmake_minimum_required(VERSION 3.15.0)

project(vmec LANGUAGES C CXX Fortran)
OPTION (USE_PROFILER "Activate code profiling." OFF)
if (USE_PROFILER)
    add_definitions (-DPROFILE_ON)
endif ()

set(CMAKE_VERBOSE_MAKEFILE ON CACHE BOOL "ON")

# Activate or Deactivate fast copies. The fast copies maybe activated by setting -DUSE_FAST_COPY=ON on the cmake command line.
OPTION (USE_FAST_COPY "Activate code fast copies." OFF)
if (USE_FAST_COPY)
	add_definitions (-DFAST_COPY)
endif ()

# Add the ability to auto generate API documentation with Doxygen
#find_package (Doxygen)
#if (DOXYGEN_FOUND)
#	configure_file (${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/build/Doxyfile @ONLY)
#	add_custom_target (doc ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/build/Doxyfile 
#	                   WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/build
#	                   COMMENT "Generating API documentation with Doxygen" VERBATIM)
#endif ()

# Allow users to define include and link directories
set(USER_INC_DIR "" CACHE FILEPATH "Set paths to include folders")
set(USER_LIB_DIR "" CACHE FILEPATH "Set paths to link folders")

include_directories(${USER_INC_DIR})
link_directories(${USER_LIB_DIR})

# Add linker and incldue directories for conda build before finding any library
if (DEFINED ENV{CONDA_BUILD})
  link_directories("$ENV{BUILD_PREFIX}/lib")
  include_directories("$ENV{BUILD_PREFIX}/include")
  message(STATUS "Prefix environment variable is $ENV{BUILD_PREFIX}")
endif ()

# Configure MPI
# If using the compiler wrapper, there is no need to find the MPI libraries.
get_filename_component (compiler ${CMAKE_Fortran_COMPILER} NAME)
if (${compiler} STREQUAL mpiifort OR ${compiler} STREQUAL mpifort OR ${compiler} STREQUAL ftn)
	add_definitions (-DMPI_OPT)
else ()
	find_package (MPI REQUIRED)
endif ()

if (MPI_Fortran_FOUND)
	add_definitions (-DMPI_OPT)
	set (CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} ${MPI_Fortran_COMPILE_FLAGS}")
	set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${MPI_Fortran_LINK_FLAGS}")
	include_directories (${MPI_Fortran_INCLUDE_PATH})

#  Some of the tests need to be oversubscribbed to run. There is a problem where
#  depending on the current platform or implementation of MPI the oversubscribe
#  flag changes. MPIEXEC may not write to stdout correctly. Add the ability to
#  set this variable manually.
	if (NOT DEFINED ${MPI_OVERSUBSCRIBE_FLAG}) 
		execute_process(COMMAND ${MPIEXEC} --version OUTPUT_VARIABLE MPI_VERSION)
		if (${MPI_VERSION} MATCHES "Open MPI" OR
		    ${MPI_VERSION} MATCHES "OpenRTE"  OR
		    ${MPI_VERSION} MATCHES "slurm"    OR
		    ${MPI_VERSION} MATCHES "aprun")
			set (MPI_OVERSUBSCRIBE_FLAG "--oversubscribe")
	 	endif ()
	endif ()
endif ()

# Set a directories to build all binary files.
set (EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_BINARY_DIR}/build/bin)
set (LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_BINARY_DIR}/build/lib)

# Add extra compiler flags to various builds.
set (CMAKE_Fortran_FLAGS_RELEASE "${CMAKE_Fortran_FLAGS_RELEASE} ${EXTRA_RELEASE_FLAGS}")
set (CMAKE_Fortran_FLAGS_DEBUG "${CMAKE_Fortran_FLAGS_DEBUG} ${EXTRA_DEBUG_FLAGS}")
set (CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} ${EXTRA_FLAGS}")

# Fortran specific settings. The first setting tells the compiler to use the C preprocessor. The second places a common directory to place all of the module files.

set (CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -cpp")
if(CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_Fortran_COMPILER_VERSION VERSION_GREATER "10.0")
        set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fallow-argument-mismatch")
    endif()
endif()
set (CMAKE_Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/build/modules)

# Set Apple specific options. This adds a predefine for DARWIN and adds lapack from the Accelerate framework.
#if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
#add_definitions (-DDARWIN)
#set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework Accelerate")
#endif ()
if (APPLE)
    add_definitions (-DMACOSX)
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework Accelerate")
endif ()

# Set Linux specific options. This adds a predefine for LINUX.
if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
	add_definitions (-DLINUX)
endif()

find_package (LAPACK)
if(NOT LAPACK_FOUND)
    message(STATUS "--> LAPACK and BLAS were not found. Reference LAPACK and BLAS will be downloaded and installed")
    include(ExternalProject)
    ExternalProject_Add(
        lapack
        URL http://www.netlib.org/lapack/lapack.tgz
        CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${CMAKE_BINARY_DIR}
        PREFIX ${CMAKE_CURRENT_BINARY_DIR}/dependencies
    )
    if (NOT MSVC)
        SET(LAPACK_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/lib/liblapack.a CACHE STRING "LAPACK library" FORCE)
        SET(BLAS_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/lib/libblas.a CACHE STRING "BLAS library" FORCE)
    else () # MSVC
        SET(LAPACK_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/lib/liblapack.lib CACHE STRING "LAPACK library" FORCE)
        SET(BLAS_LIBRARIES ${CMAKE_CURRENT_BINARY_DIR}/lib/libblas.lib CACHE STRING "BLAS library" FORCE)
    endif ()
ENDIF()
#if (NOT ${BLA_VENDOR} MATCHES "^Intel")
#    find_package (LAPACK REQUIRED)
#endif()

# Use netcdf if libraries are found. The values of NETCDF_LIB_PATH and NETCDF_INC_PATH should be set from the command line or Cmake GUI. Convert path from native format to cmake internal format.
message(STATUS "NETCDF_INC_PATH is ${NETCDF_INC_PATH}")
if (NETCDF_INC_PATH)
    file (TO_NATIVE_PATH ${NETCDF_INC_PATH} NETCDF_INC_PATH)
	find_file (NETCDF_I netcdf.inc ${NETCDF_INC_PATH} NO_DEFAULT_PATH)
else ()
    get_directory_property(INC_DIRS INCLUDE_DIRECTORIES)
    message(STATUS "INC_DIRS  are ${INC_DIRS}")
	find_file (NETCDF_I netcdf.inc ${INC_DIRS} NO_DEFAULT_PATH)
endif ()
message(STATUS "NETCDF_I value is ${NETCDF_I}")

if (NETCDF_LIB_PATH)
    file (TO_NATIVE_PATH ${NETCDF_LIB_PATH} NETCDF_LIB_PATH)
    find_library (NETCDF_F NAMES netcdff ${NETCDF_F_LIB_NAME} HINTS ${NETCDF_LIB_PATH} ENV LD_LIBRARY_PATH)
    find_library (NETCDF_C NAMES netcdf ${NETCDF_C_LIB_NAME} HINTS ${NETCDF_LIB_PATH} ENV LD_LIBRARY_PATH)
else()
    get_directory_property(LINK_DIRS LINK_DIRECTORIES)
    find_library (NETCDF_F NAMES netcdff ${NETCDF_F_LIB_NAME} HINTS ${LINK_DIRS} ENV LD_LIBRARY_PATH)
    find_library (NETCDF_C NAMES netcdf ${NETCDF_C_LIB_NAME} HINTS ${LINK_DIRS} ENV LD_LIBRARY_PATH)
endif ()
message(STATUS "NETCDF_F value is ${NETCDF_F}")
message(STATUS "NETCDF_C value is ${NETCDF_C}")
if (NETCDF_C_LIB_PATH)
    find_library (NETCDF_C NAMES netcdf ${NETCDF_C_LIB_NAME} HINTS ${NETCDF_C_LIB_PATH} ENV LD_LIBRARY_PATH)
endif ()

if (NETCDF_I AND (NETCDF_F OR NETCDF_C))
	message (STATUS "Using netcdf")
	add_definitions (-DNETCDF)
	include_directories (${NETCDF_INC_PATH})

	if (NETCDF_F)
		list (APPEND NETCDF_LIBRARIES ${NETCDF_F})
	endif ()
	if (NETCDF_C)
		list (APPEND NETCDF_LIBRARIES ${NETCDF_C})
	endif ()
else ()
	message (FATAL_ERROR "Failed to find the required netcdf libraries. These must be found by setting the NETCDF_INC_PATH and NETCDF_LIB_PATH variables.")
endif ()

set(SCALAPACK_PREFIX "/usr/local" CACHE PATH "scalapack is installed under this directory")
set(BLACS_PREFIX "/usr/local" CACHE PATH "blacs is installed under this directory")
set(SCALAPACK_LIB_DIR "/usr/local/lib" CACHE PATH "Direct path to where scalapack is installed")
set(BLACS_LIB_DIR "/usr/local/lib" CACHE PATH "Direct path to where blacs is installed")
#ENDIF ()
#FIND_PATH(yaml_cpp_INCLUDE_DIRS "yaml-cpp/yaml.h" HINTS "${YAML_CPP_PREFIX}/include")
#    IF ( NOT yaml_cpp_INCLUDE_DIRS )
#        MESSAGE(FATAL_ERROR "Could not find the yaml-cpp headers")
#    ENDIF ()
#FIND_LIBRARY(yaml_cpp_LIBRARIES "yaml-cpp" HINTS "${YAML_CPP_PREFIX}/lib" "${YAML_CPP_PREFIX}/lib" ENV LD_LIBRARY_PATH)
#    IF ( NOT yaml_cpp_LIBRARIES )
#        MESSAGE(FATAL_ERROR "Could not find the yaml-cpp library")
#    ENDIF ()

find_library (SCALAPACK_LIB NAMES ${SCALAPACK_LIB_NAME} scalapack HINTS "${SCALAPACK_LIB_DIR}" "${SCALAPACK_PREFIX}/lib" ENV LD_LIBRARY_PATH)
message(STATUS "scalapack_lib is ${SCALAPACK_LIB}")
if (SCALAPACK_LIB)
# Some builds of SCALAPACK require linking to external BLACS Libraries
    find_library (BLACS_LIB NAMES ${BLACS_LIB_NAME} blacs HINTS "${BLACS_LIB_DIR}" "${BLACS_PREFIX}/lib" ENV LD_LIBRARY_PATH)
	if (BLACS_LIB)
		list (APPEND SCALAPACK_LIB ${BLACS_LIB})
	endif ()
    find_library (BLACSCinit_LIB NAMES ${BLACS_CINIT_NAME} blacsCinit HINTS "${BLACS_LIB_DIR} ${BLACS_PREFIX}/lib" ENV LD_LIBRARY_PATH)
	if (BLACSCinit_LIB)
		list (APPEND SCALAPACK_LIB ${BLACSCinit_LIB})
	endif ()
    find_library (BLACSF77init_LIB NAMES ${BLACS_F77INIT_NAME} blacsF77init HINTS "${BLACS_LIB_DIR} ${BLACS_PREFIX}/lib" ENV LD_LIBRARY_PATH)
	if (BLACSF77init_LIB)
		list (APPEND SCALAPACK_LIB ${BLACSF77init_LIB})
	endif ()
else ()
	#if (USE_PARVMEC) # This option was not used anywhere 
    message (FATAL_ERROR "SCALAPACK library is required. A suitable library was not found.")
	#endif ()
endif ()
message(STATUS "updated scalapack_lib is ${SCALAPACK_LIB}")

add_definitions (-DMRC)



list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

add_subdirectory(Sources)
if (SKBUILD)
    add_subdirectory(python)
endif()
