cmake_minimum_required(VERSION 3.0.0)
project(lomiri-screen-recorder C CXX)

ADD_DEFINITIONS(-DQT_NO_KEYWORDS)
ADD_DEFINITIONS(-DQT_NO_DEBUG)

# Automatically create moc files
set(CMAKE_AUTOMOC ON)

option(CLICK_MODE "Installs to a contained location" ON)

find_package(PkgConfig)
find_package(Qt5Core REQUIRED)
find_package(Qt5Qml REQUIRED)
find_package(Qt5Quick REQUIRED)
find_package(Qt5QuickControls2 REQUIRED)

pkg_check_modules(MIRCLIENT REQUIRED mirclient)
pkg_check_modules(HYBRIS_MEDIA REQUIRED libmedia)
pkg_check_modules(ANDROID_HEADERS REQUIRED android-headers-29)
pkg_check_modules(LOMIRIURLDISPATCHER REQUIRED lomiri-url-dispatcher)
#pkg_check_modules(MESSAGING_MENU REQUIRED messaging-menu)
pkg_check_modules(GLIB REQUIRED
    gio-unix-2.0>=2.36
    glib-2.0>=2.36
)

execute_process(
    COMMAND dpkg-architecture -qDEB_HOST_MULTIARCH
    OUTPUT_VARIABLE ARCH_TRIPLET
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

if(CLICK_MODE)
    set(QT_IMPORTS_DIR "lib/${ARCH_TRIPLET}")
    set(PROJECT_NAME "screenrecorder")
    set(FULL_PROJECT_NAME "screenrecorder.ubports")
    set(DATA_DIR /)
    set(DESKTOP_FILE_NAME screenrecorder.desktop)
    ADD_DEFINITIONS(-DCLICK_MODE)
else()
    set(QT_IMPORTS_DIR "${CMAKE_INSTALL_LIBDIR}/qt5/qml")
    set(PROJECT_NAME "lomiri-screen-recorder")
    set(FULL_PROJECT_NAME "screenrecorder.ubports")
    set(DATA_DIR "${CMAKE_INSTALL_PREFIX}/share/applications")
    set(DESKTOP_FILE_NAME lomiri-screen-recorder.desktop)
endif()

# This command figures out the minimum SDK framework for use in the manifest
# file via the environment variable provided by Clickable or sets a default value otherwise.
if(DEFINED ENV{SDK_FRAMEWORK})
    set(CLICK_FRAMEWORK "$ENV{SDK_FRAMEWORK}")
else()
    set(CLICK_FRAMEWORK "ubuntu-sdk-20.04")
endif()

# This figures out the target architecture for use in the manifest file.
if(DEFINED ENV{ARCH})
    set(CLICK_ARCH "$ENV{ARCH}")
else()
    execute_process(
        COMMAND dpkg-architecture -qDEB_HOST_ARCH
        OUTPUT_VARIABLE CLICK_ARCH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
endif()

if (CLICK_MODE)
    configure_file(manifest.json.in ${CMAKE_CURRENT_BINARY_DIR}/manifest.json)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/manifest.json DESTINATION ${CMAKE_INSTALL_PREFIX})
    install(FILES ${PROJECT_NAME}.apparmor DESTINATION ${DATA_DIR})
    install(FILES ${PROJECT_NAME}-urldispatcher.json DESTINATION ${DATA_DIR})
    install(FILES assets/logo.png DESTINATION assets)
    install(FILES assets/splash.png DESTINATION assets)
else()
    install(FILES assets/logo.png DESTINATION "${CMAKE_INSTALL_PREFIX}/share/icons/hicolor/512x512/apps" RENAME lomiri-screen-recorder.png)
    install(FILES assets/splash.png DESTINATION "${CMAKE_INSTALL_PREFIX}/share/lomiri-screen-recorder")
    install(FILES lomiri-screen-recorder.indicator DESTINATION "${CMAKE_INSTALL_PREFIX}/share/ayatana/indicators" RENAME "ubports.screenrecorder.indicator")
    install(FILES screenrecorder-urldispatcher.json DESTINATION "${CMAKE_INSTALL_PREFIX}/share/lomiri-url-dispatcher/urls" RENAME "lomiri-screen-recorder.url-dispatcher")
endif()

qt5_add_resources(QT_RESOURCES qml/qml.qrc)
qt5_add_resources(QT_RESOURCES assets/assets.qrc)
add_executable(${PROJECT_NAME} main.cpp ${QT_RESOURCES})
set_target_properties(${PROJECT_NAME} PROPERTIES LINK_FLAGS_RELEASE -s)
target_link_libraries(${PROJECT_NAME} Qt5::Gui Qt5::Qml Qt5::Quick Qt5::QuickControls2 mirclient mircookie mircore)
install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)

# Translations
file(GLOB_RECURSE I18N_SRC_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/po qml/*.qml qml/*.js)
list(APPEND I18N_SRC_FILES ${DESKTOP_FILE_NAME}.in.h)

find_program(INTLTOOL_MERGE intltool-merge)
if(NOT INTLTOOL_MERGE)
    message(FATAL_ERROR "Could not find intltool-merge, please install the intltool package")
endif()
find_program(INTLTOOL_EXTRACT intltool-extract)
if(NOT INTLTOOL_EXTRACT)
    message(FATAL_ERROR "Could not find intltool-extract, please install the intltool package")
endif()

add_custom_target(${DESKTOP_FILE_NAME} ALL
    COMMENT "Merging translations into ${DESKTOP_FILE_NAME}..."
    COMMAND LC_ALL=C ${INTLTOOL_MERGE} -d -u ${CMAKE_SOURCE_DIR}/po ${CMAKE_SOURCE_DIR}/${DESKTOP_FILE_NAME}.in ${DESKTOP_FILE_NAME}
    COMMAND sed -i 's/${PROJECT_NAME}-//g' ${CMAKE_CURRENT_BINARY_DIR}/${DESKTOP_FILE_NAME}
)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${DESKTOP_FILE_NAME} DESTINATION ${DATA_DIR})

add_subdirectory(po)
add_subdirectory(plugins)

# Make source files visible in qtcreator
file(GLOB_RECURSE PROJECT_SRC_FILES
    RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
    qml/*.qml
    qml/*.js
    src/*
    *.json
    *.json.in
    *.apparmor
    *.desktop.in
)

add_custom_target(${PROJECT_NAME}_FILES ALL SOURCES ${PROJECT_SRC_FILES})
