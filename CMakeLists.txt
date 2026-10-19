cmake_minimum_required(VERSION 3.20)
project(spite_d)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ---------------------------------------------------------------------------
# A trimmed build of SPITE-D for running on a real robot.
#
# Compared with the research package this drops the roadmap builders,
# the ablation demo and the test suite -- none of which the live
# pipeline uses. What remains is perception, prediction, spans,
# certification, replanning and the three ROS nodes.
#
# Consequence: OMPL is not needed, because roadmaps are not generated
# here. One ships in demo_roadmap/.
# ---------------------------------------------------------------------------
find_package(ament_cmake REQUIRED)

# open-spite provides the certification geometry and AABB trees. It is
# compiled from source as part of this build, so CMake needs its source
# directory -- unlike CGAL and OpenCV, which are installed system-wide
# and found automatically. It is vendored as a submodule.
set(SPITE_D_OPEN_SPITE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/open-spite"
    CACHE PATH "Path to the open-spite source tree")

if(NOT EXISTS "${SPITE_D_OPEN_SPITE_DIR}/CMakeLists.txt")
  message(FATAL_ERROR
    "open-spite not found at ${SPITE_D_OPEN_SPITE_DIR}\n"
    "  The repository was cloned without its submodule. Run:\n"
    "    git submodule update --init --recursive")
endif()
add_subdirectory(${SPITE_D_OPEN_SPITE_DIR} open_spite_build EXCLUDE_FROM_ALL)

find_package(OpenCV REQUIRED COMPONENTS core imgproc)

# ---------------------------------------------------------------------------
# Core libraries (no ROS dependency)
# ---------------------------------------------------------------------------
add_library(spite_d_core INTERFACE)
target_include_directories(spite_d_core INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)

add_library(spite_d_perception
  src/perception/uv_detector.cpp
  src/perception/box_tracker.cpp)
target_link_libraries(spite_d_perception
  PUBLIC spite_d_core opencv_core opencv_imgproc)

add_library(spite_d_trajectory src/trajectory/constant_velocity_predictor.cpp)
target_link_libraries(spite_d_trajectory PUBLIC spite_d_core)

add_library(spite_d_planner
  src/planner/replanner.cpp
  src/planner/roadmap_io.cpp)
target_link_libraries(spite_d_planner PUBLIC spite_d_core)

add_library(spite_d_validity src/spite/validity_server.cpp)
target_link_libraries(spite_d_validity PUBLIC spite_d_core open_spite)

# The research contribution: spans freeze a prediction and its
# certification geometry, so a conforming obstacle costs a conformance
# check instead of a tree query. validity_node switches between this and
# the every-frame baseline on the `use_spans` parameter.
add_library(spite_d_dynamic_map
  src/trajectory/span.cpp
  src/trajectory/span_pipeline.cpp)
target_link_libraries(spite_d_dynamic_map
  PUBLIC spite_d_core spite_d_trajectory spite_d_validity)

# ---------------------------------------------------------------------------
# ROS interfaces and nodes
# ---------------------------------------------------------------------------
find_package(rosidl_default_generators REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(builtin_interfaces REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(visualization_msgs REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/ObstacleState.msg"
  "msg/ObstacleArray.msg"
  "msg/PredictedTrajectory.msg"
  "msg/PredictedTrajectoryArray.msg"
  DEPENDENCIES std_msgs geometry_msgs builtin_interfaces)

rosidl_get_typesupport_target(cpp_typesupport_target ${PROJECT_NAME}
                              rosidl_typesupport_cpp)

add_executable(perception_node src/nodes/perception_node.cpp)
target_link_libraries(perception_node PRIVATE
  spite_d_perception
  "${cpp_typesupport_target}"
  rclcpp::rclcpp
  ${sensor_msgs_TARGETS}
  ${visualization_msgs_TARGETS})

add_executable(prediction_node src/nodes/prediction_node.cpp)
target_link_libraries(prediction_node PRIVATE
  spite_d_trajectory
  "${cpp_typesupport_target}"
  rclcpp::rclcpp)

add_executable(validity_node src/nodes/validity_node.cpp)
target_link_libraries(validity_node PRIVATE
  spite_d_dynamic_map
  spite_d_validity
  spite_d_trajectory
  spite_d_planner
  "${cpp_typesupport_target}"
  rclcpp::rclcpp
  ${nav_msgs_TARGETS}
  ${geometry_msgs_TARGETS}
  ${builtin_interfaces_TARGETS}
  ${visualization_msgs_TARGETS})

install(TARGETS perception_node prediction_node validity_node
        DESTINATION lib/${PROJECT_NAME})

# ament sets BUILD_SHARED_LIBS=ON, so these build as .so and must be
# installed where setup.bash's LD_LIBRARY_PATH points -- including
# open-spite's own targets, whose install rules EXCLUDE_FROM_ALL skips.
# Without this the nodes link but fail to start.
install(TARGETS
          spite_d_perception spite_d_trajectory spite_d_planner
          spite_d_validity spite_d_dynamic_map
          open_spite open_spite_gdiam
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin)

install(DIRECTORY launch rviz demo_roadmap
        DESTINATION share/${PROJECT_NAME})
install(DIRECTORY include/ DESTINATION include)

ament_export_include_directories(include)
ament_package()
