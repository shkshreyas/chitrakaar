cmake_minimum_required(VERSION 3.13)

###############################################################################
###############################################################################
#
# This CMake file is provided to make using chitrakaar library as automated
# as possible. When pointed to chitrakaar folder using the variable
# chitrakaar_chitrakaar_PATH (defined below), it will build chitrakaar
# dependencies, set the include path, compile chitrakaar source, compile all
# files listed in LIST_OF_FILES (below) and link everything together.
# See `projects/README.md` in chitrakaar repository for more details.


## List of files to compile.
set( LIST_OF_FILES "main.cpp" )

# Set required C++ standard.
set( chitrakaar_CXX_STANDARD 17 )

# Absolute path to chitrakaar directory. Leave empty for console app.
set( chitrakaar_chitrakaar_PATH "${CMAKE_SOURCE_DIR}/../../" CACHE STRING "Path to chitrakaar")

# Should the project support imgui? Enter abs path to imgui or leave empty.
set( chitrakaar_IMGUI_PATH "" CACHE STRING "Path to imgui")

###############################################################################
###############################################################################


## Set our project name (use directory name)
get_filename_component(PROJECT_NAME ${CMAKE_CURRENT_SOURCE_DIR} NAME)
string(REPLACE " " "_" PROJECT_NAME ${PROJECT_NAME})
project(${PROJECT_NAME} CXX)



# Just for possible debugging:
message("Using CMake ${CMAKE_VERSION}")


set(CMAKE_CXX_STANDARD ${chitrakaar_CXX_STANDARD})
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)



include(GNUInstallDirs)

if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "No build type selected, default to Release")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type (default Release)" FORCE)
endif()


foreach (DIR ${PREFIX_PATH_CHECK})
    if (NOT EXISTS "${DIR}")
        message(FATAL_ERROR "CMAKE_PREFIX_PATH element doesn't exist: ${DIR}")
    endif ()
endforeach ()
message("CMAKE_PREFIX_PATH:${CMAKE_PREFIX_PATH}")





## Print out what compiler was identified and set compiler options
#  Set desired warning level
if (MSVC)
    # CMake<3.15 passed the /Wn argument by itself. Overriding with /W4
    # raises a warning. Remove whatever warning level is set first.
    string(REGEX REPLACE "/W[0-9] " "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4 /Zc:__cplusplus")
    if ("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Od")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2")
    endif()
    MESSAGE("MSVC compiler detected")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    MESSAGE("GCC/Clang/MinGW detected")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic-errors")
    if ("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")
    endif()
else()
    MESSAGE(FATAL_ERROR "ERROR: Unknown (and possibly unsupported) compiler. (CMAKE_CXX_COMPILER_ID: '${CMAKE_CXX_COMPILER_ID}')")
endif()


set(chitrakaar_UNSUPPORTED_GENERATOR FALSE)
if ("${CMAKE_GENERATOR}" STREQUAL "Unix Makefiles")
    if (WIN32 AND NOT MINGW)
        set(chitrakaar_UNSUPPORTED_GENERATOR TRUE)
    endif()
elseif ("${CMAKE_GENERATOR}" STREQUAL "Ninja")
    if (NOT WIN32 OR NOT "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
        set(chitrakaar_UNSUPPORTED_GENERATOR TRUE)
    endif()
endif()
if (chitrakaar_UNSUPPORTED_GENERATOR)
    MESSAGE(FATAL_ERROR "ERROR: You are attempting to use unsupported combination of build tools. Go to .vscode folder in your project's directory, open settings.json and read comments at the end of the file.")
endif()




# Add chitrakaar srcs to the project.
if (NOT "${chitrakaar_chitrakaar_PATH}" STREQUAL "" )
    set( LIST_OF_FILES "${LIST_OF_FILES};${chitrakaar_chitrakaar_PATH}/src/chitrakaar.cpp")
    message("The project uses chitrakaar library: ${chitrakaar_chitrakaar_PATH}")
else()
    message("The project does NOT use chitrakaar library")
endif()



# Create our target.
set(TARGET_NAME "${PROJECT_NAME}_executable")
add_executable(${TARGET_NAME} ${LIST_OF_FILES})
set_target_properties(${TARGET_NAME} PROPERTIES OUTPUT_NAME ${PROJECT_NAME})





if (NOT "${chitrakaar_chitrakaar_PATH}" STREQUAL "" )
    # First add chitrakaar include directory.
    target_include_directories(${TARGET_NAME} PRIVATE "${chitrakaar_chitrakaar_PATH}/src/")

    # Find OpenGL
    set(OpenGL_GL_PREFERENCE "GLVND")
    find_package(OpenGL REQUIRED)
    target_include_directories(${TARGET_NAME} PRIVATE "${OPENGL_INCLUDE_DIR}")
    target_link_libraries(${TARGET_NAME} "${OPENGL_LIBRARIES}")
    message("OpenGL found: ${OPENGL_LIBRARIES}")

    # Build deps
    if (WIN32)
        execute_process(COMMAND "${chitrakaar_chitrakaar_PATH}/deps/build_deps.bat" ${CMAKE_GENERATOR} ${CMAKE_C_COMPILER} ${CMAKE_CXX_COMPILER} WORKING_DIRECTORY "${chitrakaar_chitrakaar_PATH}/deps/")
    else()
        execute_process(COMMAND "sh" "${chitrakaar_chitrakaar_PATH}/deps/build_deps.sh" ${CMAKE_GENERATOR} ${CMAKE_C_COMPILER} ${CMAKE_CXX_COMPILER} WORKING_DIRECTORY "${chitrakaar_chitrakaar_PATH}/deps/")
    endif()

    # Find SDL2
    find_package(SDL2 2 CONFIG REQUIRED)
    target_link_libraries(${TARGET_NAME} SDL2::SDL2 )


    if (WIN32)
        # Add SDL2 dependencies (needed because it is statically linked)
        target_link_libraries(${TARGET_NAME} winmm setupapi version imm32)
    endif()
endif()



if (NOT "${chitrakaar_IMGUI_PATH}" STREQUAL "" )
    message("The project uses Dear ImGui library: ${chitrakaar_IMGUI_PATH}")
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/backends/imgui_impl_sdl.cpp" )
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/backends/imgui_impl_opengl3.cpp" )
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/imgui.cpp" )
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/imgui_demo.cpp" )
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/imgui_draw.cpp" )
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/imgui_tables.cpp" )
    target_sources( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/imgui_widgets.cpp" )
    target_compile_definitions( ${TARGET_NAME} PUBLIC chitrakaar_SUPPORT_IMGUI)
    target_include_directories( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}")
    target_include_directories( ${TARGET_NAME} PUBLIC "${chitrakaar_IMGUI_PATH}/backends")
endif()




# Tell MSVC we don't want its Release/Debug folders.
if (MSVC)
    set_target_properties( ${TARGET_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/build )
    set_target_properties( ${TARGET_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_SOURCE_DIR}/build )
    set_target_properties( ${TARGET_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_SOURCE_DIR}/build )
endif()



# Enforce required standard on the target.
target_compile_features(${TARGET_NAME} PUBLIC cxx_std_${chitrakaar_CXX_STANDARD})



# On Linux, make sure that pthread is found.
if (UNIX AND NOT APPLE)
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
    target_link_libraries( ${TARGET_NAME} Threads::Threads)
endif()
