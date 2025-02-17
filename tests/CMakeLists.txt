# Copyright 2022 Joe Drago. All rights reserved.
# SPDX-License-Identifier: BSD-2-Clause

# With testing enabled, all targets referenced by add_test() can be run
# at once with CMake's ctest command line tool from the build folder.
enable_testing()

add_executable(aviftest aviftest.c)
if(AVIF_LOCAL_LIBGAV1)
    set_target_properties(aviftest PROPERTIES LINKER_LANGUAGE "CXX")
endif()
target_link_libraries(aviftest avif ${AVIF_PLATFORM_LIBRARIES})
add_test(NAME aviftest COMMAND aviftest ${CMAKE_CURRENT_SOURCE_DIR}/data)

add_executable(avifincrtest avifincrtest.c avifincrtest_helpers.c)
if(AVIF_LOCAL_LIBGAV1)
    set_target_properties(avifincrtest PROPERTIES LINKER_LANGUAGE "CXX")
endif()
target_link_libraries(avifincrtest avif ${AVIF_PLATFORM_LIBRARIES})
add_test(NAME avifincrtest COMMAND avifincrtest ${CMAKE_CURRENT_SOURCE_DIR}/data/sofa_grid1x5_420.avif)

add_executable(avifmetadatatest avifmetadatatest.c)
if(AVIF_LOCAL_LIBGAV1)
    set_target_properties(avifmetadatatest PROPERTIES LINKER_LANGUAGE "CXX")
endif()
target_link_libraries(avifmetadatatest avif ${AVIF_PLATFORM_LIBRARIES})
add_test(NAME avifmetadatatest COMMAND avifmetadatatest)

add_executable(avifyuv avifyuv.c)
if(AVIF_LOCAL_LIBGAV1)
    set_target_properties(avifyuv PROPERTIES LINKER_LANGUAGE "CXX")
endif()
target_link_libraries(avifyuv avif ${AVIF_PLATFORM_LIBRARIES})
foreach(AVIFYUV_MODE limited rgb) # Modes drift and premultiply take more than 2 minutes each so they are disabled.
    add_test(NAME avifyuv_${AVIFYUV_MODE} COMMAND avifyuv -m ${AVIFYUV_MODE})
endforeach()

if(AVIF_ENABLE_GTEST)
    enable_language(CXX)
    set(CMAKE_CXX_STANDARD 11)
    if(AVIF_LOCAL_GTEST)
        set(GTEST_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/ext/googletest/googletest/include)
        set(GTEST_LIBRARIES ${CMAKE_SOURCE_DIR}/ext/googletest/build/lib/libgtest${CMAKE_STATIC_LIBRARY_SUFFIX})
        set(GTEST_MAIN_LIBRARIES ${CMAKE_SOURCE_DIR}/ext/googletest/build/lib/libgtest_main${CMAKE_STATIC_LIBRARY_SUFFIX})
        set(GTEST_BOTH_LIBRARIES ${GTEST_LIBRARIES} ${GTEST_MAIN_LIBRARIES})
        if(NOT EXISTS ${GTEST_INCLUDE_DIRS}/gtest/gtest.h)
            message(FATAL_ERROR "googletest(AVIF_LOCAL_GTEST): ${GTEST_INCLUDE_DIRS}/gtest/gtest.h is missing, bailing out")
        elseif(NOT EXISTS ${GTEST_LIBRARIES})
            message(FATAL_ERROR "googletest(AVIF_LOCAL_GTEST): ${GTEST_LIBRARIES} is missing, bailing out")
        elseif(NOT EXISTS ${GTEST_MAIN_LIBRARIES})
            message(FATAL_ERROR "googletest(AVIF_LOCAL_GTEST): ${GTEST_MAIN_LIBRARIES} is missing, bailing out")
        else()
            message(STATUS "Found local ext/googletest")
        endif()
    else()
        find_package(GTest REQUIRED)
    endif()

    add_executable(avifgridapitest avifgridapitest.cc)
    target_link_libraries(avifgridapitest avif ${AVIF_PLATFORM_LIBRARIES} ${GTEST_BOTH_LIBRARIES})
    target_include_directories(avifgridapitest PRIVATE ${GTEST_INCLUDE_DIRS})
    add_test(NAME avifgridapitest COMMAND avifgridapitest)

    add_executable(avify4mtest avify4mtest.cc)
    target_link_libraries(avify4mtest avif avif_apps ${AVIF_PLATFORM_LIBRARIES} ${GTEST_BOTH_LIBRARIES})
    target_include_directories(avify4mtest PRIVATE ${GTEST_INCLUDE_DIRS})
    add_test(NAME avify4mtest COMMAND avify4mtest)
else()
    message(STATUS "Most tests are disabled because AVIF_ENABLE_GTEST is OFF.")
endif()

if(AVIF_ENABLE_COVERAGE)
    add_custom_target(
        avif_coverage
        COMMAND ${CMAKE_COMMAND} -E env "LLVM_PROFILE_FILE=${CMAKE_CURRENT_BINARY_DIR}/aviftest.profraw" $<TARGET_FILE:aviftest>
                ${CMAKE_CURRENT_SOURCE_DIR}/data
        COMMAND ${XCRUN} llvm-profdata merge -sparse ${CMAKE_CURRENT_BINARY_DIR}/aviftest.profraw -o
                ${CMAKE_CURRENT_BINARY_DIR}/aviftest.profdata
        COMMAND cmake -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/coverage
        COMMAND ${XCRUN} llvm-cov show $<TARGET_FILE:aviftest> -instr-profile=${CMAKE_CURRENT_BINARY_DIR}/aviftest.profdata
                -project-title=libavif --format html -output-dir=${CMAKE_CURRENT_BINARY_DIR}/coverage
        COMMAND echo Coverage report here: ${CMAKE_CURRENT_BINARY_DIR}/coverage/index.html
        DEPENDS aviftest
    )
endif()
