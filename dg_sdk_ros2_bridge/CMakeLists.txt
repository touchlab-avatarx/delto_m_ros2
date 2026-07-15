cmake_minimum_required(VERSION 3.8)
project(dg_sdk_ros2_bridge)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(dg_msgs REQUIRED)
find_package(std_msgs REQUIRED)

# Include directories
include_directories(
  include
  ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# Add executable
add_executable(dg_sdk_ros2_bridge_node
  src/dg_sdk_ros2_bridge_node.cpp
)

# Select vendor DGSDK library for the target architecture
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64)$")
  # NOTE: vendor DGSDK version unconfirmed for ARM64 - file ships unversioned.
  # Confirm with vendor which release this corresponds to before relying on it in production.
  set(DGSDK_LIB ${CMAKE_CURRENT_SOURCE_DIR}/libs/aarch64/libDGSDK.so)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64)$")
  set(DGSDK_LIB ${CMAKE_CURRENT_SOURCE_DIR}/libs/x86_64/libDGSDK_171.so)
else()
  message(FATAL_ERROR "dg_sdk_ros2_bridge: no vendor DGSDK library for architecture '${CMAKE_SYSTEM_PROCESSOR}'")
endif()

# Link libraries
target_link_libraries(dg_sdk_ros2_bridge_node
  ${DGSDK_LIB}
)

# Target dependencies
ament_target_dependencies(dg_sdk_ros2_bridge_node
  rclcpp
  dg_msgs
  std_msgs
)

# Install executable
install(TARGETS
  dg_sdk_ros2_bridge_node
  DESTINATION lib/${PROJECT_NAME}
)

# Install headers
install(DIRECTORY include/
  DESTINATION include/
  FILES_MATCHING PATTERN "*.h"
)

# Install shared library
install(FILES ${DGSDK_LIB}
  DESTINATION lib
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()