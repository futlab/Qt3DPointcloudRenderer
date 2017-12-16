cmake_minimum_required(VERSION 3.8.0)
project(Qt3DPointcloudRenderer VERSION 1.0.0)

set(CMAKE_AUTOMOC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
if(NOT CMAKE_DEBUG_POSTFIX)
  set(CMAKE_DEBUG_POSTFIX d)
endif()

find_package(Qt5 COMPONENTS Core Qml Widgets Quick 3DInput 3DQuick 3DRender 3DQuickRender REQUIRED)

set(WITH_PCL ON  CACHE BOOL "Enable pcl")
set(WITH_LAS OFF CACHE BOOL "Enable las")

if(WITH_PCL)
    find_package(PCL REQUIRED COMPONENTS common io)
    link_directories(${PCL_LIBRARY_DIRS})
endif(WITH_PCL)

if(WITH_LAS)
    # LibLAS is used for easy integration with windows. Thus it is not included
    # with an additional find-script, but quick-and-dirty by integrating LAS into
    # the the project. Retain libLAS copyright notice in your project (BSD License).

    set(HAVE_GDAL FALSE CACHE BOOL "Enable GDAL for LibLAS")
    set(HAVE_LIBGEOTIFF FALSE CACHE BOOL "Enable libGeoTIFF for LibLAS")
    set(HAVE_LASZIP TRUE CACHE BOOL "Enable Laszip for LibLAS")
    set(LIBLAS_PATH CACHE PATH "Path to libLAS folder, containing \"include\" and \"src\"")
    set(LIBLASZIP_PATH CACHE PATH "Path to LASzip folder VERSION v2.2.0! (newer versions are incompatible)")

    file(GLOB_RECURSE LASZIP_SOURCES ${LIBLASZIP_PATH}/include/*.hpp
                                     ${LIBLASZIP_PATH}/include/*.h
                                     ${LIBLASZIP_PATH}/src/*.hpp
                                     ${LIBLASZIP_PATH}/src/*.h
                                     ${LIBLAS_PATH}/src/*.cpp )

    file(GLOB_RECURSE LIBLAS_SOURCES ${LIBLAS_PATH}/include/*.hpp
                                     ${LIBLAS_PATH}/include/*.h
                                     ${LIBLAS_PATH}/src/*.hpp
                                     ${LIBLAS_PATH}/src/*.h
                                     ${LIBLAS_PATH}/src/*.cpp )
    if(NOT HAVE_GDAL)
      file(GLOB_RECURSE GT_SRC ${LIBLAS_PATH}/src/gt_*.cpp
                               ${LIBLAS_PATH}/src/gt_*.h)
      list(REMOVE_ITEM LIBLAS_SOURCES "${GT_SRC}")
    endif(NOT HAVE_GDAL)

    if(NOT HAVE_LIBGEOTIFF)
      file(GLOB_RECURSE GEO_SRC ${LIBLAS_PATH}/src/tifvsi.cpp)
      list(REMOVE_ITEM LIBLAS_SOURCES "${GEO_SRC}")
    endif(NOT HAVE_LIBGEOTIFF)
endif(WITH_LAS)

set(SOURCE
    ${CMAKE_CURRENT_SOURCE_DIR}/src/qpointcloud.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/qpointcloudgeometry.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/qpointfield.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/qpointcloudreader.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/src/xyzcsvreader.cpp
  )
set(HEADER
    ${CMAKE_CURRENT_SOURCE_DIR}/include/qpointcloud.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/qpointcloudgeometry.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/qpointfield.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/qpointcloudreader.h
#    ${CMAKE_CURRENT_SOURCE_DIR}/include/xyzcsvreader.h
  )

set(BUILD_EXAMPLE ON CACHE BOOL "Build the example project")

if(BUILD_EXAMPLE)
    add_subdirectory(example)
endif(BUILD_EXAMPLE)

set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

add_library(Qt3DPointcloudRenderer ${SOURCE} ${HEADER} ${LIBLAS_SOURCES} ${LASZIP_SOURCES})


target_include_directories(
    Qt3DPointcloudRenderer PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>"
    # "$<BUILD_INTERFACE:${generated_headers}>"
)

#target_include_directories(Qt3DPointcloudRenderer PUBLIC  ${PROJECT_SOURCE_DIR}/include/
#                                                  PRIVATE ${PROJECT_SOURCE_DIR}/src/ )

if(WITH_PCL)
    target_link_libraries(Qt3DPointcloudRenderer ${PCL_LIBRARIES})
    target_include_directories(${PROJECT_NAME} PRIVATE PRIVATE ${PCL_INCLUDE_DIRS})
    #target_compile_definitions(${PROJECT_NAME} PRIVATE ${PCL_DEFINITIONS}) causes command-line error in cmake from ubuntu 16.04
    target_compile_definitions(${PROJECT_NAME} PUBLIC WITH_PCL=1)
endif(WITH_PCL)

if(WITH_LAS)
    target_include_directories(${PROJECT_NAME} PRIVATE ${LIBLAS_PATH}/include/)
    target_compile_definitions(${PROJECT_NAME} PUBLIC WITH_LAS=1)
    if(HAVE_LASZIP)
        target_include_directories(${PROJECT_NAME} PRIVATE ${LIBLASZIP_PATH}/include/
                                                           ${LIBLASZIP_PATH}/include/laszip
                                                           ${LIBLASZIP_PATH}/src/)
    endif(HAVE_LASZIP)
endif(WITH_LAS)

qt5_use_modules(Qt3DPointcloudRenderer Qml Widgets Quick 3DCore 3DQuick 3DRender 3DQuickRender)


################## Installation #################################################################################################################

#set(generated_headers "${CMAKE_CURRENT_BINARY_DIR}/generated_headers")
#set(Qt3DPointcloudRenderer_export "${generated_headers}/Qt3DPointcloudRenderer/qpointcloud.h")
#include(GenerateExportHeader)
#generate_export_header(Qt3DPointcloudRenderer EXPORT_FILE_NAME ${Qt3DPointcloudRenderer_export})


set(config_install_dir "lib/cmake/${PROJECT_NAME}") # Сюда устанавливается *Config.cmake
set(include_install_dir "include/${PROJECT_NAME}")




# Конфигурация

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")


#set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")

include(CMakePackageConfigHelpers)


export(PACKAGE Qt3DPointcloudRenderer)


# Create config files
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)


#target_include_directories(Qt3DPointcloudRenderer PUBLIC 
#	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>  
#    $<INSTALL_INTERFACE:include>
#)

#install(FILES include/qpointcloud.h
#	DESTINATION include
#)

install(TARGETS Qt3DPointcloudRenderer
	EXPORT "${TARGETS_EXPORT_NAME}"
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib/static
	INCLUDES DESTINATION "${include_install_dir}"
)

install( 
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Заголовочные файлы:
#   * Source/foo/Bar.hpp -> <prefix>/include/foo/Bar.hpp
#   * Source/foo/Baz.hpp -> <prefix>/include/foo/Baz.hpp
install(
    #DIRECTORY "include/"
    FILES ${HEADER}
    DESTINATION "${include_install_dir}"
    #FILES_MATCHING PATTERN "*.hpp"
)

install(
    EXPORT "${TARGETS_EXPORT_NAME}"
#    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
