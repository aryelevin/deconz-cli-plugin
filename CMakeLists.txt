cmake_minimum_required(VERSION 3.13)

project(deconz_cli_plugin VERSION 2.22.02 LANGUAGES C;CXX)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Qt5 COMPONENTS Core Widgets Network WebSockets REQUIRED)
find_package(Git REQUIRED)

if (UNIX)
    include(FindPkgConfig)
    pkg_check_modules (OPENSSL openssl)
endif()

if (NOT DECONZ_FULL_BUILD)
    # standalone build
    # add_subdirectory(sqlite3)
endif()

#--- deCONZ library ---------------------------------------------------

include(FetchContent)

FetchContent_Declare(
  deconzlib
  GIT_REPOSITORY https://github.com/dresden-elektronik/deconz-lib.git
  GIT_TAG        main
)
# FetchContent_MakeAvailable(deconzlib)
# FetchContent_MakeAvailable requires CMake 3.14, but Debian Buster has only 3.13
FetchContent_GetProperties(deconzlib)
if (NOT deconzlib_POPULATED)
    FetchContent_Populate(deconzlib)
    add_subdirectory(${deconzlib_SOURCE_DIR} ${deconzlib_BINARY_DIR})
endif()

#----------------------------------------------------------------------

set(PLUGIN_INCLUDE_FILES
    deconz_cli_plugin.h
    )

add_library(${PROJECT_NAME} SHARED
    ${PLUGIN_INCLUDE_FILES}

    deconz_cli_plugin.cpp
    )

# if (Git_FOUND)
#     execute_process(
#         COMMAND ${GIT_EXECUTABLE} log -1 --format=%at
#         WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
#         OUTPUT_VARIABLE GIT_DATE
#         OUTPUT_STRIP_TRAILING_WHITESPACE
#     )
#     execute_process(
#         COMMAND ${GIT_EXECUTABLE} rev-list HEAD --max-count=1
#         WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
#         OUTPUT_VARIABLE SRC_GIT_COMMIT
#         OUTPUT_STRIP_TRAILING_WHITESPACE
#     )
# endif()

# target_compile_definitions(${PROJECT_NAME}
#     PRIVATE
#     USE_WEBSOCKETS=1
#     USE_DUKTAPE_JS_ENGINE=1

#     GW_SW_VERSION="\"\"${PROJECT_VERSION}\"\""
#     GW_SW_DATE=${GIT_DATE}
#     GW_API_VERSION="\"\"1.16.0\"\""
#     GIT_COMMMIT="\"\"${SRC_GIT_COMMIT}\"\""

#     # Minimum version of the RaspBee firmware
#     # which shall be used in order to support all features for this software release (case sensitive)
#     GW_AUTO_UPDATE_AVR_FW_VERSION=0x260b0500
#     GW_AUTO_UPDATE_R21_FW_VERSION=0x26420700
#     GW_MIN_AVR_FW_VERSION=0x26400500
#     GW_MIN_R21_FW_VERSION=0x26720700

#     # Minimum version of the deRFusb23E0X firmware
#     # which shall be used in order to support all features for this software release
#     GW_MIN_DERFUSB23E0X_FW_VERSION=0x22030300
#     GW_DEFAULT_NAME="\"\"Phoscon-GW\"\""
# )

if (OPENSSL_FOUND)
    target_compile_definitions(${PROJECT_NAME} PRIVATE HAS_OPENSSL=1)
    target_include_directories(${PROJECT_NAME} PRIVATE ${OPENSSL_INCLUDE_DIRS})
endif()

if (Qt5Core_VERSION_STRING VERSION_LESS "5.15.0")
    target_compile_definitions(${PROJECT_NAME} PRIVATE SKIP_EMPTY_PARTS=QString::SkipEmptyParts )
else()
    target_compile_definitions(${PROJECT_NAME} PRIVATE SKIP_EMPTY_PARTS=Qt::SkipEmptyParts )
endif()

target_link_libraries(${PROJECT_NAME}
    PRIVATE Qt5::Core
    PRIVATE Qt5::Gui
    PRIVATE Qt5::Widgets
    PRIVATE Qt5::Network
    PRIVATE Qt5::WebSockets
    # PRIVATE SQLite::SQLite3
    PRIVATE deCONZLib
)

#--------------------------------------------------------------
include(GNUInstallDirs)

if (UNIX)
    if (APPLE)
        set_target_properties(${PROJECT_NAME} PROPERTIES
            INSTALL_RPATH @loader_path)
    else()
        set_target_properties(${PROJECT_NAME} PROPERTIES
            INSTALL_RPATH "${CMAKE_INSTALL_FULL_LIBDIR}")
    endif()
endif()


if (${CMAKE_HOST_SYSTEM_NAME} MATCHES "Linux")
    install(TARGETS ${PROJECT_NAME}
        LIBRARY DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/deCONZ/plugins"
    )

    # following applies only when build as part of deCONZ package
    if (DECONZ_FULL_BUILD)
        install(FILES "general.xml" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/deCONZ/zcl")
        install(DIRECTORY devices DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/deCONZ")
        install(FILES "button_maps.json" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/deCONZ/devices")

        install(DIRECTORY "linux/deCONZ/lib/systemd" DESTINATION "/lib")
        install(
            DIRECTORY "linux/deCONZ/usr/bin/"
            DESTINATION ${CMAKE_INSTALL_BINDIR}
            FILES_MATCHING
            PATTERN "*.sh"
        )
    endif()
endif()

#add_library(deconz_common INTERFACE)
#target_include_directories(deconz_common INTERFACE ${DECONZ_INCLUDE_DIR})
#target_link_directories(deconz_common INTERFACE /usr/local/lib)
#target_link_libraries(deconz_common
 #       INTERFACE Qt${QT_VERSION_MAJOR}::Core
#        INTERFACE Qt${QT_VERSION_MAJOR}::Gui
#        INTERFACE Qt${QT_VERSION_MAJOR}::Widgets
#        INTERFACE Qt${QT_VERSION_MAJOR}::Qml
#        deCONZ.so
#)

#add_subdirectory (resource)
#add_subdirectory (device_js)
#add_subdirectory (device)
#add_subdirectory (utils)
#add_subdirectory (zcl)
#add_subdirectory (zdp)

#enable_testing()

#add_subdirectory (tests)
