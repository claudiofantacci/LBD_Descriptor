PROJECT(LBDDescriptor)

cmake_minimum_required(VERSION 3.0)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

if(COMMAND cmake_policy)
    cmake_policy(SET CMP0042 NEW)
endif(COMMAND cmake_policy)

set(CMAKE_CXX_STANDARD 11)

set(BUILD_SHARED_LIBS ON)

set(LineMatchingLib_HEADER  PairwiseLineMatching.hh
                            LineDescriptor.hh
                            EDLineDetector.hh
                            LineStructure.hh)

set(LineMatchingLib_SRCS    PairwiseLineMatching.cpp
                            LineDescriptor.cpp
                            EDLineDetector.cpp)

find_package(OpenCV REQUIRED)
find_package(EIGEN3 REQUIRED)
#find_package(ARPACK REQUIRED)

function(make_global_path invar cstr)
    set(infile ${${invar}})
    if(infile)
        if(EXISTS ${infile})
            if(NOT IS_ABSOLUTE ${infile})
                get_filename_component(_infile_tmp ${CMAKE_BINARY_DIR}/${infile} REALPATH)
                set(${invar}  ${_infile_tmp} CACHE STRING ${cstr} FORCE)
            endif()
        else()
        MESSAGE(STATUS "${infile} does not exist!")
        set(${invar} ${invar}-NOTFOUND CACHE STRING ${cstr} FORCE)
        endif()
    endif()
endfunction(make_global_path)

# Prefer static libraries:
IF(WIN32)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .lib .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
ELSE(WIN32)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
ENDIF(WIN32)

# Find LAPACK and BLAS
# OPENBLAS or ( ( SystemOpenblas or BLAS) and LAPACK)
## prefer local openblas
find_library(OPENBLAS_LIB openblas PATHS external NO_DEFAULT_PATH)
IF (OPENBLAS_LIB)
    make_global_path(OPENBLAS_LIB "OpenBLAS Library")
    set(LAPACK_LIBRARIES ${OPENBLAS_LIB}) #local openblas has lapack build in
ELSE()
    find_library(OPENBLAS_LIB openblas PATH_SUFFIXES openblas-base)
    IF (OPENBLAS_LIB)
        find_library(LAPACK_LIBRARIES lapack)  #only add liblapack
        IF (NOT LAPACK_LIBRARIES)
            message( STATUS "LAPACK_LIBRARIES not found" )
            SET(ABORT_CONFIG TRUE)
        ENDIF()
        set(LAPACK_LIBRARIES ${LAPACK_LIBRARIES} ${OPENBLAS_LIB})
    ELSE()
        message( STATUS "OPENBLAS_LIB not found, trying LAPACK/BLAS..." )
        find_package(LAPACK REQUIRED)  # this will also find and add libblas
    ENDIF()
ENDIF()
IF (OPENBLAS_LIB)
    find_package( Threads REQUIRED )
    set(LAPACK_LIBRARIES ${LAPACK_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
ENDIF()
message( STATUS "LAPACK_LIBRARIES: ${LAPACK_LIBRARIES}" )


# Find libgfortran (static preferred)
IF(NOT WIN32)
    # Query gfortran to get the libgfortran.so path
    FIND_PROGRAM(_GFORTRAN_EXECUTABLE NAMES gfortran gfortran-mp-4.9)
    IF(_GFORTRAN_EXECUTABLE)
        EXECUTE_PROCESS(COMMAND ${_GFORTRAN_EXECUTABLE} -print-file-name=libgfortran.a
            OUTPUT_VARIABLE _libgfortran_path
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
    ENDIF()
    unset(_GFORTRAN_EXECUTABLE CACHE)

    IF(EXISTS ${_libgfortran_path})
        get_filename_component(GFORTRAN_PATH ${_libgfortran_path} PATH)
        find_library(GFORTRAN_LIB gfortran PATHS ${GFORTRAN_PATH})
    ELSE()
        # if libgfortran wasn't found at this point, the installation is probably broken
        # Let's try to find the library nonetheless.
        FIND_LIBRARY(GFORTRAN_LIB gfortran)
    ENDIF()
    IF (NOT GFORTRAN_LIB)
        MESSAGE(STATUS "gfortran is required but could not be found")
        SET(ABORT_CONFIG TRUE)
    ENDIF (NOT GFORTRAN_LIB)

    # also need libquadmath.a
    find_library(QUADMATH_LIB quadmath PATHS ${GFORTRAN_PATH})
    if (QUADMATH_LIB)
        list (APPEND GFORTRAN_LIB ${QUADMATH_LIB})
    else ()
        message (STATUS "quadmath could not be found")
        set (ABORT_CONFIG TRUE)
    endif ()

    IF (APPLE)
        # also need -lgcc_ext.10.5
        find_library(GCC_EXT_LIB gcc_ext.10.5 PATHS ${GFORTRAN_PATH})
        if (GCC_EXT_LIB)
            list (APPEND GFORTRAN_LIB ${GCC_EXT_LIB})
        else ()
            MESSAGE(STATUS "gcc_ext is required on MAC but could not be found")
            SET(ABORT_CONFIG TRUE)
        endif ()
    ENDIF(APPLE)
ENDIF(NOT WIN32)
message(STATUS "GFORTRAN_LIB found: ${GFORTRAN_LIB}")


# ARPACK
find_library (ARPACK_LIB NAMES arpack PATHS external external/ARPACK)

IF (NOT ARPACK_LIB)
    MESSAGE(SEND_ERROR "libarpack is required but could not be found")
    SET(ABORT_CONFIG TRUE)
ENDIF()
make_global_path(ARPACK_LIB "ARPACK Fortran Library")
message(STATUS "ARPACK_LIB found: ${ARPACK_LIB}")

# SUPERLU
find_path(SUPERLU_INC supermatrix.h "external/SuperLU/SRC" "/usr/local/include/superlu" )
IF (NOT SUPERLU_INC)
    MESSAGE(STATUS "SuperLU (supermatrix.h) is required but could not be found")
    SET(ABORT_CONFIG TRUE)
ENDIF()
make_global_path(SUPERLU_INC "SUPERLU Include Path")

find_library(SUPERLU_LIB superlu.a libsuperlu.a
PATHS ${SUPERLU_INC} ${SUPERLU_INC}/../ external /usr/local/lib )
IF (NOT SUPERLU_LIB)
    MESSAGE(STATUS "SuperLU Library is required but could not be found")
    SET(ABORT_CONFIG TRUE)
ENDIF()
make_global_path(SUPERLU_LIB "SUPERLU Library")
include_directories( ${SUPERLU_INC} )
message( STATUS "SUPERLU_LIB found: ${SUPERLU_LIB}" )


# Library
add_library(LineMatchingLib ${LineMatchingLib_SRCS} ${LineMatchingLib_HEADER})

target_include_directories(LineMatchingLib PUBLIC ${EIGEN3_INCLUDE_DIRS})

target_link_libraries(LineMatchingLib ${OpenCV_LIBS} ${SUPERLU_LIB} ${ARPACK_LIB} ${LAPACK_LIBRARIES} ${GFORTRAN_LIB})


# Application
add_executable(TestLineMatchingAlgorithm TestLineMatchingAlgorithm.cpp)

target_include_directories(TestLineMatchingAlgorithm PUBLIC /usr/local/include/arpackpp)

target_link_libraries(TestLineMatchingAlgorithm LineMatchingLib)
