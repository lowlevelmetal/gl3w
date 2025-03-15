cmake_minimum_required(VERSION 3.1.3 FATAL_ERROR)

project(gl3w)

set(CMAKE_VERBOSE_MAKEFILE false)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR}/src)
set(HEADER_DIR ${CMAKE_CURRENT_BINARY_DIR}/include)

set(HEADER_FILES
    "${HEADER_DIR}/GL/gl3w.h"
    "${HEADER_DIR}/GL/glcorearb.h"
    "${HEADER_DIR}/KHR/khrplatform.h"
)

set(SOURCE_FILES
    "${SOURCE_DIR}/gl3w.c"
)

find_package(OpenGL REQUIRED)
find_package(PythonInterp REQUIRED)

if(CMAKE_VERSION VERSION_LESS 3.8.0)
    set(EXTERNAL_INCLUDE_DIRS ${EXTERNAL_INCLUDE_DIRS} ${OPENGL_INCLUDE_DIR})
    set(EXTERNAL_LIBRARIES ${EXTERNAL_LIBRARIES} ${OPENGL_LIBRARIES})
else()
    set(EXTERNAL_LIBRARIES ${EXTERNAL_LIBRARIES} OpenGL::GL OpenGL::GLU)
endif()

add_custom_command(
    OUTPUT ${SOURCE_FILES} ${HEADER_FILES}
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/gl3w_gen.py
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/gl3w_gen.py
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

add_custom_target(gl3w_gen ALL DEPENDS ${SOURCE_FILES} ${HEADER_FILES})

# Define the library target
add_library(${PROJECT_NAME} STATIC ${SOURCE_FILES})

# Mark generated files explicitly
set_source_files_properties(${SOURCE_FILES} PROPERTIES GENERATED TRUE)

# Ensure source files are generated first
add_dependencies(${PROJECT_NAME} gl3w_gen)

target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${HEADER_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>
)

if(DEFINED EXTERNAL_INCLUDE_DIRS)
    target_include_directories(${PROJECT_NAME} PUBLIC ${EXTERNAL_INCLUDE_DIRS})
endif()

target_link_libraries(${PROJECT_NAME} PUBLIC ${EXTERNAL_LIBRARIES} ${CMAKE_DL_LIBS})

include(GNUInstallDirs)

# Install headers
foreach(HEADER_FILE ${HEADER_FILES})
    file(RELATIVE_PATH HEADER_PATH "${HEADER_DIR}" "${HEADER_FILE}")
    get_filename_component(HEADER_PATH ${HEADER_PATH} PATH)
    install(FILES "${HEADER_FILE}"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}/${HEADER_PATH}")
endforeach()

# Include directories
target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${HEADER_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>
)

# Installation rules (targets)
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}-targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

set(MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
set(BUILD_CMAKE_DIR "${CMAKE_CURRENT_BINARY_DIR}/cmake")

# Export targets for find_package
configure_file(
    "${MODULE_PATH}/${PROJECT_NAME}-config.cmake"
    "${BUILD_CMAKE_DIR}/${PROJECT_NAME}-config.cmake"
    COPYONLY
)

export(
    TARGETS ${PROJECT_NAME}
    FILE "${BUILD_CMAKE_DIR}/${PROJECT_NAME}-targets.cmake"
)

export(PACKAGE ${PROJECT_NAME})
