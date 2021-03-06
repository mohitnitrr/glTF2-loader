cmake_minimum_required(VERSION 3.1)

macro(set_option var default type docstring)
    if(NOT DEFINED ${var})
        set(${var} ${default})
    endif()

    set(${var} ${${var}} CACHE ${type} ${docstring} FORCE)
endmacro()

if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there. You may need to remove CMakeCache.txt.")
endif()

# determine the build type
set_option(CMAKE_BUILD_TYPE Release STRING "Choose the type of build (Debug or Release)")

# set android options
if(ANDROID)
    set(ANDROID_ABI arm64-v8a)
    message(STATUS "ANDROID_ABI: ${ANDROID_ABI}")

    set(ANDROID_TOOLCHAIN clang)
    message(STATUS "ANDROID_TOOLCHAIN: ${ANDROID_TOOLCHAIN}")

    set_option(ANDROID_PLATFORM android-24 STRING "Target specific android version target (default: android-24)")
    if(NOT ANDROID_PLATFORM MATCHES android-2[4-9])
        message(FATAL_ERROR "Invalid android version target: ${ANDROID_PLATFORM}")
    endif()
    message(STATUS "ANDROID_PLATFORM: ${ANDROID_PLATFORM}")

    set_option(ANDROID_STL c++_shared STRING "Choose which Android STL to use (default: c++_shared)")

    if(NOT ANDROID_STL STREQUAL c++_static AND NOT ANDROID_STL STREQUAL c++_shared)
        message(FATAL_ERROR "Invalid STL: ${ANDROID_STL}")
    endif()
endif()

# project name
project(glTF2-loader C CXX)

# use c++14
set(CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 14)

# add the include path
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR})

# enable project folders
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMake")

set(target "gltf2-loader")

set(SRCS
    "src/gltf2/glTF2.cpp"
)

set(INCS

)

add_library(${target} STATIC ${SRCS} ${INCS})
set_target_properties(${target} PROPERTIES DEBUG_POSTFIX -d)
set_target_properties(${target} PROPERTIES POSITION_INDEPENDENT_CODE ON)

# setup the install of headers
install(DIRECTORY include
        DESTINATION .
        COMPONENT devel
        FILES_MATCHING PATTERN "*.hpp" PATTERN "*.inl" PATTERN "*.h" PATTERN "*.cc"
)

# setup the install of the library
install(TARGETS ${target}
        RUNTIME DESTINATION bin COMPONENT bin
        LIBRARY DESTINATION lib COMPONENT bin
        ARCHIVE DESTINATION lib COMPONENT devel
        FRAMEWORK DESTINATION ${CMAKE_INSTALL_FRAMEWORK_PREFIX} COMPONENT bin
)

# c++14 on android (CMAKE_CXX_STANDARD doesn't work with the Android Toolchain)
if (ANDROID)
    target_compile_options(${target} PUBLIC -std=c++14)
endif()

# use warnings and errors
if (MSVC)
    target_compile_options(${target} PUBLIC /W4 /WX)
else()
    target_compile_options(${target} PUBLIC -Wall -Wextra -Werror)
endif()

# add parallel build for MSVC
if (MSVC)
    target_compile_options(${target} PUBLIC /MP)
endif()

# use gcc's libstdc++ because it has the modern namespaced ABI for std containers
if(CMAKE_CXX_COMPILER MATCHES ".*clang[+][+]" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang++")
    target_compile_options(${target} PUBLIC "-stdlib=libstdc++")
endif()

# unit tests
set_option(BUILD_TESTS FALSE BOOL "TRUE to enable unit tests, FALSE to disable unit tests")
if(BUILD_TESTS)
    # Note: enable_testing() MUST be on the top level CMakeLists.txt
    enable_testing()
    add_subdirectory(test/)
endif()
