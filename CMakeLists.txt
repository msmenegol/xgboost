cmake_minimum_required (VERSION 3.2)
project(xgboost)
include(cmake/Utils.cmake)
find_package(OpenMP)

set_default_configuration_release()
msvc_use_static_runtime()

# Options
option(PLUGIN_UPDATER_GPU "Build GPU accelerated tree construction plugin")
option(JVM_BINDINGS "Build JVM bindings" OFF)
option(GOOGLE_TEST "Build google tests" OFF)
set(GPU_COMPUTE_VER 35;50;52;60;61 CACHE STRING
  "Space separated list of compute versions to be built against")


# Compiler flags
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
if(MSVC)
  # Multithreaded compilation
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
else()
  # Correct error for GCC 5 and cuda
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_MWAITXINTRIN_H_INCLUDED -D_FORCE_INLINES")
  # Performance
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -funroll-loops")
endif()

include_directories (
    ${PROJECT_SOURCE_DIR}/include
    ${PROJECT_SOURCE_DIR}/dmlc-core/include
    ${PROJECT_SOURCE_DIR}/rabit/include
)

file(GLOB_RECURSE SOURCES 
    src/*.cc
    src/*.h
)
# Only add main function for executable target
list(REMOVE_ITEM SOURCES ${PROJECT_SOURCE_DIR}/src/cli_main.cc)

# TODO: Create rabit cmakelists.txt
set(RABIT_SOURCES
    rabit/src/allreduce_base.cc
    rabit/src/allreduce_robust.cc
    rabit/src/engine.cc
    rabit/src/c_api.cc
)

file(GLOB_RECURSE CUDA_SOURCES
    plugin/updater_gpu/src/*.cu
    plugin/updater_gpu/src/*.cuh
)

add_subdirectory(dmlc-core)
set(LINK_LIBRARIES dmlccore rabit)

# GPU Plugin
if(PLUGIN_UPDATER_GPU)
  find_package(CUDA 7.5 REQUIRED)
  cmake_minimum_required(VERSION 3.5)
  add_subdirectory(nccl)
  
  include_directories(
    nccl/src
    cub
  )

  set(GENCODE_FLAGS "")
  format_gencode_flags("${GPU_COMPUTE_VER}" GENCODE_FLAGS)
  set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};--expt-extended-lambda;${GENCODE_FLAGS};-lineinfo;")
  if(NOT MSVC)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-Xcompiler -fPIC; -std=c++11")
  endif()

  cuda_add_library(gpuxgboost ${CUDA_SOURCES} STATIC)
  target_link_libraries(gpuxgboost nccl)
  list(APPEND LINK_LIBRARIES gpuxgboost) 
  list(APPEND SOURCES plugin/updater_gpu/src/register_updater_gpu.cc) 
endif()

add_library(rabit STATIC ${RABIT_SOURCES})

add_library(objxgboost OBJECT ${SOURCES})

# Executable
add_executable(runxgboost $<TARGET_OBJECTS:objxgboost> src/cli_main.cc)
set_target_properties(runxgboost PROPERTIES 
  OUTPUT_NAME xgboost 
)
set_output_directory(runxgboost ${PROJECT_SOURCE_DIR})
target_link_libraries(runxgboost ${LINK_LIBRARIES})

# Shared library
add_library(xgboost SHARED $<TARGET_OBJECTS:objxgboost>)
target_link_libraries(xgboost ${LINK_LIBRARIES})
set_output_directory(xgboost ${PROJECT_SOURCE_DIR}/lib)

#Ensure these two targets do not build simultaneously, as they produce outputs with conflicting names
add_dependencies(xgboost runxgboost)

# JVM
if(JVM_BINDINGS)
    find_package(JNI QUIET REQUIRED)

    include_directories(${JNI_INCLUDE_DIRS} jvm-packages/xgboost4j/src/native)

    add_library(xgboost4j SHARED
        $<TARGET_OBJECTS:objxgboost>
        jvm-packages/xgboost4j/src/native/xgboost4j.cpp)
    set_output_directory(xgboost4j ${PROJECT_SOURCE_DIR}/lib)
    target_link_libraries(xgboost4j
        ${LINK_LIBRARIES}
        ${JAVA_JVM_LIBRARY})
endif()

# Test
if(GOOGLE_TEST)
  enable_testing()
  find_package(GTest REQUIRED)

  file(GLOB_RECURSE TEST_SOURCES "tests/cpp/*.cc")
  auto_source_group("${TEST_SOURCES}")
  include_directories(${GTEST_INCLUDE_DIRS})

  if(PLUGIN_UPDATER_GPU)
    file(GLOB_RECURSE CUDA_TEST_SOURCES "plugin/updater_gpu/test/cpp/*.cu")
    set(CUDA_VERBOSE_BUILD ON)
    cuda_compile(CUDA_TEST_OBJS ${CUDA_TEST_SOURCES})
  else()
    set(CUDA_TEST_OBJS "")
  endif()

  add_executable(testxgboost ${TEST_SOURCES} ${CUDA_TEST_OBJS} $<TARGET_OBJECTS:objxgboost>)
  set_output_directory(testxgboost ${PROJECT_SOURCE_DIR})
  target_link_libraries(testxgboost ${GTEST_BOTH_LIBRARIES} ${LINK_LIBRARIES})

  add_test(TestXGBoost testxgboost)
endif()

# Group sources
auto_source_group("${SOURCES}")
