SET(PROJECT_NAME DSO)

PROJECT(${PROJECT_NAME})
CMAKE_MINIMUM_REQUIRED(VERSION 3.1)
#set(CMAKE_VERBOSE_MAKEFILE ON)


set(BUILD_TYPE Release)
#set(BUILD_TYPE RelWithDebInfo)

set(EXECUTABLE_OUTPUT_PATH bin)
set(LIBRARY_OUTPUT_PATH lib)
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

# required libraries

# optional libraries
find_package(LibZip REQUIRED)
find_package(libjpeg-turbo REQUIRED)
find_package(Eigen3 REQUIRED CONFIG)
find_package(Pangolin REQUIRED CONFIG)
find_package(OpenCV REQUIRED)


# Sources files
set(dso_SOURCE_FILES
  ${PROJECT_SOURCE_DIR}/src/FullSystem/FullSystem.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/FullSystem.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/FullSystemOptimize.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/FullSystemOptPoint.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/FullSystemDebugStuff.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/FullSystemMarginalize.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/Residuals.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/Residuals.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/CoarseTracker.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/CoarseTracker.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/CoarseInitializer.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/CoarseInitializer.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/ImmaturePoint.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/ImmaturePoint.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/HessianBlocks.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/HessianBlocks.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/PixelSelector2.cpp
  ${PROJECT_SOURCE_DIR}/src/FullSystem/PixelSelector.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/PixelSelector2.h
  ${PROJECT_SOURCE_DIR}/src/FullSystem/ResidualProjections.h
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/EnergyFunctional.cpp
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/EnergyFunctional.h
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/AccumulatedTopHessian.cpp
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/AccumulatedTopHessian.h
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/AccumulatedSCHessian.cpp
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/AccumulatedSCHessian.h
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/EnergyFunctionalStructs.cpp
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/EnergyFunctionalStructs.h
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/MatrixAccumulators.h
  ${PROJECT_SOURCE_DIR}/src/OptimizationBackend/RawResidualJacobian.h
  ${PROJECT_SOURCE_DIR}/src/util/settings.cpp
  ${PROJECT_SOURCE_DIR}/src/util/settings.h
  ${PROJECT_SOURCE_DIR}/src/util/DatasetReader.h
  ${PROJECT_SOURCE_DIR}/src/util/FrameShell.h
  ${PROJECT_SOURCE_DIR}/src/util/globalFuncs.h
  ${PROJECT_SOURCE_DIR}/src/util/ImageAndExposure.h
  ${PROJECT_SOURCE_DIR}/src/util/IndexThreadReduce.h
  ${PROJECT_SOURCE_DIR}/src/util/MinimalImage.h
  ${PROJECT_SOURCE_DIR}/src/util/nanoflann.h
  ${PROJECT_SOURCE_DIR}/src/util/NumType.h
  ${PROJECT_SOURCE_DIR}/src/util/Undistort.cpp
  ${PROJECT_SOURCE_DIR}/src/util/Undistort.h
  ${PROJECT_SOURCE_DIR}/src/util/globalCalib.cpp
  ${PROJECT_SOURCE_DIR}/src/util/globalCalib.h
)


# decide if we have pangolin
if (Pangolin_FOUND)
	message("--- found PANGOLIN, compiling dso_pangolin library.")
	#include_directories( ${Pangolin_INCLUDE_DIRS} ) 
	set(dso_pangolin_SOURCE_FILES 
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/Pangolin/KeyFrameDisplay.cpp
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/Pangolin/KeyFrameDisplay.h
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/Pangolin/PangolinDSOViewer.cpp
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/Pangolin/PangolinDSOViewer.h)
	set(HAS_PANGOLIN 1)
else ()
	message("--- could not find PANGOLIN, not compiling dso_pangolin library.")
	message("    this means there will be no 3D display / GUI available for dso_dataset.")
	set(dso_pangolin_SOURCE_FILES )
	set(HAS_PANGOLIN 0)
endif ()


# compile main library.
#find_package(csparse REQUIRED)
#include_directories( ${CSPARSE_INCLUDE_DIR} ${CHOLMOD_INCLUDE_DIR}) 
add_library(dso ${dso_SOURCE_FILES} ${dso_opencv_SOURCE_FILES} ${dso_pangolin_SOURCE_FILES})
target_compile_definitions(dso PUBLIC -DENABLE_SSE)
if (MSVC)
    target_compile_options(dso PUBLIC /EHsc /MP)
elseif(CMAKE_COMPILER_IS_GNUCXX )
    target_compile_options(dso PUBLIC "${SSE_FLAGS} -O3 -g -std=c++0x -march=native")
endif()

target_link_libraries(dso PUBLIC ::Eigen3)
target_include_directories(dso PUBLIC 
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/thirdparty/Sophus>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/thirdparty/sse2neon>
)


# decide if we have ziplib.
if (LIBZIP_LIBRARY)
	message("--- found ziplib (${LIBZIP_VERSION}), compiling with zip capability.")
	target_compile_definitions(dso PUBLIC -DHAS_ZIPLIB=1)
	target_include_directories(dso PUBLIC ${LIBZIP_INCLUDE_DIR_ZIP} ${LIBZIP_INCLUDE_DIR_ZIPCONF}) 
else()
	message("--- not found ziplib (${LIBZIP_LIBRARY}), compiling without zip capability.")
	set(LIBZIP_LIBRARY "")
endif()

# decide if we have openCV
if (OpenCV_FOUND)
	message("--- found OpenCV, compiling dso_opencv library. ${OpenCV_INCLUDE_DIRS}")
    target_include_directories(dso PUBLIC ${OpenCV_INCLUDE_DIRS}) 
	target_sources(dso PRIVATE
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/OpenCV/ImageDisplay_OpenCV.cpp
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/OpenCV/ImageRW_OpenCV.cpp)
else ()
	message("--- could not find OpenCV, not compiling dso_opencv library.")
	message("    this means there will be no image display, and image read / load functionality.")
	target_sources(dso 
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/ImageDisplay_dummy.cpp
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/ImageRW_dummy.cpp
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/ImageRW.h
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/Output3DWrapper.h
	  ${PROJECT_SOURCE_DIR}/src/IOWrapper/ImageDisplay.h)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin") # OSX
    set(BOOST_THREAD_LIBRARY boost_thread-mt)
else()
    set(BOOST_THREAD_LIBRARY boost_thread)
endif()

# build main executable (only if we have both OpenCV and Pangolin)
if (OpenCV_FOUND AND Pangolin_FOUND)
	message("--- compiling dso_dataset.")
	add_executable(dso_dataset ${PROJECT_SOURCE_DIR}/src/main_dso_pangolin.cpp )
    target_link_libraries(dso_dataset dso ${LIBZIP_LIBRARY}  ${OpenCV_LIBRARIES})
    target_link_libraries(dso_dataset ::pangolin)
    target_link_libraries(dso PUBLIC ::pangolin)
else()
	message("--- not building dso_dataset, since either don't have openCV or Pangolin.")
endif()

