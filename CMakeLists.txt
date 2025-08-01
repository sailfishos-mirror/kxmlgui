cmake_minimum_required(VERSION 3.16)

set(KF_VERSION "6.18.0") # handled by release scripts
set(KF_DEP_VERSION "6.17.0") # handled by release scripts
project(KXmlGui VERSION ${KF_VERSION})

# ECM setup
include(FeatureSummary)
find_package(ECM 6.17.0  NO_MODULE)
set_package_properties(ECM PROPERTIES TYPE REQUIRED DESCRIPTION "Extra CMake Modules." URL "https://commits.kde.org/extra-cmake-modules")
feature_summary(WHAT REQUIRED_PACKAGES_NOT_FOUND FATAL_ON_MISSING_REQUIRED_PACKAGES)


set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH})

include(ECMMarkNonGuiExecutable)
include(ECMSetupVersion)
include(ECMGenerateHeaders)
include(ECMGenerateExportHeader)

include(KDEInstallDirs)
include(KDEFrameworkCompilerSettings NO_POLICY_SCOPE)
include(KDECMakeSettings)
include(KDEGitCommitHooks)
include(ECMQtDeclareLoggingCategory)
include(ECMDeprecationSettings)
include(ECMGenerateQDoc)
include(CMakeDependentOption)

set(kxmlgui_version_header "${CMAKE_CURRENT_BINARY_DIR}/src/kxmlgui_version.h")
ecm_setup_version(PROJECT
                  VARIABLE_PREFIX KXMLGUI
                  VERSION_HEADER "${kxmlgui_version_header}"
                  PACKAGE_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/KF6XmlGuiConfigVersion.cmake"
                  SOVERSION 6)

set(EXCLUDE_DEPRECATED_BEFORE_AND_AT 0 CACHE STRING "Control the range of deprecated API excluded from the build [default=0].")

option(FORCE_DISABLE_KGLOBALACCEL "Force building KXmlGui without KGlobalAccel. Doing this will break global shortcut support. [default=OFF]" OFF)

cmake_dependent_option(BUILD_DESIGNERPLUGIN "Build plugin for Qt Designer" ON "NOT CMAKE_CROSSCOMPILING" OFF)
add_feature_info(DESIGNERPLUGIN ${BUILD_DESIGNERPLUGIN} "Build plugin for Qt Designer")

# Dependencies
set(REQUIRED_QT_VERSION 6.7.0)
find_package(Qt6 ${REQUIRED_QT_VERSION} CONFIG REQUIRED Widgets Xml Network PrintSupport)
if(Qt6Core_VERSION VERSION_GREATER_EQUAL "6.9.0")
    find_package(Qt6 ${REQUIRED_QT_VERSION} CONFIG REQUIRED CorePrivate)
endif()

# shall we use DBus?
# enabled per default on Linux & BSD systems
set(USE_DBUS_DEFAULT OFF)
if(UNIX AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
    set(USE_DBUS_DEFAULT ON)
endif()
option(USE_DBUS "Build components using DBus" ${USE_DBUS_DEFAULT})
if(USE_DBUS)
    find_package(Qt6 ${REQUIRED_QT_VERSION} CONFIG REQUIRED DBus)
    set(HAVE_QTDBUS ${Qt6DBus_FOUND})

    # needs DBus + is only available on Linux & BSD
    if (NOT FORCE_DISABLE_KGLOBALACCEL AND NOT WIN32 AND NOT APPLE AND NOT ANDROID AND NOT HAIKU)
        find_package(KF6GlobalAccel ${KF_DEP_VERSION} REQUIRED)
        set (HAVE_GLOBALACCEL ${KF6GlobalAccel_FOUND})
    endif()
endif()

find_package(KF6CoreAddons ${KF_DEP_VERSION} REQUIRED)
find_package(KF6ItemViews ${KF_DEP_VERSION} REQUIRED)
find_package(KF6Config ${KF_DEP_VERSION} REQUIRED)
find_package(KF6ConfigWidgets ${KF_DEP_VERSION} REQUIRED)
find_package(KF6GuiAddons ${KF_DEP_VERSION} REQUIRED)
find_package(KF6I18n ${KF_DEP_VERSION} REQUIRED)
find_package(KF6IconThemes ${KF_DEP_VERSION} REQUIRED)
find_package(KF6WidgetsAddons ${KF_DEP_VERSION} REQUIRED)

option(BUILD_PYTHON_BINDINGS "Build Python bindings" ON)

# Only Linux and FreeBSD CI has the relevant packages
if (ANDROID OR APPLE OR WIN32 OR HAIKU OR NOT BUILD_SHARED_LIBS)
    set(BUILD_PYTHON_BINDINGS OFF)
endif()

if (BUILD_PYTHON_BINDINGS)
    find_package(Python3 3.9 REQUIRED COMPONENTS Interpreter Development)
    find_package(Shiboken6 REQUIRED)
    find_package(PySide6 REQUIRED)
endif()

ecm_set_disabled_deprecation_versions(
    QT 6.10
    KF 6.16
)

add_definitions(-DTRANSLATION_DOMAIN=\"kxmlgui6\")
ki18n_install(po)

add_feature_info(PYTHON_BINDINGS ${BUILD_PYTHON_BINDINGS} "Python bindings")

# Subdirectories
add_subdirectory(src)
if (BUILD_TESTING)
    add_subdirectory(tests)
    add_subdirectory(autotests)
endif()

if (BUILD_PYTHON_BINDINGS)
    include(ECMGeneratePythonBindings)
    add_subdirectory(python)
endif()

# create a Config.cmake and a ConfigVersion.cmake file and install them
set(CMAKECONFIG_INSTALL_DIR "${KDE_INSTALL_CMAKEPACKAGEDIR}/KF6XmlGui")

include(CMakePackageConfigHelpers)

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/KF6XmlGuiConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/KF6XmlGuiConfig.cmake"
  INSTALL_DESTINATION  ${CMAKECONFIG_INSTALL_DIR}
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/KF6XmlGuiConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/KF6XmlGuiConfigVersion.cmake"
  DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
  COMPONENT Devel
)

install(EXPORT KF6XmlGuiTargets DESTINATION "${CMAKECONFIG_INSTALL_DIR}" FILE KF6XmlGuiTargets.cmake NAMESPACE KF6:: )

install(FILES
   ${kxmlgui_version_header}
  DESTINATION ${KDE_INSTALL_INCLUDEDIR_KF}/KXmlGui COMPONENT Devel
)

include(ECMFeatureSummary)
ecm_feature_summary(WHAT ALL   FATAL_ON_MISSING_REQUIRED_PACKAGES)

kde_configure_git_pre_commit_hook(CHECKS CLANG_FORMAT)
