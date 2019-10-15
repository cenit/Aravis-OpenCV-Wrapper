#Copyright 2019, Stefano Sinigardi

cmake_minimum_required(VERSION 3.10)

set(GIGE_OCV_WRAPPER_MAJOR_VERSION 0)
set(GIGE_OCV_WRAPPER_MINOR_VERSION 0)
set(GIGE_OCV_WRAPPER_PATCH_VERSION 1)
set(GIGE_OCV_WRAPPER_VERSION ${GIGE_OCV_WRAPPER_MAJOR_VERSION}.${GIGE_OCV_WRAPPER_MINOR_VERSION}.${GIGE_OCV_WRAPPER_PATCH_VERSION})

set(CMAKE_VERBOSE_MAKEFILE "FALSE" CACHE BOOL "Create verbose makefile")
set(BUILD_SHARED_LIBS "TRUE" CACHE BOOL "Create GIGE_OCV_WRAPPER as a shared library")
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules/" ${CMAKE_MODULE_PATH})

if(DEFINED ENV{VCPKG_ROOT} AND NOT DEFINED CMAKE_TOOLCHAIN_FILE)
  set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
  message(STATUS "VCPKG found: $ENV{VCPKG_ROOT}")
  message(STATUS "Using VCPKG integration")
elseif(DEFINED ENV{WORKSPACE} AND NOT DEFINED CMAKE_TOOLCHAIN_FILE)
  set(CMAKE_TOOLCHAIN_FILE "$ENV{WORKSPACE}/vcpkg_opencv4/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
  message(STATUS "VCPKG found: $ENV{WORKSPACE}/vcpkg")
  message(STATUS "Using VCPKG integration")
endif()

project(GIGE_OCV_WRAPPER VERSION ${GIGE_OCV_WRAPPER_VERSION})

enable_language(C)
enable_language(CXX)

set(CMAKE_CXX_STANDARD 11)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_LIST_DIR}" CACHE PATH "Install prefix" FORCE)
endif()

set(INSTALL_BIN_DIR      "bin"              CACHE PATH "Path where exe and dll will be installed")
set(INSTALL_LIB_DIR      "lib"              CACHE PATH "Path where lib will be installed")
set(INSTALL_INCLUDE_DIR  "include"          CACHE PATH "Path where headers will be installed (we will explicitly include the subfolder ${PROJECT_NAME}")
set(INSTALL_CMAKE_DIR    "share/gige-ocv"   CACHE PATH "Path where cmake configs will be installed")

set(CMAKE_INSTALL_RPATH "${CMAKE_CURRENT_LIST_DIR}/${INSTALL_LIB_DIR}")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

set(CMAKE_DEBUG_POSTFIX d)

find_library(LIBGIO2 gio-2.0)
if(NOT LIBGIO2)
  message(FATAL_ERROR "Unable to find libgio-2.0")
endif()

find_library(LIBGLIB2 glib-2.0)
if(NOT LIBGLIB2)
  message(FATAL_ERROR "Unable to find libglib-2.0")
endif()

find_library(LIBGMODULE2 gmodule-2.0)
if(NOT LIBGMODULE2)
  message(FATAL_ERROR "Unable to find libgmodule-2.0")
endif()

find_library(LIBGOBJECT2 gobject-2.0)
if(NOT LIBGOBJECT2)
  message(FATAL_ERROR "Unable to find libgobject-2.0")
endif()

find_library(LIBGTHREAD2 gthread-2.0)
if(NOT LIBGTHREAD2)
  message(FATAL_ERROR "Unable to find libgthread-2.0")
endif()

find_library(LIBINOTIFY2 inotify-2.0)
if(NOT LIBINOTIFY2)
  message(FATAL_ERROR "Unable to find libinotify-2.0")
endif()

find_library(LIBXDGMIME2 xdgmime-2.0)
if(NOT LIBXDGMIME2)
  message(FATAL_ERROR "Unable to find libxdgmime-2.0")
endif()

find_library(LIBRESOLV resolv)
if(NOT LIBRESOLV)
  message(FATAL_ERROR "Unable to find libresolv")
endif()

find_library(LIBPCRE pcre)
if(NOT LIBPCRE)
  message(FATAL_ERROR "Unable to find libpcre")
endif()


find_library(LIBM m)

find_package(Aravis REQUIRED)
find_package(Threads REQUIRED)
find_package(LibXml2 REQUIRED)
find_package(ZLIB REQUIRED)
find_package(PNG REQUIRED)
find_package(libffi REQUIRED)
find_package(OpenCV 3.4.7 EXACT REQUIRED)

if(WIN32)
  if(TARGET opencv_cudaoptflow)
    list(APPEND OpenCV_LINKED_COMPONENTS "opencv_cudaoptflow")
  endif()
  if(TARGET opencv_cudaimgproc)
    list(APPEND OpenCV_LINKED_COMPONENTS "opencv_cudaimgproc")
  endif()
  if(TARGET opencv_world)
    list(APPEND OpenCV_LINKED_COMPONENTS "opencv_world")
  else()
    if(TARGET opencv_video)
      list(APPEND OpenCV_LINKED_COMPONENTS "opencv_video")
    endif()
    if(TARGET opencv_videoio)
      list(APPEND OpenCV_LINKED_COMPONENTS "opencv_videoio")
    endif()
    if(TARGET opencv_highgui)
      list(APPEND OpenCV_LINKED_COMPONENTS "opencv_highgui")
    endif()
    if(TARGET opencv_imgproc)
      list(APPEND OpenCV_LINKED_COMPONENTS "opencv_imgproc")
    endif()
    if(TARGET opencv_imgcodecs)
      list(APPEND OpenCV_LINKED_COMPONENTS "opencv_imgcodecs")
    endif()
    if(TARGET opencv_core)
      list(APPEND OpenCV_LINKED_COMPONENTS "opencv_core")
    endif()
  endif()
else()
  list(APPEND OpenCV_LINKED_COMPONENTS ${OpenCV_LIBS})
endif()

add_library(Camera
  ${CMAKE_CURRENT_LIST_DIR}/src/Camera.cpp
  ${CMAKE_CURRENT_LIST_DIR}/src/Camera.hpp
)
set_target_properties(Camera PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
target_include_directories(Camera PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src> $<BUILD_INTERFACE:${Aravis_INCLUDE_DIR}>)
target_include_directories(Camera PRIVATE $<BUILD_INTERFACE:${LIBXML2_INCLUDE_DIR}>)
target_link_libraries(Camera PUBLIC  ${Aravis_LIBRARY} ${LIBXML2_LIBRARIES} ${OpenCV_LINKED_COMPONENTS} ZLIB::ZLIB PNG::PNG Threads::Threads ${GLIB2} ${LIBM} ${LIBGIO2} ${LIBGOBJECT2} ${LIBGTHREAD2} ${LIBXDGMIME2} ${LIBINOTIFY2} ${LIBGMODULE2} ${LIBRESOLV} ${LIBPCRE} libffi)

add_executable(Example_main
  ${CMAKE_CURRENT_LIST_DIR}/examples/main.cpp
)
target_link_libraries(Example_main PRIVATE Camera)

add_executable(Example_trigger
  ${CMAKE_CURRENT_LIST_DIR}/examples/trigger.cpp
)
target_link_libraries(Example_trigger PRIVATE Camera)

install(TARGETS Example_main Example_trigger DESTINATION "${INSTALL_BIN_DIR}")

install(TARGETS Camera EXPORT CameraTargets
  RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
  PUBLIC_HEADER DESTINATION "${INSTALL_INCLUDE_DIR}"
  COMPONENT dev
)
install(EXPORT CameraTargets
  FILE CameraTargets.cmake
  NAMESPACE Camera::
  DESTINATION "${INSTALL_CMAKE_DIR}"
)
