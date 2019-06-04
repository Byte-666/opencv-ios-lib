cmake_minimum_required (VERSION 3.7)
set(CMAKE_VERBOSE_MAKEFILE TRUE)

# OpenCV setup

macro(FW_OPTION variable value)
    unset(${variable} CACHE)
    option(${variable} ${value})
endmacro()

#set(OPENCV_FPON6_DISABLE ON)
FW_OPTION(BUILD_FAT_JAVA_LIB "fat" OFF)
FW_OPTION(BUILD_opencv_apps "opencv_apps" OFF) 
FW_OPTION(BUILD_SHARED_LIBS OFF)
FW_OPTION(BUILD_OPENCV_WORLD ON)
FW_OPTION(BUILD_TESTS OFF)
FW_OPTION(BUILD_EXAMPLES OFF)
FW_OPTION(BUILD_PERF_TESTS OFF)
FW_OPTION(BUILD_JPEG ON)
FW_OPTION(BUILD_JASPER ON)
FW_OPTION(BUILD_PNG ON)
FW_OPTION(WITH_TBB ON)

add_subdirectory(opencv)
find_library(OpenCV REQUIRED)

# find_package(OpenCV REQUIRED)

ocv_get_all_libs(OCV_MODULES OCV_EXTRA OCV_3RDPARTY)
message(STATUS "OpenCV libraries: ${OCV_MODULES}")

set(OCV_INCLUDE_DIR "")
foreach(m ${OCV_MODULES})
    # only "public" headers need to be installed
    get_target_property(incl ${m} INCLUDE_DIRECTORIES)
    list(INSERT OCV_INCLUDE_DIR 0 ${incl})
endforeach()
ocv_list_unique(OCV_INCLUDE_DIR)
ocv_list_reverse(OCV_INCLUDE_DIR)
message(STATUS "OpenCV include_directories: ${OCV_INCLUDE_DIR}")

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.

set(parent_DIR ${CMAKE_SOURCE_DIR})
set(include_DIR ${parent_DIR}/opencv/include)
include_directories(
    ${include_DIR}
    ${include_DIR}/opencv
    ${include_DIR}/opencv2
    ${OCV_INCLUDE_DIR})

set(LIBRARY_SOURCE
    ${CMAKE_CURRENT_LIST_DIR}/Foo.hh
    ${CMAKE_CURRENT_LIST_DIR}/Foo.cpp
)

add_library(
    ${FRAMEWORK_NAME} SHARED
    ${LIBRARY_SOURCE}
)

target_link_libraries(
    ${FRAMEWORK_NAME}

    # OpenCV
    ${OCV_3RDPARTY}
    ${OCV_MODULES}
    ${OCV_EXTRA}

)

set(CMAKE_SHARED_LINKER_FLAGS "-Wall")

set_target_properties(
    ${FRAMEWORK_NAME} PROPERTIES
    FRAMEWORK TRUE
    FRAMEWORK_VERSION A
    MACOSX_FRAMEWORK_IDENTIFIER ${FRAMEWORK_BUNDLE_IDENTIFIER}
    MACOSX_FRAMEWORK_INFO_PLIST ${CMAKE_CURRENT_LIST_DIR}/framework.plist.in
    # "current version" in semantic format in Mach-O binary file
    VERSION 1.0.0
    # "compatibility version" in semantic format in Mach-O binary file
    SOVERSION 1.0.0
    PUBLIC_HEADER "${CMAKE_CURRENT_LIST_DIR}/Foo.hh"
    XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET ${DEPLOYMENT_TARGET}
    XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY ${CODE_SIGN_IDENTITY}
    XCODE_ATTRIBUTE_DEVELOPMENT_TEAM ${DEVELOPMENT_TEAM_ID}
    XCODE_ATTRIBUTE_TARGETED_DEVICE_FAMILY ${DEVICE_FAMILY}
    XCODE_ATTRIBUTE_SKIP_INSTALL "YES"
)

# Symbol visibility setup, COMPILE_FLAGS only affect C++ so for Objective C we
# have to use XCODE_ATTRIBUTE_OTHER_CFLAGS.
set_target_properties(
    ${FRAMEWORK_NAME} PROPERTIES
    COMPILE_FLAGS "-fvisibility=hidden -fvisibility-inlines-hidden"
    XCODE_ATTRIBUTE_OTHER_CFLAGS "-fvisibility=hidden -fvisibility-inlines-hidden")

# set_target_properties(${FRAMEWORK_NAME} PROPERTIES COMPILE_FLAGS "-x c++")

add_custom_command(
    TARGET ${FRAMEWORK_NAME}
    POST_BUILD
    COMMAND /bin/bash -c "${CMAKE_CURRENT_LIST_DIR}/install_name.sh \${BUILT_PRODUCTS_DIR}/\${PRODUCT_NAME}.framework/\${PRODUCT_NAME}"
)

add_custom_command(
    TARGET ${FRAMEWORK_NAME}
    POST_BUILD
    COMMAND install_name_tool -id \"@rpath/\${PRODUCT_NAME}.framework/\${PRODUCT_NAME}\"
    \${BUILT_PRODUCTS_DIR}/\${PRODUCT_NAME}.framework/\${PRODUCT_NAME}
)

