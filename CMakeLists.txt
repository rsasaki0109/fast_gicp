cmake_minimum_required(VERSION 3.10.0)
project(fast_gicp_ros2)

add_definitions(-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2)
set(CMAKE_C_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")
set(CMAKE_CXX_FLAGS "-msse -msse2 -msse3 -msse4 -msse4.1 -msse4.2")

set(CMAKE_BUILD_TYPE "Release")

#find_package(catkin REQUIRED)
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)

find_package(PCL REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(Boost REQUIRED)

find_package(OpenMP)
if (OPENMP_FOUND)
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()


find_package(CUDA)
if(CUDA_FOUND)
  include_directories(${CUDA_INCLUDE_DIRS})
  link_directories(${CUDA_LIBRARY_DIRS})
endif()

###################################
## catkin specific configuration ##
###################################
#catkin_package(
#  INCLUDE_DIRS include thirdparty/Sophus
#  LIBRARIES fast_gicp
#  CATKIN_DEPENDS other_catkin_pkg
#  DEPENDS system_lib
#)

###########
## Build ##
###########
include_directories(
  include
)

add_library(fast_gicp
  src/fast_gicp/gicp/fast_gicp.cpp
  src/fast_gicp/gicp/fast_gicp_st.cpp
  src/fast_gicp/gicp/fast_vgicp.cpp
  src/fast_gicp/gicp/experimental/fast_gicp_mp.cpp
)

target_link_libraries(fast_gicp
  ${PCL_LIBRARIES}
)
target_include_directories(fast_gicp PUBLIC
  include
  thirdparty/Sophus
  ${PCL_INCLUDE_DIRS}
  ${EIGEN3_INCLUDE_DIR}
  ${catkin_INCLUDE_DIRS}
)

add_executable(gicp_align src/align.cpp)
#add_dependencies(gicp_align ${${PROJECT_NAME}_EXPORTED_TARGETS} 
#${catkin_EXPORTED_TARGETS}
#)
target_link_libraries(gicp_align
  #${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  fast_gicp
)


if(CUDA_FOUND)
  set(CUDA_NVCC_FLAGS "--expt-relaxed-constexpr")
  add_definitions(-DUSE_VGICP_CUDA)

  cuda_add_library(fast_vgicp_cuda
    src/fast_gicp/cuda/fast_vgicp_cuda.cu
  )
  target_include_directories(fast_vgicp_cuda PRIVATE
    include
    thirdparty/Sophus
    thirdparty/Eigen
    thirdparty/nvbio
  )
  target_link_libraries(fast_vgicp_cuda
    cublas
  )

  # add vgicp_cuda to libfast_gicp
  target_sources(fast_gicp PRIVATE
    src/fast_gicp/gicp/fast_vgicp_cuda.cpp
  )
  target_link_libraries(fast_gicp
    fast_vgicp_cuda
  )
  add_dependencies(fast_gicp fast_vgicp_cuda)
endif()


############
## INSTAL ##
############

install(TARGETS 
  gicp_align
  DESTINATION lib/${PROJECT_NAME}
)

install(
  TARGETS
    gicp_align
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(
  DIRECTORY "include/"
  DESTINATION include
)

ament_export_dependencies(rclcpp)
ament_export_include_directories(include)

ament_package()