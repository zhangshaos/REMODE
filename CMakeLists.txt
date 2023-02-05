# This file is part of REMODE - REgularized MOnocular Depth Estimation.
#
# Copyright (C) 2014 Matia Pizzoli <matia dot pizzoli at gmail dot com>
# Robotics and Perception Group, University of Zurich, Switzerland
# http://rpg.ifi.uzh.ch
#
# REMODE is free software: you can redistribute it and/or modify it under the
# terms of the GNU General Public License as published by the Free Software
# Foundation, either version 3 of the License, or any later version.
#
# REMODE is distributed in the hope that it will be useful, but WITHOUT ANY
# WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS
# FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  if not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.18)
if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
  set(CMAKE_CUDA_ARCHITECTURES 86) #根据自己的GPU版本设置
endif()
project(REMODE LANGUAGES CXX CUDA)

set(CMAKE_BUILD_TYPE Debug) # Release, RelWithDebInfo
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/CMakeModules/")
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)

# NCC patch size (must be odd integer, typical values: 3, 5, 7)
add_definitions(-DRMD_CORR_PATCH_SIDE=5)
# Maximum extent of epipolar line search in pixels
add_definitions(-DRMD_MAX_EXTENT_EPIPOLAR_SEARCH=100)
add_definitions(-D_USE_MATH_DEFINES)

#############################
#     find dependents       #
#############################
find_package(CUDA  REQUIRED) #use cuda_runtime.h
find_package(OpenCV REQUIRED)
find_package(Eigen REQUIRED)
find_package(Boost REQUIRED COMPONENTS system filesystem)

include_directories(
  include
  ${CUDA_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
  ${Eigen_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS})

# set link libraries
list(APPEND LINK_LIBS
     ${OpenCV_LIBS}
     ${Boost_LIBRARIES})

#################################
#      Crate main library       #
#################################

# set CUDA sourcefiles
list(APPEND CU_SOURCE_FILES
     src/check_cuda_device.cu
     src/seed_matrix.cu
     src/depthmap_denoiser.cu
     src/reduction.cu)

# set sourcefiles
list(APPEND SOURCE_FILES
     src/depthmap.cpp)

add_library(REMODE_CUDA STATIC ${CU_SOURCE_FILES})
#target_compile_options(REMODE_CUDA PUBLIC
#        "$<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CXX_COMPILER_ID:MSVC>>:/MP;/bigobj;/utf-8>")
#set_target_properties(REMODE_CUDA PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

add_library(REMODE STATIC ${SOURCE_FILES})
#target_compile_options(REMODE PUBLIC
#        "$<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CXX_COMPILER_ID:MSVC>>:/MP;/bigobj;/utf-8>")
#set_target_properties(REMODE PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
target_link_libraries(REMODE PUBLIC ${LINK_LIBS})

################################
#       Test on dataset        #
################################
add_executable(dataset_main
        test/dataset_main.cpp
        test/dataset.cpp)
#target_compile_options(dataset_main PUBLIC
#        "$<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CXX_COMPILER_ID:MSVC>>:/MP;/bigobj;/utf-8>")
#set_target_properties(dataset_main PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
target_link_libraries(dataset_main PRIVATE
                      REMODE
                      REMODE_CUDA)

##################
#    单元测试     #
#################
option(BUILD_TESTS "Build test programs" OFF)

if(BUILD_TESTS)
  add_definitions(-DRMD_BUILD_TESTS)
  enable_testing()
  find_package(GTest CONFIG REQUIRED)

  add_library(REMODE_test STATIC
              test/copy.cu
              test/sobel.cu)
#  target_compile_options(REMODE_test PUBLIC
#          "$<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CXX_COMPILER_ID:MSVC>>:/MP;/bigobj;/utf-8>")
#  set_target_properties(REMODE_test PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

  # set source files for the tests
  list(APPEND TEST_SOURCE_FILES
       test/device_image_test.cpp
       test/seed_matrix_test.cpp
       test/main_test.cpp
       test/dataset.cpp
       test/epipolar_test.cpp
       test/reduction_test.cpp)

  add_executable(all_tests ${TEST_SOURCE_FILES})
#  target_compile_options(all_test PUBLIC
#          "$<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CXX_COMPILER_ID:MSVC>>:/MP;/bigobj;/utf-8>")
#  set_target_properties(all_test PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
  target_link_libraries(all_tests PRIVATE
                        REMODE
                        REMODE_CUDA
                        REMODE_test
                        GTest::gmock
                        GTest::gtest
                        GTest::gmock_main
                        GTest::gtest_main)

  # Allow run of 'make test'
  add_test(
    RMDCuTests
    all_tests --gtest_repeat=2)
endif(BUILD_TESTS)
