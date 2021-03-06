cmake_minimum_required(VERSION 3.5.0)
project(voxcraft-sim LANGUAGES CXX CUDA)

set(CMAKE_CXX_STANDARD 17)

include_directories(/usr/local/cuda/include) # for cuda intellisense

include_directories(src)
include_directories(src/VX3)
include_directories(src/Utils)
include_directories(src/VXA)
include_directories(src/old)

file( GLOB VX3_SRC_cu src/VX3/*.cu )
file( GLOB VX3_SRC_cpp src/VX3/*.cpp )
file( GLOB VXA_SRC_cpp src/VXA/*.cpp )
file( GLOB OLD_SRC src/old/*.c* )

add_library(voxelyze.3.0 ${VX3_SRC_cu} ${VX3_SRC_cpp} ${VXA_SRC_cpp} ${OLD_SRC})
add_executable(voxcraft-sim src/Executables/voxcraft-sim.cpp)
add_executable(vx3_node_worker src/Executables/vx3_node_worker.cu)
add_executable(vx3_node_daemon src/Executables/vx3_node_daemon.cu)
add_executable(vx3_test src/Executables/vx3_test.cu)

target_link_libraries(vx3_node_worker PUBLIC voxelyze.3.0)
target_link_libraries(vx3_test PUBLIC voxelyze.3.0)
# for Boost
set(Boost_USE_STATIC_LIBS   ON)
find_package(Boost REQUIRED COMPONENTS filesystem thread chrono program_options)
target_link_libraries(voxelyze.3.0 PUBLIC ${Boost_LIBRARIES})
target_link_libraries(voxcraft-sim PUBLIC ${Boost_LIBRARIES})
target_link_libraries(vx3_node_worker PUBLIC ${Boost_LIBRARIES})
target_link_libraries(vx3_node_daemon PUBLIC ${Boost_LIBRARIES})


# CUDA options
if(NOT DEFINED CUDA_DEBUG)
  set(CUDA_DEBUG "-G or not" ON)
  message("Set CUDA_DEBUG to ON.")
endif(NOT DEFINED CUDA_DEBUG)
set_target_properties(voxelyze.3.0 PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
set_target_properties(vx3_node_worker PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
set_target_properties(vx3_test PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
    #-G means debug info in GPU, so you can step in GPU kernels
    if (CUDA_DEBUG)
      message("Compile with debug information.")
      target_compile_options(voxelyze.3.0     PUBLIC $<$<COMPILE_LANGUAGE:CUDA>:-G -arch=compute_60 -code=compute_60>)
      target_compile_options(vx3_node_worker  PUBLIC $<$<COMPILE_LANGUAGE:CUDA>:-G -arch=compute_60 -code=compute_60>)
      target_compile_options(vx3_test         PUBLIC $<$<COMPILE_LANGUAGE:CUDA>:-G -arch=compute_60 -code=compute_60>)
    else (CUDA_DEBUG)
      target_compile_options(voxelyze.3.0     PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: -arch=compute_60 -code=compute_60>)
      target_compile_options(vx3_node_worker  PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: -arch=compute_60 -code=compute_60>)
      target_compile_options(vx3_test         PUBLIC $<$<COMPILE_LANGUAGE:CUDA>: -arch=compute_60 -code=compute_60>)
    endif(CUDA_DEBUG)
list(APPEND CMAKE_CUDA_FLAGS "-gencode=arch=compute_60,code=sm_60")


#for gtest
################################
# Testing
################################
option(test "Build all tests." OFF) # Makes boolean 'test' available.
if (test)
  if (APPLE)
    add_definitions(-DGTEST_USE_OWN_TR1_TUPLE)
    add_definitions(-D__GLIBCXX__)
  endif (APPLE)

  # This adds another subdirectory, which has 'project(gtest)'.
  add_subdirectory(test/googletest)

  enable_testing()

  # Include the gtest library. gtest_SOURCE_DIR is available due to
  # 'project(gtest)' above.
  include_directories(${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR})

  ##############
  # Unit Tests
  ##############
  set(TEST_DIR test/testcase)
  file(GLOB TEST_FILES ${TEST_DIR}/test_*.c*)
  add_executable(runUnitTests ${TEST_FILES})

  # Standard linking to gtest stuff.
  target_link_libraries(runUnitTests gtest gtest_main)

  # Extra linking for the project.
  target_link_libraries(runUnitTests voxelyze.3.0)

  # This is so you can do 'make test' to see all your tests run, instead of
  # manually running the executable runUnitTests to see those specific tests.
  add_test(NAME that-test-I-made COMMAND runUnitTests)

  # You can also omit NAME and COMMAND. The second argument could be some other
  # test executable.
  add_test(that-other-test-I-made runUnitTests)
  
  set_target_properties(runUnitTests PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

  target_compile_options(runUnitTests         PUBLIC $<$<COMPILE_LANGUAGE:CUDA>:-G -arch=compute_60 -code=compute_60>)
endif()

set(CMAKE_VERBOSE_MAKEFILE OFF) #turn on if we want to debug in compilation.