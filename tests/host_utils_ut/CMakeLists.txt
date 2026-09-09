#Copyright (c) Microsoft. All rights reserved.
#Licensed under the MIT license. See LICENSE file in the project root for full license information.

cmake_minimum_required(VERSION 3.12)

compileAsC11()
set(theseTestsName host_utils_ut)

add_executable(${theseTestsName}
    ${theseTestsName}.c
    ../../src/host_utils.c
)

target_compile_definitions(${theseTestsName} PRIVATE -DUSE_CTEST)
target_include_directories(${theseTestsName} PRIVATE
    ${TESTRUNNERSWITCHER_INCLUDES}
    ${CTEST_INCLUDES}
    ${UMOCK_C_INCLUDES}
    ${AZURE_C_SHARED_UTILITY_INCLUDES}
    ${MICROMOCK_INC_FOLDER}
    ${UMOCK_C_INC_FOLDER}
    ${TESTRUNNERSWITCHER_INC_FOLDER}
    ${CTEST_INC_FOLDER}
    ${SAL_INC_FOLDER}
    ${SHARED_UTIL_INC_FOLDER}
    ${SHARED_UTIL_SRC_FOLDER}
)
target_link_libraries(${theseTestsName} PRIVATE umock_c ctest testrunnerswitcher)

add_test(NAME ${theseTestsName} COMMAND $<TARGET_FILE:${theseTestsName}>)
