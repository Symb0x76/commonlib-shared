cmake_minimum_required(VERSION 3.24)

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    project(CommonLibShared VERSION 0.0.0 LANGUAGES CXX)
endif()

include(GNUInstallDirs)

if(NOT WIN32)
    message(FATAL_ERROR "commonlib-shared only supports Windows targets.")
endif()

if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8)
    message(FATAL_ERROR "commonlib-shared requires a 64-bit toolchain.")
endif()

if(NOT MSVC)
    message(FATAL_ERROR "commonlib-shared requires MSVC or clang-cl.")
endif()

option(COMMONLIBF4_INSTALL "Install CommonLibF4 targets and package exports" ${PROJECT_IS_TOP_LEVEL})
option(COMMONLIBF4_ENABLE_INI "Enable REX::INI settings support" OFF)
option(COMMONLIBF4_ENABLE_JSON "Enable REX::JSON settings support" OFF)
option(COMMONLIBF4_ENABLE_TOML "Enable REX::TOML settings support" OFF)
option(COMMONLIBF4_ENABLE_XBYAK "Enable xbyak support for REL::Trampoline" OFF)

find_package(spdlog CONFIG REQUIRED)

set(commonlib_shared_public_deps spdlog::spdlog)
set(commonlib_shared_optional_exports)

if(COMMONLIBF4_ENABLE_INI)
    find_path(COMMONLIBF4_SIMPLEINI_INCLUDE_DIR NAMES SimpleIni.h REQUIRED)

    add_library(commonlibf4_simpleini INTERFACE)
    add_library(CommonLibF4::commonlibf4_simpleini ALIAS commonlibf4_simpleini)
    target_include_directories(commonlibf4_simpleini
        INTERFACE
            "$<BUILD_INTERFACE:${COMMONLIBF4_SIMPLEINI_INCLUDE_DIR}>"
            "$<INSTALL_INTERFACE:${COMMONLIBF4_SIMPLEINI_INCLUDE_DIR}>")

    list(APPEND commonlib_shared_public_deps commonlibf4_simpleini)
    list(APPEND commonlib_shared_optional_exports commonlibf4_simpleini)
endif()

if(COMMONLIBF4_ENABLE_JSON)
    find_package(glaze CONFIG REQUIRED)
    list(APPEND commonlib_shared_public_deps glaze::glaze)
endif()

if(COMMONLIBF4_ENABLE_TOML)
    find_package(toml11 CONFIG REQUIRED)
    list(APPEND commonlib_shared_public_deps toml11::toml11)
endif()

if(COMMONLIBF4_ENABLE_XBYAK)
    find_package(xbyak CONFIG REQUIRED)
    list(APPEND commonlib_shared_public_deps xbyak::xbyak)
endif()

file(GLOB_RECURSE commonlib_shared_sources CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
file(GLOB_RECURSE commonlib_shared_headers CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/include/*.h")

add_library(commonlib-shared STATIC)
add_library(CommonLibF4::commonlib-shared ALIAS commonlib-shared)

target_sources(commonlib-shared
    PRIVATE
        ${commonlib_shared_sources}
        ${commonlib_shared_headers})

target_compile_features(commonlib-shared PUBLIC cxx_std_23)
target_include_directories(commonlib-shared
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
target_link_libraries(commonlib-shared
    PUBLIC
        ${commonlib_shared_public_deps}
        advapi32
        bcrypt
        d3d11
        d3dcompiler
        dbghelp
        dxgi
        ole32
        shell32
        user32
        version
        ws2_32)

target_precompile_headers(commonlib-shared
    PRIVATE
        "$<$<COMPILE_LANGUAGE:CXX>:${CMAKE_CURRENT_SOURCE_DIR}/src/REX/PCH.h>")

target_compile_definitions(commonlib-shared PUBLIC SPDLOG_USE_STD_FORMAT)

if(COMMONLIBF4_ENABLE_INI)
    target_compile_definitions(commonlib-shared PUBLIC COMMONLIB_OPTION_INI=1)
endif()

if(COMMONLIBF4_ENABLE_JSON)
    target_compile_definitions(commonlib-shared PUBLIC COMMONLIB_OPTION_JSON=1)
endif()

if(COMMONLIBF4_ENABLE_TOML)
    target_compile_definitions(commonlib-shared PUBLIC COMMONLIB_OPTION_TOML=1)
endif()

if(COMMONLIBF4_ENABLE_XBYAK)
    target_compile_definitions(commonlib-shared PUBLIC COMMONLIB_OPTION_XBYAK=1)
endif()

if(MSVC)
    target_compile_options(commonlib-shared
        PRIVATE
            /utf-8
        PUBLIC
            /EHsc
            /permissive-
            /bigobj
            /cgthreads8
            /diagnostics:caret
            /external:W0
            /fp:contract
            /fp:except-
            /guard:cf-
            /wd4200
            /wd4201
            /wd4324
            /we4715
            /Zc:enumTypes
            /Zc:preprocessor
            /Zc:templateScope)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND MSVC)
    target_compile_options(commonlib-shared
        PUBLIC
            -fms-compatibility
            -fms-extensions
            -Wno-delete-non-abstract-non-virtual-dtor
            -Wno-deprecated-volatile
            -Wno-ignored-qualifiers
            -Wno-inconsistent-missing-override
            -Wno-invalid-offsetof
            -Wno-microsoft-include
            -Wno-overloaded-virtual
            -Wno-pragma-system-header-outside-header
            -Wno-reinterpret-base-class
            -Wno-switch
            -Wno-unused-local-typedef
            -Wno-unused-private-field)
endif()

source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" FILES ${commonlib_shared_sources} ${commonlib_shared_headers})

if(COMMONLIBF4_INSTALL)
    install(TARGETS commonlib-shared ${commonlib_shared_optional_exports}
        EXPORT CommonLibF4Targets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

    install(DIRECTORY
        "${CMAKE_CURRENT_SOURCE_DIR}/include/REL"
        "${CMAKE_CURRENT_SOURCE_DIR}/include/REX"
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif()
