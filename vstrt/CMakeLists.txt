cmake_minimum_required(VERSION 3.20)

project(vs-trt LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

set(VAPOURSYNTH_INCLUDE_DIRECTORY "" CACHE PATH "Path to VapourSynth headers")
set(TENSORRT_HOME "" CACHE PATH "Path to TensorRT")
option(USE_NVINFER_PLUGIN "Initialize nvinfer_plugin" FALSE)

FIND_PACKAGE(CUDAToolkit REQUIRED)

add_library(vstrt SHARED 
    vs_tensorrt.cpp
    win32.cpp
)

target_include_directories(vstrt PRIVATE
    ${VAPOURSYNTH_INCLUDE_DIRECTORY}
    ${CUDAToolkit_INCLUDE_DIRS}
    ${TENSORRT_HOME}/include
)

set_target_properties(vstrt PROPERTIES
    CXX_EXTENSIONS OFF
    POSITION_INDEPENDENT_CODE ON
    CXX_STANDARD 20
    CXX_STANDARD_REQUIRED ON
)

target_link_directories(vstrt PRIVATE ${TENSORRT_HOME}/lib)
target_link_libraries(vstrt PRIVATE CUDA::cudart_static nvinfer)

if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    target_link_options(vstrt PRIVATE "/DELAYLOAD:nvinfer.dll" "delayimp.lib")
endif()

if (USE_NVINFER_PLUGIN)
    add_definitions(-DUSE_NVINFER_PLUGIN)
    target_link_libraries(vstrt PRIVATE nvinfer_plugin)

    if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
        target_link_options(vstrt PRIVATE "/DELAYLOAD:nvinfer_plugin.dll")
    endif()
endif()

install(TARGETS vstrt
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)