cmake_minimum_required(VERSION 3.10)

project(Rufusl LANGUAGES CXX C)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

find_package(Qt6 COMPONENTS Widgets REQUIRED)
find_package(LIBPARTED REQUIRED)

add_executable(rufusl
    main.cpp
    rufusworker.cpp
    iso.c
    linux/devices.c
    linux/fat32.c
    linux/mounting.c
    linux/partition.c
    linux/user.c
    ui/about.cpp
    ui/devicecombobox.cpp
    ui/errordialog.cpp
    ui/log.cpp
    ui/log.h
    ui/rufuswindow.cpp
)

set_target_properties(rufusl PROPERTIES
    AUTOMOC ON
    AUTOUIC ON
)

include_directories(
    ui
    ${CMAKE_SOURCE_DIR}
    linux
    ${LIBPARTED_INCLUDE_DIR}
)

target_link_libraries(rufusl Qt6::Widgets ${LIBPARTED_LIBS})
