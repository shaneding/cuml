#=============================================================================
# Copyright (c) 2018-2021, NVIDIA CORPORATION.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#=============================================================================

cmake_minimum_required(VERSION 3.20.1 FATAL_ERROR)
file(DOWNLOAD https://raw.githubusercontent.com/rapidsai/rapids-cmake/branch-21.10/RAPIDS.cmake
    ${CMAKE_BINARY_DIR}/RAPIDS.cmake)
include(${CMAKE_BINARY_DIR}/RAPIDS.cmake)
include(rapids-cmake)
include(rapids-cpm)
include(rapids-cuda)
include(rapids-export)
include(rapids-find)

rapids_cuda_init_architectures(CUML)

project(CUML VERSION 21.10.00 LANGUAGES CXX CUDA)

# Write the version header
rapids_cmake_write_version_file(include/cuml/version_config.hpp)

##############################################################################
# - build type ---------------------------------------------------------------

# Set a default build type if none was specified
rapids_cmake_build_type(Release)

# this is needed for clang-tidy runs
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

##############################################################################
# - User Options  ------------------------------------------------------------

option(BUILD_CUML_C_LIBRARY "Build libcuml_c shared library. Contains the cuML C API" ON)
option(BUILD_CUML_CPP_LIBRARY "Build libcuml shared library" ON)
option(BUILD_CUML_TESTS "Build cuML algorithm tests" ON)
option(BUILD_CUML_MG_TESTS "Build cuML multigpu algorithm tests" OFF)
option(BUILD_PRIMS_TESTS "Build ml-prim tests" ON)
option(BUILD_CUML_EXAMPLES "Build C++ API usage examples" ON)
option(BUILD_CUML_BENCH "Build cuML C++ benchmark tests" ON)
option(BUILD_CUML_PRIMS_BENCH "Build ml-prims C++ benchmark tests" ON)
option(BUILD_CUML_STD_COMMS "Build the standard NCCL+UCX Communicator" ON)
option(BUILD_CUML_MPI_COMMS "Build the MPI+NCCL Communicator (used for testing)" OFF)
option(CUDA_ENABLE_KERNEL_INFO "Enable kernel resource usage info" OFF)
option(CUDA_ENABLE_LINE_INFO "Enable lineinfo in nvcc" OFF)
option(DETECT_CONDA_ENV "Enable detection of conda environment for dependencies" ON)
option(DISABLE_DEPRECATION_WARNINGS "Disable depreaction warnings " ON)
option(DISABLE_OPENMP "Disable OpenMP" OFF)
option(ENABLE_CUMLPRIMS_MG "Enable algorithms that use libcumlprims_mg" ON)
option(NVTX "Enable nvtx markers" OFF)
option(SINGLEGPU "Disable all mnmg components and comms libraries" OFF)
option(USE_CCACHE "Cache build artifacts with ccache" OFF)

message(VERBOSE "CUML: Building libcuml_c shared library. Contains the cuML C API: ${BUILD_CUML_C_LIBRARY}")
message(VERBOSE "CUML: Building libcuml shared library: ${BUILD_CUML_CPP_LIBRARY}")
message(VERBOSE "CUML: Building cuML algorithm tests: ${BUILD_CUML_TESTS}")
message(VERBOSE "CUML: Building cuML multigpu algorithm tests: ${BUILD_CUML_MG_TESTS}")
message(VERBOSE "CUML: Building ml-prims tests: ${BUILD_PRIMS_TESTS}")
message(VERBOSE "CUML: Building C++ API usage examples: ${BUILD_CUML_EXAMPLES}")
message(VERBOSE "CUML: Building cuML C++ benchmark tests: ${BUILD_CUML_BENCH}")
message(VERBOSE "CUML: Building ml-prims C++ benchmark tests: ${BUILD_CUML_PRIMS_BENCH}")
message(VERBOSE "CUML: Building the standard NCCL+UCX Communicator: ${BUILD_CUML_STD_COMMS}")
message(VERBOSE "CUML: Building the MPI+NCCL Communicator (used for testing): ${BUILD_CUML_MPI_COMMS}")
message(VERBOSE "CUML: Enabling detection of conda environment for dependencies: ${DETECT_CONDA_ENV}")
message(VERBOSE "CUML: Disabling OpenMP: ${DISABLE_OPENMP}")
message(VERBOSE "CUML: Enabling algorithms that use libcumlprims_mg: ${ENABLE_CUMLPRIMS_MG}")
message(VERBOSE "CUML: Enabling kernel resource usage info: ${KERNEL_INFO}")
message(VERBOSE "CUML: Enabling kernelinfo in nvcc: ${CUDA_ENABLE_KERNEL_INFO}")
message(VERBOSE "CUML: Enabling lineinfo in nvcc: ${CUDA_ENABLE_LINE_INFO}")
message(VERBOSE "CUML: Enabling nvtx markers: ${NVTX}")
message(VERBOSE "CUML: Disabling all mnmg components and comms libraries: ${SINGLEGPU}")
message(VERBOSE "CUML: Cache build artifacts with ccache: ${USE_CCACHE}")

# Set RMM logging level
set(RMM_LOGGING_LEVEL "INFO" CACHE STRING "Choose the logging level.")
set_property(CACHE RMM_LOGGING_LEVEL PROPERTY STRINGS "TRACE" "DEBUG" "INFO" "WARN" "ERROR" "CRITICAL" "OFF")
message(VERBOSE "CUML: RMM_LOGGING_LEVEL = '${RMM_LOGGING_LEVEL}'.")

##############################################################################
# - Target names -------------------------------------------------------------

set(CUML_CPP_TARGET "cuml++")
set(CUML_CPP_TEST_TARGET "ml")
set(CUML_CPP_BENCH_TARGET "sg_benchmark")
set(CUML_C_TARGET "cuml")
set(CUML_C_TEST_TARGET "${CUML_C_TARGET}_test")
set(CUML_MG_TEST_TARGET "ml_mg")
set(PRIMS_TEST_TARGET "prims")
set(PRIMS_BENCH_TARGET "prims_benchmark")

##############################################################################
# - Conda environment detection ----------------------------------------------

if(DETECT_CONDA_ENV)
  rapids_cmake_support_conda_env( conda_env MODIFY_PREFIX_PATH )
  if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT AND DEFINED ENV{CONDA_PREFIX})
      message(STATUS "CUML: No CMAKE_INSTALL_PREFIX argument detected, setting to: $ENV{CONDA_PREFIX}")
      set(CMAKE_INSTALL_PREFIX "$ENV{CONDA_PREFIX}")
  endif()
endif()

##############################################################################
# - compiler options ---------------------------------------------------------

if (NOT DISABLE_OPENMP)
  find_package(OpenMP)
  if(OpenMP_FOUND)
    message(STATUS "CUML: OpenMP found in ${OPENMP_INCLUDE_DIRS}")
  endif()
endif()

# * find CUDAToolkit package
# * determine GPU architectures
# * enable the CMake CUDA language
# * set other CUDA compilation flags
rapids_find_package(CUDAToolkit REQUIRED
    BUILD_EXPORT_SET cuml-exports
    INSTALL_EXPORT_SET cuml-exports
    )
include(cmake/modules/ConfigureCUDA.cmake)

##############################################################################
# - Set options based on user defined one  -----------------------------------

# Enabling libcuml enables building libcuml++
if(BUILD_CUML_C_LIBRARY)
  set(BUILD_CUML_CPP_LIBRARY ON)
endif()

# Disabling libcuml++ disables building algorithm tests and examples
if(NOT BUILD_CUML_CPP_LIBRARY)
  set(BUILD_CUML_C_LIBRARY OFF)
  set(BUILD_CUML_TESTS OFF)
  set(BUILD_CUML_MG_TESTS OFF)
  set(BUILD_CUML_EXAMPLES OFF)
endif()

# SingleGPU build disables cumlprims_mg and comms components
if(SINGLEGPU)
  message(STATUS "Detected SINGLEGPU build option")
  message(STATUS "Disabling Multi-GPU components and comms libraries")
  set(BUILD_CUML_MG_TESTS OFF)
  set(BUILD_CUML_STD_COMMS OFF)
  set(BUILD_CUML_MPI_COMMS OFF)
  set(ENABLE_CUMLPRIMS_MG OFF)
  set(WITH_UCX OFF)
endif()

if(BUILD_CUML_MG_TESTS AND NOT SINGLEGPU)
  message(STATUS "Detected BUILD_CUML_MG_TESTS set to ON. Enabling BUILD_CUML_MPI_COMMS")
  set(BUILD_CUML_MPI_COMMS ON)
endif()

if(USE_CCACHE)
  set(CMAKE_C_COMPILER_LAUNCHER ccache)
  set(CMAKE_CXX_COMPILER_LAUNCHER ccache)
  set(CMAKE_CUDA_COMPILER_LAUNCHER ccache)
endif()

##############################################################################
# - Requirements -------------------------------------------------------------

# add third party dependencies using CPM
rapids_cpm_init()

include(cmake/thirdparty/get_thrust.cmake)
include(cmake/thirdparty/get_rmm.cmake)
include(cmake/thirdparty/get_faiss.cmake)
include(cmake/thirdparty/get_treelite.cmake)
include(cmake/thirdparty/get_raft.cmake)

if(NOT SINGLEGPU)
  include(cmake/thirdparty/get_nccl.cmake)
  include(cmake/thirdparty/get_ucx.cmake)
endif()

if(ENABLE_CUMLPRIMS_MG)
  include(cmake/thirdparty/get_cumlprims_mg.cmake)
endif()

if(BUILD_CUML_TESTS OR BUILD_PRIMS_TESTS)
  include(cmake/thirdparty/get_gtest.cmake)
endif()

if(BUILD_CUML_BENCH)
  include(cmake/thirdparty/get_gbench.cmake)
endif()

##############################################################################
# - build libcuml++ shared library -------------------------------------------

if(BUILD_CUML_C_LIBRARY OR BUILD_CUML_CPP_LIBRARY)
  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/fatbin.ld"
[=[
SECTIONS
{
.nvFatBinSegment : { *(.nvFatBinSegment) }
.nv_fatbin : { *(.nv_fatbin) }
}
]=])
endif()

if(BUILD_CUML_CPP_LIBRARY)

  # single GPU components
  add_library(${CUML_CPP_TARGET} SHARED
    src/arima/batched_arima.cu
    src/arima/batched_kalman.cu
    src/common/logger.cpp
    src/common/nvtx.cu
    src/datasets/make_arima.cu
    src/datasets/make_blobs.cu
    src/datasets/make_regression.cu
    src/dbscan/dbscan.cu
    src/decisiontree/decisiontree.cu
    src/explainer/kernel_shap.cu
    src/explainer/permutation_shap.cu
    src/fil/fil.cu
    src/fil/infer.cu
    src/glm/glm.cu
    src/genetic/genetic.cu
    src/genetic/node.cu
    src/hdbscan/hdbscan.cu
    src/hdbscan/condensed_hierarchy.cu
    src/holtwinters/holtwinters.cu
    src/kmeans/kmeans.cu
    src/knn/knn.cu
    src/knn/knn_sparse.cu
    src/hierarchy/linkage.cu
    src/metrics/accuracy_score.cu
    src/metrics/adjusted_rand_index.cu
    src/metrics/completeness_score.cu
    src/metrics/entropy.cu
    src/metrics/homogeneity_score.cu
    src/metrics/kl_divergence.cu
    src/metrics/mutual_info_score.cu
    src/metrics/pairwise_distance.cu
    src/metrics/pairwise_distance_canberra.cu
    src/metrics/pairwise_distance_chebyshev.cu
    src/metrics/pairwise_distance_cosine.cu
    src/metrics/pairwise_distance_euclidean.cu
    src/metrics/pairwise_distance_hellinger.cu
    src/metrics/pairwise_distance_l1.cu
    src/metrics/pairwise_distance_minkowski.cu
    src/metrics/r2_score.cu
    src/metrics/rand_index.cu
    src/metrics/silhouette_score.cu
    src/metrics/trustworthiness.cu
    src/metrics/v_measure.cu
    src/pca/pca.cu
    src/randomforest/randomforest.cu
    src/random_projection/rproj.cu
    src/solver/lars.cu
    src/solver/solver.cu
    src/spectral/spectral.cu
    src/svm/svc.cu
    src/svm/svr.cu
    src/svm/ws_util.cu
    src/tsa/auto_arima.cu
    src/tsa/stationarity.cu
    src/tsne/tsne.cu
    src/tsvd/tsvd.cu
    src/umap/umap.cu)

  # multi GPU components
  # todo: separate mnmg that require cumlprims from those that don't
  if(NOT SINGLEGPU)
    target_sources(${CUML_CPP_TARGET}
      PRIVATE
        src/glm/ols_mg.cu
        src/glm/preprocess_mg.cu
        src/glm/ridge_mg.cu
        src/kmeans/kmeans_mg.cu
        src/knn/knn_mg.cu
        src/knn/knn_classify_mg.cu
        src/knn/knn_regress_mg.cu
        src/pca/pca_mg.cu
        src/pca/sign_flip_mg.cu
        src/solver/cd_mg.cu
        src/tsvd/tsvd_mg.cu
    )
  endif()

  add_library(cuml::${CUML_CPP_TARGET} ALIAS ${CUML_CPP_TARGET})

  set_target_properties(${CUML_CPP_TARGET}
    PROPERTIES BUILD_RPATH                         "\$ORIGIN"
               INSTALL_RPATH                       "\$ORIGIN"
               # set target compile options
               CXX_STANDARD                        17
               CXX_STANDARD_REQUIRED               ON
               CUDA_STANDARD                       17
               CUDA_STANDARD_REQUIRED              ON
               INTERFACE_POSITION_INDEPENDENT_CODE ON
  )

  target_compile_definitions(${CUML_CPP_TARGET}
    PUBLIC
      $<$<BOOL:${NVTX}>:NVTX_ENABLED>
      DISABLE_CUSPARSE_DEPRECATED
    PRIVATE
      CUML_CPP_API
  )

  target_compile_options(${CUML_CPP_TARGET}
        PRIVATE "$<$<COMPILE_LANGUAGE:CXX>:${CUML_CXX_FLAGS}>"
                "$<$<COMPILE_LANGUAGE:CUDA>:${CUML_CUDA_FLAGS}>"
  )

  target_include_directories(${CUML_CPP_TARGET}
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
      $<BUILD_INTERFACE:$<$<BOOL:${ENABLE_CUMLPRIMS_MG}>:${cumlprims_mg_INCLUDE_DIRS}>>
    PRIVATE
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/metrics>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src_prims>
      $<$<OR:$<BOOL:${BUILD_CUML_STD_COMMS}>,$<BOOL:${BUILD_CUML_MPI_COMMS}>>:${NCCL_INCLUDE_DIRS}>
      $<$<BOOL:${BUILD_CUML_MPI_COMMS}>:${MPI_CXX_INCLUDE_PATH}>
    INTERFACE
      $<INSTALL_INTERFACE:include>
  )

  target_link_libraries(${CUML_CPP_TARGET}
    PUBLIC
      rmm::rmm
      cuml::Thrust
      raft::raft
    PRIVATE
      CUDA::cublas
      CUDA::cufft
      CUDA::curand
      CUDA::cusolver
      CUDA::cudart
      CUDA::cusparse
      $<$<BOOL:${NVTX}>:CUDA::nvToolsExt>
      FAISS::FAISS
      $<IF:$<BOOL:${Treelite_ADDED}>,treelite::treelite_static,treelite::treelite>
      $<IF:$<BOOL:${Treelite_ADDED}>,treelite::treelite_runtime_static,treelite::treelite_runtime>
      $<$<BOOL:${OpenMP_FOUND}>:OpenMP::OpenMP_CXX>
      $<$<OR:$<BOOL:${BUILD_CUML_STD_COMMS}>,$<BOOL:${BUILD_CUML_MPI_COMMS}>>:NCCL::NCCL>
      $<$<BOOL:${BUILD_CUML_MPI_COMMS}>:${MPI_CXX_LIBRARIES}>
      $<$<BOOL:${ENABLE_CUMLPRIMS_MG}>:cumlprims_mg::cumlprims_mg>
  )

  # If we export the libdmlc symbols, they can lead to weird crashes with other
  # libraries that use libdmlc. This just hides the symbols internally.
  target_link_options(${CUML_CPP_TARGET} PRIVATE "-Wl,--exclude-libs,libdmlc.a")
  # same as above, but for protobuf library
  target_link_options(${CUML_CPP_TARGET} PRIVATE "-Wl,--exclude-libs,libprotobuf.a")
  # ensure CUDA symbols aren't relocated to the middle of the debug build binaries
  target_link_options(${CUML_CPP_TARGET} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/fatbin.ld")

endif()

#############################################################################
# - build libcuml C shared library -------------------------------------------

if(BUILD_CUML_C_LIBRARY)
  add_library(${CUML_C_TARGET} SHARED
    src/common/cumlHandle.cpp
    src/common/cuml_api.cpp
    src/dbscan/dbscan_api.cpp
    src/glm/glm_api.cpp
    src/holtwinters/holtwinters_api.cpp
    src/knn/knn_api.cpp
    src/svm/svm_api.cpp
  )

  add_library(cuml::${CUML_C_TARGET} ALIAS ${CUML_C_TARGET})

  target_compile_definitions(${CUML_C_TARGET}
    PRIVATE
      CUML_C_API)

  target_include_directories(${CUML_C_TARGET}
    PRIVATE
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
  )

  target_link_libraries(${CUML_C_TARGET}
    PUBLIC
      ${CUML_CPP_TARGET}
    PRIVATE
      FAISS::FAISS
  )

  # ensure CUDA symbols aren't relocated to the middle of the debug build binaries
  target_link_options(${CUML_C_TARGET} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/fatbin.ld")

endif()

##############################################################################
# - build test executables ---------------------------------------------------

if(BUILD_CUML_TESTS)
  include(CTest)
  add_subdirectory(test)
endif()

##############################################################################
# - build examples -----------------------------------------------------------

if(BUILD_CUML_EXAMPLES)
  add_subdirectory(examples)
endif()

# ###################################################################################################
# # - install targets -------------------------------------------------------------------------------
rapids_cmake_install_lib_dir( lib_dir )
include(CPack)

set(CUML_TARGETS ${CUML_CPP_TARGET})

if(BUILD_CUML_C_LIBRARY)
  list(APPEND CUML_TARGETS
         ${CUML_C_TARGET})
endif()

install(TARGETS
          ${CUML_TARGETS}
        DESTINATION
          ${lib_dir}
        EXPORT
          cuml-exports)

install(DIRECTORY include/cuml/
        DESTINATION include/cuml)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/cuml/version_config.hpp
        DESTINATION include/cuml)

# ################################################################################################
# # - install export -------------------------------------------------------------------------------
set(doc_string
[=[
Provide targets for cuML.

cuML is a suite of libraries that implement machine learning algorithms and mathematical primitives
functions that share compatible APIs with other RAPIDS projects.

]=])

 rapids_export(INSTALL cuml
    EXPORT_SET cuml-exports
    GLOBAL_TARGETS cuml
    NAMESPACE cuml::
    DOCUMENTATION doc_string
    )

################################################################################################
# - build export -------------------------------------------------------------------------------
set(code_string [=[thrust_create_target(cuml::Thrust FROM_OPTIONS)]=])
rapids_export(BUILD cuml
    EXPORT_SET cuml-exports
    GLOBAL_TARGETS cuml
    NAMESPACE cuml::
    DOCUMENTATION doc_string
    FINAL_CODE_BLOCK code_string
    )

##############################################################################
# - build benchmark executable -----------------------------------------------

if(BUILD_CUML_BENCH OR BUILD_CUML_PRIMS_BENCH)
  add_subdirectory(bench)
endif()

##############################################################################
# - doxygen targets ----------------------------------------------------------

include(cmake/doxygen.cmake)
add_doxygen_target(IN_DOXYFILE Doxyfile.in
  OUT_DOXYFILE ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
  CWD ${CMAKE_CURRENT_BINARY_DIR})
