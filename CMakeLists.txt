cmake_minimum_required(VERSION 3.12)

# Global Policy for modern CMake 4.x compatibility
if(POLICY CMP0000)
    set(CMAKE_POLICY_VERSION_MINIMUM 3.5)
endif()

# --- Helper Functions ---
function(COPY_FROM_TO_DIR src_dir dst_dir)
    file(GLOB src_files ${src_dir}/*)
    foreach(src_file IN LISTS src_files)
        get_filename_component(file_and_ext ${src_file} NAME)
        set(dst_file ${dst_dir}/${file_and_ext})
        add_custom_command(
            TARGET ${PROJECT_NAME} PRE_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy ${src_file} ${dst_file}
        )
    endforeach()
endfunction()

macro(GroupSources curdir)
    if(MSVC)
        file(GLOB children RELATIVE ${PROJECT_SOURCE_DIR}/${curdir} ${PROJECT_SOURCE_DIR}/${curdir}/*)
        foreach(child ${children})
            if(IS_DIRECTORY ${PROJECT_SOURCE_DIR}/${curdir}/${child})
                GroupSources(${curdir}/${child})
            else()
                string(REPLACE "/" "\\" groupname ${curdir})
                source_group(${groupname} FILES ${PROJECT_SOURCE_DIR}/${curdir}/${child})
            endif()
        endforeach()
    endif()
endmacro()

# --- Project Setup ---
set(DIR_ROOT       ${CMAKE_CURRENT_LIST_DIR})
set(DIR_SOURCES    "${DIR_ROOT}/src")

project (ComputerGraphics CXX)

find_package(OpenGL REQUIRED)
if(NOT TARGET OpenGL::GLU)
    message(FATAL_ERROR "GLU could not be found")
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

# --- Dependency Management ---
if(UNIX)
    # Use Arch Linux system packages
    find_package(SDL2 REQUIRED)
    find_package(GLEW REQUIRED)
    
    # Map system targets to the framework's expected names
    if(NOT TARGET SDL2)
        add_library(SDL2 INTERFACE)
        target_link_libraries(SDL2 INTERFACE SDL2::SDL2)
    endif()
    
    if(NOT TARGET libglew_static)
        add_library(libglew_static INTERFACE)
        target_link_libraries(libglew_static INTERFACE GLEW::GLEW)
    endif()

    if(NOT TARGET SDL2main)
        add_library(SDL2main INTERFACE)
    endif()
else()
    # Windows: Build from source as originally intended
    add_subdirectory(libraries/sdl2 EXCLUDE_FROM_ALL)
    add_subdirectory(libraries/glew-cmake EXCLUDE_FROM_ALL)
endif()

# --- Source Collection ---
macro(CG_FILES_APPEND)
    file(GLOB FILES_APPEND CONFIGURE_DEPENDS ${ARGV})
    list(APPEND CG_SOURCES ${FILES_APPEND})
endmacro()

macro(CG_SOURCES_APPEND)
    CG_FILES_APPEND(${ARGV0}/*.h)
    CG_FILES_APPEND(${ARGV0}/*.cpp)
endmacro()

CG_SOURCES_APPEND(${DIR_SOURCES}/extra)
CG_SOURCES_APPEND(${DIR_SOURCES}/framework)
CG_SOURCES_APPEND(${DIR_SOURCES}/main)

# --- Target Definition ---
add_executable(ComputerGraphics ${CG_SOURCES})

target_include_directories(ComputerGraphics PUBLIC ${DIR_SOURCES})
set_property(TARGET ComputerGraphics PROPERTY CXX_STANDARD 11)
set_property(TARGET ComputerGraphics PROPERTY CXX_STANDARD_REQUIRED ON)

GroupSources(src)

# --- Platform Definitions & Linking ---
if (NOT UNIX)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        add_definitions(-D_AMD64_)
    elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
        add_definitions(-D_X86_)
    endif()
endif()

# Linking SDL2
target_link_libraries(ComputerGraphics PRIVATE SDL2)
if(NOT UNIX)
    target_link_libraries(ComputerGraphics PRIVATE SDL2main)
    target_include_directories(ComputerGraphics PRIVATE libraries/sdl2/include)
endif()

# Linking GLEW & OpenGL
add_definitions(-DGLEW_STATIC)
target_link_libraries(ComputerGraphics PRIVATE libglew_static)
target_link_libraries(ComputerGraphics PRIVATE OpenGL::GL OpenGL::GLU)

# Folder Properties (MSVC Only)
if(NOT UNIX)
    set_property(TARGET SDL2 PROPERTY FOLDER "External/SDL2")
    set_property(TARGET SDL2main PROPERTY FOLDER "External/SDL2")
    set_property(TARGET libglew_static PROPERTY FOLDER "External/libglew_static")
endif()