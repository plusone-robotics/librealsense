cmake_minimum_required(VERSION 3.28.3)

set( LRS_TARGET realsense2 )
project( ${LRS_TARGET} VERSION 2.54.3 LANGUAGES CXX C )
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_BUILD_TYPE "Release")

# Allow librealsense2 and all of the nested project to include the main repo folder
set(REPO_ROOT ${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${REPO_ROOT})

include(CMake/lrs_options.cmake)
include(CMake/connectivity_check.cmake)
#Deprecation message, should be removed in future releases
if(${FORCE_LIBUVC} OR ${FORCE_WINUSB_UVC} OR ${ANDROID_USB_HOST_UVC})
    MESSAGE(DEPRECATION "FORCE_LIBUVC, FORCE_WINUSB_UVC and ANDROID_USB_HOST_UVC are deprecated, use FORCE_RSUSB_BACKEND instead")
    set(FORCE_RSUSB_BACKEND ON)
endif()

# Checking Internet connection, as DEPTH CAM needs to download the FW from amazon cloud
if(IMPORT_DEPTH_CAM_FW AND NOT INTERNET_CONNECTION)
    message(WARNING "No internet connection, disabling IMPORT_DEPTH_CAM_FW")
    set(IMPORT_DEPTH_CAM_FW OFF)
endif()

if (BUILD_PC_STITCHING AND NOT BUILD_GLSL_EXTENSIONS)
    MESSAGE(STATUS "BUILD_PC_STITCHING explicitely depends on BUILD_GLSL_EXTENSIONS, set it ON")
    SET(BUILD_GLSL_EXTENSIONS ON)
endif()

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/CMake)

# include librealsense general configuration
include(CMake/global_config.cmake)

# include os specific macros
# macro definition located at "CMake/global_config.cmake"
include(CMake/include_os.cmake)

# set os specific configuration flags
# macro definition located at "CMake/<OS>_config.cmake"
os_set_flags()

# set global configuration flags
# macro definition located at "CMake/global_config.cmake"
global_set_flags()

if(BUILD_SHARED_LIBS)
    add_library(${LRS_TARGET} SHARED "")
else()
    add_library(${LRS_TARGET} STATIC "")
endif()

config_cxx_flags()

# set library version
set_target_properties(${LRS_TARGET} PROPERTIES VERSION ${REALSENSE_VERSION_STRING} SOVERSION "${REALSENSE_VERSION_MAJOR}.${REALSENSE_VERSION_MINOR}")

include(include/CMakeLists.txt)
include(src/CMakeLists.txt)
include(third-party/CMakeLists.txt)
include(common/utilities/CMakeLists.txt)

target_link_libraries( ${LRS_TARGET} PUBLIC rsutils )

# configure the project according to OS specific requirments
# macro definition located at "CMake/<OS>_config.cmake"
os_target_config()

# global project configuration
# macro definition located at "CMake/global_config.cmake"
global_target_config()

include(CMake/install_config.cmake)

add_subdirectory(wrappers)
if ( ( BUILD_EXAMPLES OR BUILD_PC_STITCHING ) AND BUILD_GLSL_EXTENSIONS )
    find_package(glfw3 3.3 QUIET)
    if(NOT TARGET glfw)
        message(STATUS "GLFW 3.3 not found; using internal version")
        set(GLFW_INSTALL ON CACHE BOOL "" FORCE)
        add_subdirectory(third-party/glfw)
    endif()
    add_subdirectory(src/gl)
endif()

if(BUILD_EXAMPLES)
    add_subdirectory(examples)
    add_subdirectory(tools)
elseif(BUILD_TOOLS)
    add_subdirectory(tools)
endif()

if(BUILD_UNIT_TESTS OR BUILD_LEGACY_LIVE_TEST)
    add_subdirectory(unit-tests)
endif()

if(IMPORT_DEPTH_CAM_FW)
    add_subdirectory(common/fw)
endif()

include(CMake/embedd_udev_rules.cmake)

set(CPACK_PACKAGE_NAME librealsense2)
set(CPACK_PACKAGE_CONTACT "Aaron Wood <aaron.wood@plusonerobotics.com>")
set(CPACK_GENERATOR DEB)
set(CPACK_DEBIAN_PACKAGE_DEPENDS
        "libusb-1.0-0 (>= 2:1.0.16), libudev1 (>= 183), libc6 (>= 2.29), libgcc-s1 (>= 3.0), libstdc++6 (>= 9)")
set(CPACK_DEBIAN_PACKAGE_RECOMMENDS "librealsense2-udev-rules")
set(CPACK_DEBIAN_PACKAGE_SECTION libs)
set(CPACK_DEBIAN_PACKAGE_PRIORITY extra)
set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE amd64)
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE https://github.com/plusone-robotics/librealsense)
set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "Intel(R) RealSense(tm) Cross Platform API - runtime
This package contains the shared library.
To interact with an attached realsense device, install
the udev rules to configure device permissions (recommended).")
include(CPack)
