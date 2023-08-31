#  ----------------------------------------------------------------------------
#  2020 Bernd Pfrommer bernd.pfrommer@gmail.com
#

cmake_minimum_required(VERSION 3.5)
project(flir_spinnaker_ros2)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif()

# list of all packages that use ament for export (may work for others too)

set(ROS2_DEPENDENCIES
  "rclcpp"
  "rclcpp_components"
  "sensor_msgs"
  "std_msgs"
  "camera_info_manager"
  "image_transport"
  "flir_spinnaker_common"
  "image_meta_msgs_ros2"
  "camera_control_msgs_ros2"
  "OpenCV"
)

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(ament_cmake_auto REQUIRED)
find_package(ament_cmake_ros REQUIRED)

foreach(pkg ${ROS2_DEPENDENCIES})
  find_package(${pkg} REQUIRED)
endforeach()

###### Cuda ######
set(CUDA_LIBRARIES "libcuda.so")
find_package(CUDA  REQUIRED)
include_directories("${CUDA_INCLUDE_DIRS}")
###### END Cuda ######


################ FFMPEG Stuff ################
#set(FFMPEG_PKGCONFIG "/home/nuc-haptics/ffmpeg_build/lib/pkgconfig" CACHE STRING "ffmpeg_pkgconfig string")
#set(CMAKE_INSTALL_RPATH "/home/nuc-haptics/ffmpeg_build/lib")
set(FFMPEG_PKGCONFIG "" CACHE STRING "extra path to pkgconfig")
if("${FFMPEG_PKGCONFIG}" STREQUAL "")
else()
  message(WARNING "using FFMPEG package from ${FFMPEG_PKGCONFIG}")
endif()

set(ENV{PKG_CONFIG_PATH} ${FFMPEG_PKGCONFIG})
find_package(PkgConfig REQUIRED)

pkg_check_modules(LIBAV REQUIRED IMPORTED_TARGET
    libavcodec
    libswresample
    libswscale
    libavutil)

# link_libraries("/../../install/ffmpeg_image_transport/lib/libffmpeg_image_transport_component.so")
find_library(FFMPEG_IMG_TRANSPORT_LIB ffmpeg_image_transport_component HINTS "/../../install/ffmpeg_image_transport/lib/")
################

ament_auto_find_build_dependencies(REQUIRED ${ROS2_DEPENDENCIES})

ament_auto_add_library(camera_driver SHARED
  src/camera_driver.cpp
)

ament_auto_add_library(encoder_driver SHARED
  src/encoder_driver.cpp
)

ament_auto_add_executable(camera_driver_node
  src/camera_driver_node.cpp
)

ament_auto_add_executable(encoder_driver_node
  src/encoder_driver_node.cpp
)

rclcpp_components_register_nodes(camera_driver "flir_spinnaker_ros2::CameraDriver")
# rclcpp_components_register_nodes(encoder_driver "flir_spinnaker_ros2::EncoderDriver") # don't need this, should be able to run it using the encoder_driver_node


target_include_directories(camera_driver PRIVATE include)
target_include_directories(encoder_driver PRIVATE include)
target_include_directories(encoder_driver PRIVATE "../../install/ffmpeg_image_transport/include/ffmpeg_image_transport")
target_include_directories(encoder_driver PRIVATE "../../install/ffmpeg_image_transport_msgs/include/ffmpeg_image_transport_msgs")
target_include_directories(encoder_driver_node PRIVATE "../../install/ffmpeg_image_transport/include/ffmpeg_image_transport")
target_include_directories(encoder_driver_node PRIVATE "../../install/ffmpeg_image_transport_msgs/include/ffmpeg_image_transport_msgs")
target_link_libraries(encoder_driver PkgConfig::LIBAV ${FFMPEG_IMG_TRANSPORT_LIB})
target_link_libraries(encoder_driver_node PkgConfig::LIBAV)

# the node must go into the project specific lib directory or else
# the launch file will not find it

install(TARGETS
  camera_driver_node
  encoder_driver_node
  DESTINATION lib/${PROJECT_NAME}/)

# the shared library goes into the global lib dir so it can
# be used as a composable node by other projects

install(TARGETS
  camera_driver
  encoder_driver
  DESTINATION lib
)

install(DIRECTORY
  config
  DESTINATION share/${PROJECT_NAME}/
)

install(DIRECTORY
  launch
  DESTINATION share/${PROJECT_NAME}/
)

if(BUILD_TESTING)
  find_package(ament_cmake REQUIRED)
  find_package(ament_cmake_copyright REQUIRED)
  find_package(ament_cmake_cppcheck REQUIRED)
  find_package(ament_cmake_cpplint REQUIRED)
  find_package(ament_cmake_flake8 REQUIRED)
  find_package(ament_cmake_lint_cmake REQUIRED)
  find_package(ament_cmake_pep257 REQUIRED)
  find_package(ament_cmake_clang_format REQUIRED)
  find_package(ament_cmake_xmllint REQUIRED)

  ament_copyright()
  ament_cppcheck(LANGUAGE c++)
  ament_cpplint(FILTERS "-build/include,-runtime/indentation_namespace")
  ament_flake8()
  ament_lint_cmake()
  ament_pep257()
  ament_clang_format(CONFIG_FILE .clang-format)
  ament_xmllint()
endif()

ament_package()
