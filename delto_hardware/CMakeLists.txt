cmake_minimum_required(VERSION 3.8)
project(delto_hardware)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

set(CMAKE_CXX_STANDARD 17)

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(hardware_interface REQUIRED)
find_package(pluginlib REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)
find_package(std_srvs REQUIRED)
find_package(delto_tcp_comm REQUIRED)
find_package(Boost REQUIRED COMPONENTS system)

# Select vendor delto_gripper_helper library for the target architecture
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64)$")
  # NOTE: vendor build provenance/version unconfirmed for ARM64 - confirm before production use.
  set(GRIPPER_HELPER_LIB ${CMAKE_CURRENT_SOURCE_DIR}/lib/aarch64/libdelto_gripper_helper.so)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64)$")
  set(GRIPPER_HELPER_LIB ${CMAKE_CURRENT_SOURCE_DIR}/lib/x86_64/libdelto_gripper_helper.so)
else()
  message(FATAL_ERROR "delto_hardware: no vendor delto_gripper_helper library for architecture '${CMAKE_SYSTEM_PROCESSOR}'")
endif()

# Import delto_gripper_helper library (shared across all drivers).
# The vendor .so ships without an embedded SONAME, so IMPORTED_NO_SONAME
# tells CMake to link it by name (-l) instead of embedding this source-tree
# path in consumers' DT_NEEDED, which would otherwise break once the source
# directory is removed post-install.
add_library(delto_gripper_helper SHARED IMPORTED)
set_target_properties(delto_gripper_helper PROPERTIES
  IMPORTED_LOCATION ${GRIPPER_HELPER_LIB}
  IMPORTED_NO_SONAME TRUE
)

# Build the hardware interface library
add_library(${PROJECT_NAME} SHARED
  src/system_interface.cpp
)

target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)

ament_target_dependencies(${PROJECT_NAME}
  hardware_interface
  pluginlib
  rclcpp
  rclcpp_lifecycle
  std_srvs
  delto_tcp_comm
)

target_link_libraries(${PROJECT_NAME}
  delto_gripper_helper
  Boost::system
)

# delto_gripper_helper installs alongside this library in lib/, so point the
# runtime loader there via $ORIGIN instead of relying on the (post-install
# deleted) source-tree path baked in at build time.
set_target_properties(${PROJECT_NAME} PROPERTIES
  INSTALL_RPATH "$ORIGIN"
)

# Export hardware plugins
pluginlib_export_plugin_description_file(hardware_interface delto_hardware.xml)

# Install
install(
  TARGETS ${PROJECT_NAME}
  EXPORT export_${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(
  DIRECTORY include/
  DESTINATION include
)

install(
  FILES delto_hardware.xml
  DESTINATION share/${PROJECT_NAME}
)

install(
  FILES ${GRIPPER_HELPER_LIB}
  DESTINATION lib
)

ament_export_include_directories(include)
ament_export_libraries(${PROJECT_NAME})
ament_export_targets(export_${PROJECT_NAME})
ament_export_dependencies(
  hardware_interface
  pluginlib
  rclcpp
  rclcpp_lifecycle
  std_srvs
  delto_tcp_comm
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
