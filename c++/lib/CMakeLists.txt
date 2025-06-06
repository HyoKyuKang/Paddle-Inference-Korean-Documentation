cmake_minimum_required(VERSION 3.0)
project(cpp_inference_demo CXX C CUDA)

option(WITH_MKL "Compile demo with MKL/OpenBlas support, default use MKL." ON)
option(WITH_GPU "Compile demo with GPU/CPU, default use CPU." OFF)
option(WITH_STATIC_LIB
       "Compile demo with static/shared library, default don't use static." OFF)
option(USE_TENSORRT "Compile demo with TensorRT." OFF)
option(WITH_OPENVINO "Compile demo with OpenVINO." OFF)
option(WITH_SHARED_PHI "Compile demo with phi shared lib" ON)
option(WITH_ROCM "Compile demo with rocm." OFF)
option(WITH_ONNXRUNTIME "Compile demo with ONNXRuntime" OFF)
option(WITH_ARM "Compile demo with ARM" OFF)
option(WITH_MIPS "Compile demo with MIPS" OFF)
option(WITH_LOONGARCH "Compile demo with LOONGARCH" OFF)
option(WITH_SW "Compile demo with SW" OFF)
option(WITH_XPU "Compile demo with xpu" OFF)
option(WITH_NPU "Compile demo with npu" OFF)
option(CUSTOM_OPERATOR_FILES "List of file names for custom operators" "")
option(CUSTOM_PASS_FILES "List of file names for custom passes" "")

if(NOT WITH_STATIC_LIB)
  add_definitions("-DPADDLE_WITH_SHARED_LIB")
else()
  # PD_INFER_DECL is mainly used to set the dllimport/dllexport attribute in dynamic library mode.
  # Set it to empty in static library mode to avoid compilation issues.
  add_definitions("/DPD_INFER_DECL=")
endif()

macro(safe_set_static_flag)
  foreach(flag_var
          CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
          CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    if(${flag_var} MATCHES "/MD")
      string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
    endif()
  endforeach()
endmacro()

if(NOT DEFINED PADDLE_LIB)
  message(
    FATAL_ERROR "please set PADDLE_LIB with -DPADDLE_LIB=/path/paddle/lib")
endif()
if(NOT DEFINED DEMO_NAME)
  message(FATAL_ERROR "please set DEMO_NAME with -DDEMO_NAME=demo_name")
endif()

include_directories("${PADDLE_LIB}/paddle/include")
set(PADDLE_LIB_THIRD_PARTY_PATH "${PADDLE_LIB}/third_party/install/")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}protobuf/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}glog/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}gflags/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}xxhash/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}flashattn/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}cryptopp/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}onnxruntime/include")
include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}paddle2onnx/include")

link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}protobuf/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}glog/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}gflags/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}xxhash/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}flashattn/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}cryptopp/lib")
link_directories("${PADDLE_LIB}/paddle/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}onnxruntime/lib")
link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}paddle2onnx/lib")

if(WITH_GPU)
  find_package(CUDA REQUIRED)
  add_definitions("-DPADDLE_WITH_CUDA")
endif()

if(USE_TENSORRT)
  add_definitions("-DPADDLE_WITH_TENSORRT")
endif()

if(WIN32)
  add_definitions("/DGOOGLE_GLOG_DLL_DECL=")
  option(MSVC_STATIC_CRT "use static C Runtime library by default" ON)
  if(MSVC_STATIC_CRT)
    if(WITH_MKL)
      set(FLAG_OPENMP "/openmp")
    endif()
    set(CMAKE_C_FLAGS_DEBUG
        "${CMAKE_C_FLAGS_DEBUG} /bigobj /MTd ${FLAG_OPENMP}")
    set(CMAKE_C_FLAGS_RELEASE
        "${CMAKE_C_FLAGS_RELEASE} /bigobj /MT ${FLAG_OPENMP}")
    set(CMAKE_CXX_FLAGS_DEBUG
        "${CMAKE_CXX_FLAGS_DEBUG} /bigobj /MTd ${FLAG_OPENMP}")
    set(CMAKE_CXX_FLAGS_RELEASE
        "${CMAKE_CXX_FLAGS_RELEASE} /bigobj /MT ${FLAG_OPENMP}")
    safe_set_static_flag()
    if(WITH_STATIC_LIB)
      add_definitions(-DSTATIC_LIB)
    endif()
  endif()
else()
  if(WITH_MKL)
    set(FLAG_OPENMP "-fopenmp")
  endif()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 ${FLAG_OPENMP}")
endif()

if(WITH_GPU)
  if(NOT WIN32)
    include_directories("/usr/local/cuda/include")
    if(CUDA_LIB STREQUAL "")
      set(CUDA_LIB
          "/usr/local/cuda/lib64/"
          CACHE STRING "CUDA Library")
    endif()
  else()
    include_directories(
      "C:\\Program\ Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v8.0\\include")
    if(CUDA_LIB STREQUAL "")
      set(CUDA_LIB
          "C:\\Program\ Files\\NVIDIA GPU Computing Toolkit\\CUDA\\v8.0\\lib\\x64"
      )
    endif()
  endif()
endif()

if(USE_TENSORRT AND WITH_GPU)
  set(TENSORRT_ROOT
      ""
      CACHE STRING "The root directory of TensorRT library")
  if("${TENSORRT_ROOT}" STREQUAL "")
    message(
      FATAL_ERROR
        "The TENSORRT_ROOT is empty, you must assign it a value with CMake command. Such as: -DTENSORRT_ROOT=TENSORRT_ROOT_PATH "
    )
  endif()
  set(TENSORRT_INCLUDE_DIR ${TENSORRT_ROOT}/include)
  set(TENSORRT_LIB_DIR ${TENSORRT_ROOT}/lib)
  file(READ ${TENSORRT_INCLUDE_DIR}/NvInfer.h TENSORRT_VERSION_FILE_CONTENTS)
  string(REGEX MATCH "define NV_TENSORRT_MAJOR +([0-9]+)"
               TENSORRT_MAJOR_VERSION "${TENSORRT_VERSION_FILE_CONTENTS}")
  if("${TENSORRT_MAJOR_VERSION}" STREQUAL "")
    file(READ ${TENSORRT_INCLUDE_DIR}/NvInferVersion.h
         TENSORRT_VERSION_FILE_CONTENTS)
    string(REGEX MATCH "define NV_TENSORRT_MAJOR +([0-9]+)"
                 TENSORRT_MAJOR_VERSION "${TENSORRT_VERSION_FILE_CONTENTS}")
  endif()
  if("${TENSORRT_MAJOR_VERSION}" STREQUAL "")
    message(SEND_ERROR "Failed to detect TensorRT version.")
  endif()
  string(REGEX REPLACE "define NV_TENSORRT_MAJOR +([0-9]+)" "\\1"
                       TENSORRT_MAJOR_VERSION "${TENSORRT_MAJOR_VERSION}")
  message(
    STATUS "Current TensorRT header is ${TENSORRT_INCLUDE_DIR}/NvInfer.h. "
           "Current TensorRT version is v${TENSORRT_MAJOR_VERSION}. ")
  include_directories("${TENSORRT_INCLUDE_DIR}")
  link_directories("${TENSORRT_LIB_DIR}")
endif()

if(WITH_MKL)
  set(MATH_LIB_PATH "${PADDLE_LIB_THIRD_PARTY_PATH}mklml")
  include_directories("${MATH_LIB_PATH}/include")
  if(WIN32)
    set(MATH_LIB ${MATH_LIB_PATH}/lib/mklml${CMAKE_STATIC_LIBRARY_SUFFIX}
                 ${MATH_LIB_PATH}/lib/libiomp5md${CMAKE_STATIC_LIBRARY_SUFFIX})
  else()
    set(MATH_LIB
        ${MATH_LIB_PATH}/lib/libmklml_intel${CMAKE_SHARED_LIBRARY_SUFFIX}
        ${MATH_LIB_PATH}/lib/libiomp5${CMAKE_SHARED_LIBRARY_SUFFIX})
  endif()
  set(MKLDNN_PATH "${PADDLE_LIB_THIRD_PARTY_PATH}onednn")
  if(EXISTS ${MKLDNN_PATH})
    include_directories("${MKLDNN_PATH}/include")
    if(WIN32)
      set(MKLDNN_LIB ${MKLDNN_PATH}/lib/mkldnn.lib)
    else()
      set(MKLDNN_LIB ${MKLDNN_PATH}/lib/libdnnl.so.3)
    endif()
  endif()
elseif((NOT (WITH_MIPS OR WITH_LOONGARCH)) AND (NOT WITH_SW) AND (NOT APPLE))
  set(OPENBLAS_LIB_PATH "${PADDLE_LIB_THIRD_PARTY_PATH}openblas")
  include_directories("${OPENBLAS_LIB_PATH}/include/openblas")
  if(WIN32)
    set(MATH_LIB
        ${OPENBLAS_LIB_PATH}/lib/openblas${CMAKE_STATIC_LIBRARY_SUFFIX})
  else()
    set(MATH_LIB
        ${OPENBLAS_LIB_PATH}/lib/libopenblas${CMAKE_STATIC_LIBRARY_SUFFIX})
  endif()
endif()

if(WITH_STATIC_LIB)
  set(DEPS
      ${PADDLE_LIB}/paddle/lib/libpaddle_inference${CMAKE_STATIC_LIBRARY_SUFFIX}
  )
else()
  if(WIN32)
    set(DEPS
        ${PADDLE_LIB}/paddle/lib/paddle_inference${CMAKE_STATIC_LIBRARY_SUFFIX})
  else()
    set(DEPS
        ${PADDLE_LIB}/paddle/lib/libpaddle_inference${CMAKE_SHARED_LIBRARY_SUFFIX}
    )
  endif()
endif()

if(WITH_ONNXRUNTIME)
  if(WIN32)
    set(DEPS
        ${DEPS} ${PADDLE_LIB_THIRD_PARTY_PATH}onnxruntime/lib/onnxruntime.lib
        paddle2onnx)
  elseif(APPLE)
    set(DEPS
        ${DEPS}
        ${PADDLE_LIB_THIRD_PARTY_PATH}onnxruntime/lib/libonnxruntime.1.10.0.dylib
        paddle2onnx)
  else()
    set(DEPS
        ${DEPS}
        ${PADDLE_LIB_THIRD_PARTY_PATH}onnxruntime/lib/libonnxruntime.so.1.10.0
        paddle2onnx)
  endif()
endif()

if(NOT WIN32)
  set(EXTERNAL_LIB "-lrt -ldl -lpthread")
  set(DEPS
      ${DEPS}
      ${MATH_LIB}
      ${MKLDNN_LIB}
      glog
      gflags
      protobuf
      xxhash
      cryptopp
      ${EXTERNAL_LIB})
  if(WITH_SHARED_PHI)
    set(DEPS ${DEPS} ${PADDLE_LIB}/paddle/lib/libphi${CMAKE_SHARED_LIBRARY_SUFFIX})
    set(DEPS ${DEPS} ${PADDLE_LIB}/paddle/lib/libphi_core${CMAKE_SHARED_LIBRARY_SUFFIX})
    if(WITH_GPU)
       set(DEPS ${DEPS} ${PADDLE_LIB}/paddle/lib/libphi_gpu${CMAKE_SHARED_LIBRARY_SUFFIX})
    endif()
  endif()
  set(DEPS ${DEPS} ${PADDLE_LIB}/paddle/lib/libcommon${CMAKE_SHARED_LIBRARY_SUFFIX})
else()
  set(DEPS
      ${DEPS}
      ${MATH_LIB}
      ${MKLDNN_LIB}
      glog
      gflags_static
      libprotobuf
      xxhash
      cryptopp-static
      ${EXTERNAL_LIB})
  set(DEPS ${DEPS} shlwapi.lib)
  set(DEPS ${DEPS} ${PADDLE_LIB}/paddle/lib/common.lib)
endif()

if(WITH_OPENVINO)
  if(LINUX)
    add_definitions("-DPADDLE_WITH_OPENVINO")
    include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}/openvino/include")
    include_directories("${PADDLE_LIB_THIRD_PARTY_PATH}/tbb/include")
    string(TOLOWER ${CMAKE_SYSTEM_PROCESSOR} ARCH)
    if(ARCH STREQUAL "x86_64" OR ARCH STREQUAL "amd64"
        OR CMAKE_OSX_ARCHITECTURES STREQUAL "x86_64")
        set(ARCH intel64)
    elseif(ARCH STREQUAL "i386")
        set(ARCH ia32)
    endif()
    link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}openvino/${ARCH}")
    link_directories("${PADDLE_LIB_THIRD_PARTY_PATH}tbb/lib")
    if(UNIX)
      set(DEPS ${DEPS} openvino tbb openvino_paddle_frontend openvino_intel_cpu_plugin)
    endif()
  endif()
endif()

if(WITH_GPU)
  if(NOT WIN32)
    if(USE_TENSORRT)
      set(DEPS ${DEPS}
               ${TENSORRT_LIB_DIR}/libnvinfer${CMAKE_SHARED_LIBRARY_SUFFIX})
      set(DEPS
          ${DEPS}
          ${TENSORRT_LIB_DIR}/libnvinfer_plugin${CMAKE_SHARED_LIBRARY_SUFFIX})
    endif()
    set(DEPS ${DEPS} ${CUDA_LIB}/libcudart${CMAKE_SHARED_LIBRARY_SUFFIX})
  else()
    if(USE_TENSORRT)
      set(DEPS ${DEPS}
               ${TENSORRT_LIB_DIR}/nvinfer${CMAKE_STATIC_LIBRARY_SUFFIX})
      set(DEPS ${DEPS}
               ${TENSORRT_LIB_DIR}/nvinfer_plugin${CMAKE_STATIC_LIBRARY_SUFFIX})
      if(${TENSORRT_MAJOR_VERSION} EQUAL 7)
        set(DEPS ${DEPS}
                 ${TENSORRT_LIB_DIR}/myelin64_1${CMAKE_STATIC_LIBRARY_SUFFIX})
      endif()
    endif()
    set(DEPS ${DEPS} ${CUDA_LIB}/cudart${CMAKE_STATIC_LIBRARY_SUFFIX})
    set(DEPS ${DEPS} ${CUDA_LIB}/cublas${CMAKE_STATIC_LIBRARY_SUFFIX})
    set(DEPS ${DEPS} ${CUDA_LIB}/cudnn${CMAKE_STATIC_LIBRARY_SUFFIX})
  endif()
endif()

if(WITH_ROCM AND NOT WIN32)
  set(DEPS ${DEPS} ${ROCM_LIB}/libamdhip64${CMAKE_SHARED_LIBRARY_SUFFIX})
endif()

if(WITH_XPU AND NOT WIN32)
  set(XPU_INSTALL_PATH "${PADDLE_LIB_THIRD_PARTY_PATH}xpu")
  set(DEPS ${DEPS}
           ${XPU_INSTALL_PATH}/lib/libxpuapi${CMAKE_SHARED_LIBRARY_SUFFIX})
  set(DEPS ${DEPS}
           ${XPU_INSTALL_PATH}/lib/libxpurt${CMAKE_SHARED_LIBRARY_SUFFIX})
endif()

if(WITH_NPU AND NOT WIN32)
  set(DEPS
      ${DEPS}
      ${ASCEND_DIR}/ascend-toolkit/latest/fwkacllib/lib64/libgraph${CMAKE_SHARED_LIBRARY_SUFFIX}
  )
  set(DEPS
      ${DEPS}
      ${ASCEND_DIR}/ascend-toolkit/latest/fwkacllib/lib64/libge_runner${CMAKE_SHARED_LIBRARY_SUFFIX}
  )
  set(DEPS
      ${DEPS}
      ${ASCEND_DIR}/ascend-toolkit/latest/fwkacllib/lib64/libascendcl${CMAKE_SHARED_LIBRARY_SUFFIX}
  )
  set(DEPS
      ${DEPS}
      ${ASCEND_DIR}/ascend-toolkit/latest/fwkacllib/lib64/libascendcl${CMAKE_SHARED_LIBRARY_SUFFIX}
  )
  set(DEPS
      ${DEPS}
      ${ASCEND_DIR}/ascend-toolkit/latest/fwkacllib/lib64/libacl_op_compiler${CMAKE_SHARED_LIBRARY_SUFFIX}
  )
endif()

if(CUSTOM_OPERATOR_FILES) 
  add_library(pd_infer_custom_op SHARED ${CUSTOM_OPERATOR_FILES})
  set(DEPS ${DEPS} pd_infer_custom_op)
endif()

if(CUSTOM_PASS_FILES)
  add_library(pd_infer_custom_pass SHARED ${CUSTOM_PASS_FILES})
  set(DEPS ${DEPS} pd_infer_custom_pass)
endif()

add_executable(${DEMO_NAME} ${DEMO_NAME}.cc)
target_link_libraries(${DEMO_NAME} ${DEPS})

if(WIN32)
  if(USE_TENSORRT)
    add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${TENSORRT_LIB_DIR}/nvinfer${CMAKE_SHARED_LIBRARY_SUFFIX}
        ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE}
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${TENSORRT_LIB_DIR}/nvinfer_plugin${CMAKE_SHARED_LIBRARY_SUFFIX}
        ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
    if(${TENSORRT_MAJOR_VERSION} EQUAL 7)
      add_custom_command(
        TARGET ${DEMO_NAME}
        POST_BUILD
        COMMAND
          ${CMAKE_COMMAND} -E copy
          ${TENSORRT_LIB_DIR}/myelin64_1${CMAKE_SHARED_LIBRARY_SUFFIX}
          ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
    endif()
  endif()
  if(WITH_MKL)
    add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy ${MATH_LIB_PATH}/lib/mklml.dll
              ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE}
      COMMAND ${CMAKE_COMMAND} -E copy ${MATH_LIB_PATH}/lib/libiomp5md.dll
              ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
    set(MKLDNN_PATH "${PADDLE_LIB_THIRD_PARTY_PATH}onednn")
    if(EXISTS ${MKLDNN_PATH})
      add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy ${MKLDNN_PATH}/lib/mkldnn.dll
              ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
    endif()
  else()
    add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy ${OPENBLAS_LIB_PATH}/lib/openblas.dll
              ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
  endif()
  if(WITH_ONNXRUNTIME)
    add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${PADDLE_LIB_THIRD_PARTY_PATH}onnxruntime/lib/onnxruntime.dll
        ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE}
      COMMAND
        ${CMAKE_COMMAND} -E copy
        ${PADDLE_LIB_THIRD_PARTY_PATH}paddle2onnx/lib/paddle2onnx.dll
        ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
  endif()
  if(NOT WITH_STATIC_LIB)
    add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND
        ${CMAKE_COMMAND} -E copy
        "${PADDLE_LIB}/paddle/lib/paddle_inference.dll"
        ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
  endif()
  add_custom_command(
      TARGET ${DEMO_NAME}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy ${PADDLE_LIB}/paddle/lib/common.dll
              ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE})
endif()
