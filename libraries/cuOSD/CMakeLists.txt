cmake_minimum_required(VERSION 3.10)

project(cuosd VERSION 0.1 LANGUAGES CXX CUDA)

set(CMAKE_CXX_STANDARD 17)

find_package(CUDA REQUIRED)


# setup CUDA
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/utils/cuda" )
message("-- CUDA version: ${CUDA_VERSION}")

set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -O3)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64")
	message("-- CUDA ${CUDA_VERSION} detected (${CMAKE_SYSTEM_PROCESSOR}), enabling SM_53 SM_62")
	set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_53,code=sm_53 -gencode arch=compute_62,code=sm_62)

	if(CUDA_VERSION_MAJOR GREATER 9)
		message("-- CUDA ${CUDA_VERSION} detected (${CMAKE_SYSTEM_PROCESSOR}), enabling SM_72")
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_72,code=sm_72)
	endif()

	if(CUDA_VERSION_MAJOR GREATER 10)
		message("-- CUDA ${CUDA_VERSION} detected (${CMAKE_SYSTEM_PROCESSOR}), enabling SM_87")
		set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -gencode arch=compute_87,code=sm_87)
	endif()
endif()

option(BUILD_PANGO_BACKEND "Build with Pango backend" OFF)
option(BUILD_STB_BACKEND "Build with stb backend" ON)

set(CUOSD_SOURCES
    src/cuosd.cpp
    src/cuosd_kernel.cu
    src/textbackend/backend.cpp
    src/textbackend/pango-cairo.cpp
    src/textbackend/stb.cpp
)

include_directories(
    ${CUDA_INCLUDE_DIRS}  
    ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}
    src
    src/textbackend
)

#select include files
file(GLOB ProjectIncludes src/*.h  src/textbackend/*.h)

if(BUILD_PANGO_BACKEND)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(PANGO REQUIRED IMPORTED_TARGET pango pangocairo)
    add_definitions(-DENABLE_TEXT_BACKEND_PANGO)
endif()

if(BUILD_STB_BACKEND)
    add_definitions(-DENABLE_TEXT_BACKEND_STB)
endif()

cuda_add_library(cuosd SHARED ${CUOSD_SOURCES})

target_include_directories(cuosd PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<INSTALL_INTERFACE:include>
)

if(BUILD_PANGO_BACKEND)
    target_link_libraries(cuosd PUBLIC
        PkgConfig::PANGO
    )
endif()

# install
# transfer all headers to the include directory
# file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR}/${PROJECT_NAME})

# foreach(include ${ProjectIncludes})
# 	message("-- Copying ${include}")
# 	configure_file(${include} ${PROJECT_INCLUDE_DIR}/${PROJECT_NAME} COPYONLY)
# endforeach()

# set the version of the target library
set_target_properties(${PROJECT_NAME} PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

# install the shared library
install(TARGETS ${PROJECT_NAME} DESTINATION lib EXPORT ${PROJECT_NAME}Config)

# install cusod.h to the include directory
install(FILES src/cuosd.h DESTINATION include)

# install the cmake project, for importing
install(EXPORT ${PROJECT_NAME}Config DESTINATION share/${PROJECT_NAME}/cmake)

# install the headers
# install(DIRECTORY ${PROJECT_INCLUDE_DIR}/${PROJECT_NAME} DESTINATION include)
