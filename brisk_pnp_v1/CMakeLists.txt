cmake_minimum_required(VERSION 2.8.11)
include(CheckIncludeFiles)

project(brisk_pnp CXX)

find_package(roscpp)
set(INCLUDE_DIRS ${INCLUDE_DIRS} ${roscpp_INCLUDE_DIRS})
set(LIBRARIES ${LIBRARIES} ${roscpp_LIBRARIES})

set(GCC_COVERAGE_COMPILE_FLAGS "-Wno-reorder -Wno-sign-compare -Wno-unused-value -Wno-unused-variable -Wno-unused-but-set-variable -Wall -Wno-comment")
add_definitions(${GCC_COVERAGE_COMPILE_FLAGS})

set(CMAKE_CXX_FLAGS   "${CMAKE_CXX_FLAGS} -std=c++11 -pthread")
set(CMAKE_CXX_FLAGS   "${CMAKE_CXX_FLAGS} -march=native")
set(CMAKE_BUILD_TYPE  Release)
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/modules")
set(INCLUDE           "${CMAKE_SOURCE_DIR}/include")

# The version number
set(BRISK_MAJOR_VERSION 2)
set(BRISK_MINOR_VERSION 0)
set(BRISK_PATCH_VERSION 2)
set(BRISK_VERSION
        ${BRISK_MAJOR_VERSION}.${BRISK_MINOR_VERSION}.${BRISK_PATCH_VERSION})

set(FREENECT2_INCLUDE_DIRS "~/freenect2/include")
set(FREENECT2_LIBRARY "~/freenect2/lib/libfreenect2.so")

# Offer the user the choice of overriding the installation directories
set(INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set(INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDE_DIR include CACHE PATH
  "Installation directory for header files")
if(WIN32 AND NOT CYGWIN)
  set(DEF_INSTALL_CMAKE_DIR CMake)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/CMake/brisk)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH
  "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE)
  set(var INSTALL_${p}_DIR)
  if(NOT IS_ABSOLUTE "${${var}}")
    set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
  endif()
endforeach()

# make sure we use Release and warn otherwise
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

if (NOT "${CMAKE_BUILD_TYPE}" STREQUAL "Release")
  message(WARNING "CMAKE_BUILD_TYPE not set to 'Release'. Performance may be terrible.")
else()
  message(STATUS "Building with build type '${CMAKE_BUILD_TYPE}'")
endif()

# require OpenCV
find_package( OpenCV REQUIRED )
include_directories(BEFORE ${OpenCV_INCLUDE_DIRS})
find_package(Freenect2  REQUIRED)
include_directories(${FREENECT2_INCLUDE_DIRS})
include_directories(${INCLUDE})
#find_package( OpenGV REQUIRED)

if(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "arm*")
  message(STATUS "ARM processor detected, will attempt to use NEON.")
  add_definitions(-mfpu=neon -DHAVE_OPENCV -Wall -pedantic -std=c++0x -fPIC)
else()
  message(STATUS "Assuming SSE instructions available.")
  add_definitions(-mssse3 -DHAVE_OPENCV -Wall -pedantic -std=c++0x -fPIC)
endif()

# set up the output tree
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# check include file safely
get_property(dirs DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
foreach(dir ${dirs})
  if(EXISTS "${dir}/opencv2/nonfree/nonfree.hpp")
    set(HAVE_OPENCV_NONFREE_H TRUE)
    #message(STATUS "Found ${dir}/opencv2/nonfree/nonfree.hpp, setting HAVE_OPENCV_NONFREE_H=TRUE")
  endif()
endforeach()
if(${OpenCV_VERSION_MAJOR} STREQUAL 3)
  message(WARNING "Nonfree module disabled, demo application with limited functionality")
endif()

if(NOT APPLE)
  # The clang compiler (on osx) is somehow much more strict
  # than the compilers on ubuntu and so this does not seem
  # possible on OSX just yet.
  add_definitions( -Werror )
endif()

option(WITH_PCL "adds pcl cloud support" ON)
if(${WITH_PCL})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DWITH_PCL")
    find_package(PCL REQUIRED)
    link_directories(${PCL_LIBRARY_DIRS})
    add_definitions(${PCL_DEFINITIONS})
    include_directories(${PCL_INCLUDE_DIRS})
endif()

# the absolute path to the demo images
set(BRISK_IMAGE_PATH \"${PROJECT_SOURCE_DIR}/images\")
add_definitions(-DBRISK_IMAGE_PATH=${BRISK_IMAGE_PATH})

# build agast
include_directories(agast/include)
include_directories(include)
add_subdirectory(agast)

# build brisk
add_library(${PROJECT_NAME} STATIC
                            src/brisk/brisk-descriptor-extractor.cc
                            src/brisk/brisk-feature-detector.cc
                            src/brisk/brisk-layer.cc
                            src/brisk/brisk-scale-space.cc
                            src/brisk/brute-force-matcher.cc
                            src/brisk/harris-feature-detector.cc
                            src/brisk/harris-score-calculator.cc
                            src/brisk/harris-score-calculator-float.cc
                            src/brisk/harris-scores.cc
                            src/brisk/image-down-sampling.cc
                            src/brisk/pattern-provider.cc
                            src/brisk/vectorized-filters.cc)

# and link it

target_link_libraries(${PROJECT_NAME} PUBLIC ${OpenCV_LIBS} agast)
add_executable(demo src/main.cpp src/RGB.cpp src/RGBD.cpp src/matcher.cpp src/EyeTracking_2D3D.cpp src/Utils.cpp src/Mesh.cpp)
target_link_libraries(demo ${PROJECT_NAME} ${FREENECT2_LIBRARY} ${PCL_LIBRARIES} ${BOOST_LIBS})

# installation if required
install(TARGETS ${PROJECT_NAME} EXPORT briskTargets ARCHIVE DESTINATION ${INSTALL_LIB_DIR})
install(TARGETS demo EXPORT briskTargets DESTINATION ${INSTALL_BIN_DIR})
install(DIRECTORY include/ DESTINATION ${INSTALL_INCLUDE_DIR} COMPONENT dev FILES_MATCHING PATTERN "*.h")

# installation is invoked in the individual modules...
export (TARGETS 
    brisk_pnp
    agast
    demo
    FILE "${PROJECT_BINARY_DIR}/briskTargets.cmake")
export ( PACKAGE brisk )

# Create the briskConfig.cmake and briskConfigVersion files
file(RELATIVE_PATH REL_INCLUDE_DIR "${INSTALL_CMAKE_DIR}"
   "${INSTALL_INCLUDE_DIR}")
# ... for the build tree
set(CONF_INCLUDE_DIRS 
  "${PROJECT_SOURCE_DIR}/include"
  "${PROJECT_BINARY_DIR}"
)
configure_file(cmake/briskConfig.cmake.in
  "${PROJECT_BINARY_DIR}/briskConfig.cmake" @ONLY)
# ... for the install tree
set(CONF_INCLUDE_DIRS "\${BRISK_CMAKE_DIR}/${REL_INCLUDE_DIR}")
configure_file(cmake/briskConfig.cmake.in
  "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/briskConfig.cmake" @ONLY)
# ... for both
configure_file(cmake/briskConfigVersion.cmake.in
  "${PROJECT_BINARY_DIR}/briskConfigVersion.cmake" @ONLY)
 
# Install the briskConfig.cmake and briskConfigVersion.cmake
install(FILES
  "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/briskConfig.cmake"
  "${PROJECT_BINARY_DIR}/briskConfigVersion.cmake"
  DESTINATION "${INSTALL_CMAKE_DIR}")
 
# Install the export set for use with the install-tree
install(EXPORT briskTargets DESTINATION
  "${INSTALL_CMAKE_DIR}")

