# --------------------------------------------------------------------------
#  CppADCodeGen: C++ Algorithmic Differentiation with Source Code Generation:
#    Copyright (C) 2018 Joao Leal
#    Copyright (C) 2012 Ciengis
#
#  CppADCodeGen is distributed under multiple licenses:
#
#   - Eclipse Public License Version 1.0 (EPL1), and
#   - GNU General Public License Version 3 (GPL3).
#
#  EPL1 terms and conditions can be found in the file "epl-v10.txt", while
#  terms and conditions for the GPL3 can be found in the file "gpl3.txt".
# ----------------------------------------------------------------------------
#
# Author: Joao Leal
#
# ----------------------------------------------------------------------------
SET(CMAKE_BUILD_TYPE DEBUG)

################################################################################
# download and compile GTest
################################################################################
find_package(GTest)
if(NOT GTEST_FOUND)
    INCLUDE(ExternalProject)

    # Download from GitHub
    INCLUDE(FetchContent)
    FetchContent_Declare(
        googletest
        URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip)

    # For Windows: Prevent overriding the parent project's compiler/linker settings
    SET(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    FetchContent_MakeAvailable(googletest)

    INCLUDE(GoogleTest)
endif()

################################################################################
# add gtest dependency
################################################################################
FUNCTION(add_gest_link_libraries test_target_name)
    IF(CMAKE_COMPILER_IS_GNUCXX OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        TARGET_LINK_LIBRARIES(${test_target_name} gtest)

        TARGET_LINK_LIBRARIES(${test_target_name} GTest::gtest_main)

        TARGET_LINK_LIBRARIES(${test_target_name} pthread)
    ELSE()
        TARGET_LINK_LIBRARIES(${test_target_name} GTest::gtest_main)

  ENDIF()
ENDFUNCTION()

################################################################################
# create a temporary folder for a set of tests
################################################################################
FUNCTION(create_temp_folder_target)
    IF(NOT TARGET "create_tmp_folder_${folder_name}")
        GET_FILENAME_COMPONENT(folder_name ${CMAKE_CURRENT_SOURCE_DIR} NAME)

        ADD_CUSTOM_COMMAND(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/tmp"
                           COMMAND ${CMAKE_COMMAND} -E remove_directory "tmp" # make sure tmp is empty
                           COMMAND ${CMAKE_COMMAND} -E make_directory   "tmp"
                           COMMENT "Creating tmp folder")
        ADD_CUSTOM_TARGET("create_tmp_folder_${folder_name}" ALL
                          DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/tmp")
    ENDIF()
ENDFUNCTION()

################################################################################
# add test function
################################################################################
ADD_CUSTOM_TARGET(build_tests)
FUNCTION(add_cppadcg_test source_file)

    GET_FILENAME_COMPONENT(test_target_name "${source_file}" NAME_WE)

    ADD_EXECUTABLE("${test_target_name}"
                   # sources:
                   "${source_file}"
                   ${ARGN})

    GET_FILENAME_COMPONENT(folder_name ${source_file} ABSOLUTE)
    GET_FILENAME_COMPONENT(folder_name ${folder_name} PATH)
    GET_FILENAME_COMPONENT(folder_name ${folder_name} NAME)

    create_temp_folder_target()
    ADD_DEPENDENCIES("${test_target_name}" "create_tmp_folder_${folder_name}")

    # Specify MainTest's link libraries
    add_gest_link_libraries(${test_target_name} ON)

    IF( UNIX )
        TARGET_LINK_LIBRARIES(${test_target_name} ${DL_LIBRARIES})
    ENDIF()
    TARGET_LINK_LIBRARIES(${test_target_name} ${CPPAD_LIBRARIES})
    TARGET_INCLUDE_DIRECTORIES(${test_target_name} PRIVATE ${CPPAD_INCLUDE_DIRS})
    TARGET_COMPILE_DEFINITIONS(${test_target_name}
                               PRIVATE CPPAD_CG_C_COMPILER=\"${CMAKE_C_COMPILER}\")

    ADD_TEST(NAME ${test_target_name}
            COMMAND ${test_target_name})

    IF($ENABLE_TEST_CPPCHECKS)
        add_test_cppcheck(${test_target_name} STYLE MISSING_INCLUDE)
    ENDIF()

    IF(Valgrind_FOUND)
        ADD_TEST(NAME ${test_target_name}_memcheck
                 COMMAND ${GTEST_MEMORYCHECK_COMMAND}
                         --read-var-info=yes
                         --leak-check=full
                         --track-origins=yes
                         --leak-resolution=high
                         --vgdb=no
                         --quiet
                         --error-exitcode=1
                         --log-file=${CMAKE_CURRENT_BINARY_DIR}/valgrind.log.${test_target_name}
                         --num-callers=20
                         ./${test_target_name}
                 COMMENT "Running valgrind on test \"${test_target_name}\"")
    ENDIF()

    ################################################################################
    # precompile headers
    ################################################################################
    IF(USE_PRECOMPILED_HEADERS)
        depends_on_precompile_header(${test_target_name} "CppADCGTest.hpp")
    ENDIF()

    ADD_DEPENDENCIES(build_tests ${test_target_name})

ENDFUNCTION()
################################################################################
# use valgrind if possible
################################################################################
OPTION(USE_Valgrind "Use valgrind in tests" ON)

IF(${USE_Valgrind})
    FIND_PACKAGE(Valgrind)

    IF(${Valgrind_FOUND})
        SET(GTEST_MEMORYCHECK_COMMAND "${Valgrind_EXECUTABLE}")
        MESSAGE(STATUS "Valgrind found")
    ELSE()
        MESSAGE(WARNING "Valgrind not found")
    ENDIF()
ELSE()
    IF(CMAKE_COMPILER_IS_GNUCXX)
        OPTION(USE_SANITIZE "Use gcc compiler sanitize flags in tests" OFF)

        IF(${USE_SANITIZE})
            SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address -fno-omit-frame-pointer")
            ADD_DEFINITIONS("-fsanitize=address")
            ADD_DEFINITIONS("-DCPPADCG_TEST_SANITIZE")
        ENDIF()
    ENDIF()
ENDIF()

################################################################################
# add tests
################################################################################

INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})

################################################################################

INCLUDE(${CMAKE_SOURCE_DIR}/cmake/PrecompiledHeaders.cmake)

OPTION(USE_PRECOMPILED_HEADERS "Use precompiled headers in tests" OFF)

################################################################################

IF(USE_PRECOMPILED_HEADERS)
    precompile_header(CppADCGTest.hpp
                      cppad/cg/cppadcg.hpp)
ENDIF()

ADD_SUBDIRECTORY(cppad/cg)

FILE(COPY "${CMAKE_CURRENT_SOURCE_DIR}/valgrind.sh"
     DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
