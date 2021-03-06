cmake_minimum_required(VERSION 3.14)
project(torch-js)

set(TORCH_VERSION 1.7.1)
set(TORCH_DEFAULT_CUDA "102")
# set(TORCHVISION_VERSION v0.9.0-rc2)
set(TORCHVISION_VERSION v0.8.2)


# Not had libtorch at source dir
if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/build/libtorch/")
  find_package(CUDA)
  if(CUDA_FOUND)
    # Check for unsupported CUDA version
    if(NOT "${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}" MATCHES
       "^(101|102|110)$")
      message(
        FATAL_ERROR
          "${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR} are not provided by libtorch. Please build from source and place it at ${CMAKE_CURRENT_BINARY_DIR}/libtorch/"
      )
    endif()
    set(LIBTORCH_SAVE_PATH
        "${CMAKE_CURRENT_SOURCE_DIR}/libtorch-cu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}.zip"
    )
    if(NOT EXISTS ${LIBTORCH_SAVE_PATH})
      if(WIN32)
        if("${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}" STREQUAL
           ${TORCH_DEFAULT_CUDA})
          set(DOWNLOAD_LINK
              "https://download.pytorch.org/libtorch/cu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}/libtorch-win-shared-with-deps-${TORCH_VERSION}.zip"
          )
          message(
            STATUS "Downloading libtorch for Windows from ${DOWNLOAD_LINK}")
          file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
        else()
          set(DOWNLOAD_LINK
              "https://download.pytorch.org/libtorch/cu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}/libtorch-win-shared-with-deps-${TORCH_VERSION}%2Bcu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}-.zip"
          )
          message(
            STATUS "Downloading libtorch for Windows from ${DOWNLOAD_LINK}")
          file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
        endif()
      elseif(APPLE)
        message(
          SEND_ERROR
            "MacOS does not support NVIDIA GPU, hence CUDA, will use the CPU version instead"
        )
        set(DOWNLOAD_LINK
            "https://download.pytorch.org/libtorch/cu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}/libtorch-macos-${TORCH_VERSION}.zip"
        )
        message(STATUS "Downloading libtorch for MacOS from ${DOWNLOAD_LINK}")
        file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
      else()
        if("${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}" STREQUAL
           ${TORCH_DEFAULT_CUDA})
          set(DOWNLOAD_LINK
              "https://download.pytorch.org/libtorch/cu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}/libtorch-shared-with-deps-${TORCH_VERSION}.zip"
          )
          message(STATUS "Downloading libtorch for Linux from ${DOWNLOAD_LINK}")
          file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
        else()
          set(DOWNLOAD_LINK
              "https://download.pytorch.org/libtorch/cu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}/libtorch-shared-with-deps-${TORCH_VERSION}%2Bcu${CUDA_VERSION_MAJOR}${CUDA_VERSION_MINOR}.zip"
          )
          message(STATUS "Downloading libtorch for Linux from ${DOWNLOAD_LINK}")
          file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
        endif()
      endif()
    endif()
    message(STATUS "Unziping ${LIBTORCH_SAVE_PATH}")
    execute_process(COMMAND unzip -qq -o ${LIBTORCH_SAVE_PATH})
  else()
    set(LIBTORCH_SAVE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/libtorch-cpu.zip")
    if(NOT EXISTS ${LIBTORCH_SAVE_PATH})
      if(WIN32)
        set(DOWNLOAD_LINK
            "https://download.pytorch.org/libtorch/cpu/libtorch-win-shared-with-deps-${TORCH_VERSION}%2Bcpu.zip"
        )
        message(STATUS "Downloading libtorch for Windows from ${DOWNLOAD_LINK}")
        file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
      elseif(APPLE)
        set(DOWNLOAD_LINK
            "https://download.pytorch.org/libtorch/cpu/libtorch-macos-${TORCH_VERSION}.zip"
        )
        message(STATUS "Downloading libtorch for MacOS from ${DOWNLOAD_LINK}")
        file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
      else()
        set(DOWNLOAD_LINK
            "https://download.pytorch.org/libtorch/cpu/libtorch-shared-with-deps-${TORCH_VERSION}%2Bcpu.zip"
        )
        message(STATUS "Downloading libtorch for Linux from ${DOWNLOAD_LINK}")
        file(DOWNLOAD ${DOWNLOAD_LINK} ${LIBTORCH_SAVE_PATH})
      endif()
    endif()
    message(STATUS "Unziping ${LIBTORCH_SAVE_PATH}")
    execute_process(COMMAND unzip -qq -o ${LIBTORCH_SAVE_PATH})
  endif(CUDA_FOUND)
else()
  message(STATUS "libtorch found in ${CMAKE_CURRENT_BINARY_DIR}/libtorch/")
endif()

# Use C++14
set(CMAKE_CXX_STANDARD 14)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_PREFIX_PATH ${CMAKE_CURRENT_BINARY_DIR}/libtorch/)
find_package(Torch REQUIRED)

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/torchvision")
  execute_process(
    COMMAND git clone --depth 1 --branch ${TORCHVISION_VERSION}
            https://github.com/pytorch/vision)
  execute_process(
    COMMAND
      git -C "${CMAKE_CURRENT_BINARY_DIR}/vision/" apply
      "${CMAKE_CURRENT_SOURCE_DIR}/patches/0001-Remove-native-image-support.patch"
  )
  execute_process(
    COMMAND
      ${CMAKE_COMMAND} -DCMAKE_BUILD_TYPE=Release -DWITH_CUDA=${CUDA_FOUND}
      -DCMAKE_PREFIX_PATH=${CMAKE_CURRENT_BINARY_DIR}/libtorch -S
      "${CMAKE_CURRENT_BINARY_DIR}/vision/" -B
      "${CMAKE_CURRENT_BINARY_DIR}/vision/build/")
  execute_process(
    COMMAND ${CMAKE_COMMAND} --build "${CMAKE_CURRENT_BINARY_DIR}/vision/build"
            --config Release -j${PROCESSOR_COUNT})
  execute_process(
    COMMAND
      ${CMAKE_COMMAND} --install "${CMAKE_CURRENT_BINARY_DIR}/vision/build"
      --prefix "${CMAKE_CURRENT_BINARY_DIR}/torchvision")
endif()

set(CMAKE_PREFIX_PATH
    ${CMAKE_PREFIX_PATH};${CMAKE_CURRENT_BINARY_DIR}/torchvision/)
find_package(TorchVision)

if(WIN32)

elseif(APPLE)
  set(CMAKE_BUILD_RPATH @loader_path;@executable_path)
else()
  set(CMAKE_BUILD_RPATH $ORIGIN)
endif()

include_directories(${CMAKE_JS_INC})

file(GLOB SOURCE_FILES "src/*.cc" "src/*.h")
add_library(${PROJECT_NAME} SHARED ${SOURCE_FILES} ${CMAKE_JS_SRC})
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "" SUFFIX ".node")

# Include N-API wrappers
execute_process(
  COMMAND node -p "require('node-addon-api').include"
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE NODE_ADDON_API_DIR)
string(REPLACE "\n" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
string(REPLACE "\"" "" NODE_ADDON_API_DIR ${NODE_ADDON_API_DIR})
target_include_directories(${PROJECT_NAME} PRIVATE ${NODE_ADDON_API_DIR})
add_definitions(-DNAPI_VERSION=3)

target_link_libraries(${PROJECT_NAME} ${TORCH_LIBRARIES})

target_include_directories(
  ${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/torchvision/include")
target_link_libraries(${PROJECT_NAME} TorchVision::TorchVision)

target_link_libraries(${PROJECT_NAME} ${CMAKE_JS_LIB})

add_custom_command(
  TARGET torch-js
  POST_BUILD
  COMMAND
    ${CMAKE_COMMAND} -E copy_directory
    "${CMAKE_CURRENT_BINARY_DIR}/libtorch/lib" $<TARGET_FILE_DIR:torch-js>)

if(EXISTS "${CMAKE_CURRENT_BINARY_DIR}/torchvision/bin")
  add_custom_command(
    TARGET torch-js
    POST_BUILD
    COMMAND
      ${CMAKE_COMMAND} -E copy_directory
      "${CMAKE_CURRENT_BINARY_DIR}/torchvision/bin" $<TARGET_FILE_DIR:torch-js>)
elseif(EXISTS "${CMAKE_CURRENT_BINARY_DIR}/torchvision/lib")
  add_custom_command(
    TARGET torch-js
    POST_BUILD
    COMMAND
      ${CMAKE_COMMAND} -E copy_directory
      "${CMAKE_CURRENT_BINARY_DIR}/torchvision/lib" $<TARGET_FILE_DIR:torch-js>)
elseif(EXISTS "${CMAKE_CURRENT_BINARY_DIR}/torchvision/lib64")
  add_custom_command(
    TARGET torch-js
    POST_BUILD
    COMMAND
      ${CMAKE_COMMAND} -E copy_directory
      "${CMAKE_CURRENT_BINARY_DIR}/torchvision/lib64"
      $<TARGET_FILE_DIR:torch-js>)
endif()
