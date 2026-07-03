cmake_minimum_required(VERSION 4.0)

project(xxHash
    VERSION 0.8.5
    LANGUAGES C
)

if(WIN32)
    enable_language(RC)
endif()
if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
endif()
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
set(CMAKE_C_VISIBILITY_PRESET "hidden")
foreach(__lang C)
    if(NOT "x${CMAKE_${__lang}_FLAGS}" STREQUAL "x")
        if(MSVC)
            string(REGEX REPLACE " [/-]W[01234] " " " CMAKE_${__lang}_FLAGS ${CMAKE_${__lang}_FLAGS})
        else()
            string(REGEX REPLACE " -W(all)?(extra)? " " " CMAKE_${__lang}_FLAGS ${CMAKE_${__lang}_FLAGS})
            string(REGEX REPLACE " -[W]?pedantic " " " CMAKE_${__lang}_FLAGS ${CMAKE_${__lang}_FLAGS})
        endif()
    endif()
    string(APPEND CMAKE_${__lang}_FLAGS " -w ")
endforeach()

set(__top_level ON)
if("x${CMAKE_SOURCE_DIR}" STREQUAL "x${PROJECT_SOURCE_DIR}")
    message("xxHash: Building as a top-level project.")
else()
    set(__top_level OFF)
    message("xxHash: Building as a subproject.")
endif()

option(XXHASH_BUILD_SHARED "Build shared library edition of xxHash." ON)
option(XXHASH_BUILD_STATIC "Build static library edition of xxHash." ON)
option(XXHASH_INSTALL "Enable installation for xxHash." OFF)
if(NOT XXHASH_BUILD_SHARED AND NOT XXHASH_BUILD_STATIC)
    message(SEND_ERROR "You MUST build either the shared library or the static library or both of them!")
endif()
if(APPLE AND CMAKE_OSX_ARCHITECTURES)
    set(__architecture "${CMAKE_OSX_ARCHITECTURES}")
elseif(CMAKE_C_COMPILER_ARCHITECTURE_ID)
    set(__architecture "${CMAKE_C_COMPILER_ARCHITECTURE_ID}")
else()
    set(__architecture "${CMAKE_SYSTEM_PROCESSOR}")
endif()
string(TOLOWER "${__architecture}" __architecture)
set(__x86_dispatch OFF)
foreach(__arch IN LISTS __architecture)
    if(__arch MATCHES "^(amd64|x86_64|x64|x86|i[3-6]86)$")
        set(__x86_dispatch ON)
    endif()
endforeach()
set(__targets)
if(XXHASH_BUILD_SHARED)
    add_library(xxhash_shared SHARED)
    add_library(xxhash::xxhash_shared ALIAS xxhash_shared)
    # The amalgamated header only adds explicit export annotations on MSVC.
    set_target_properties(xxhash_shared PROPERTIES
        OUTPUT_NAME xxHash
        C_VISIBILITY_PRESET default
    )
    target_compile_definitions(xxhash_shared
        PUBLIC XXH_DLL
        PRIVATE XXH_EXPORT
        INTERFACE XXH_IMPORT
    )
    if(WIN32)
        target_sources(xxhash_shared PRIVATE xxhash.rc)
    endif()
    list(APPEND __targets xxhash_shared)
endif()
if(XXHASH_BUILD_STATIC)
    add_library(xxhash_static STATIC)
    add_library(xxhash::xxhash_static ALIAS xxhash_static)
    list(APPEND __targets xxhash_static)
endif()
if(XXHASH_BUILD_SHARED AND NOT XXHASH_BUILD_STATIC)
    add_library(xxhash::xxhash ALIAS xxhash_shared)
endif()
if(XXHASH_BUILD_STATIC AND NOT XXHASH_BUILD_SHARED)
    add_library(xxhash::xxhash ALIAS xxhash_static)
endif()

include(GNUInstallDirs)
foreach(__target IN LISTS __targets)
    target_sources(${__target} PRIVATE xxhash.h xxhash.c)
    if(__x86_dispatch)
        target_sources(${__target} PRIVATE xxh_x86dispatch.h xxh_x86dispatch.c)
    endif()
    target_include_directories(${__target} PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}
    )
    if(__top_level)
        if(WIN32)
            target_compile_definitions(${__target} PRIVATE
                UNICODE _UNICODE
                NOMINMAX
                _USE_MATH_DEFINES
                WIN32_LEAN_AND_MEAN
            )
        endif()
        if(MSVC)
            target_compile_options(${__target} PRIVATE
                /utf-8 /Zc:__cplusplus
                $<$<NOT:$<CONFIG:Debug>>:/Gw /Gy /Zc:inline>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:/OPT:REF /OPT:ICF>
            )
        else()
            if(APPLE)
                target_link_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,-dead_strip>
                )
            else()
                target_compile_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-ffunction-sections -fdata-sections>
                )
                target_link_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,--gc-sections>
                )
            endif()
        endif()
    endif()
    set_target_properties(${__target} PROPERTIES
        VERSION ${PROJECT_VERSION}
    )
    get_target_property(__type ${__target} TYPE)
    if(NOT __type STREQUAL "STATIC_LIBRARY")
        set_target_properties(${__target} PROPERTIES
            SOVERSION ${PROJECT_VERSION_MAJOR}
        )
    endif()
    if(XXHASH_INSTALL)
        install(TARGETS ${__target}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
    endif()
endforeach()

if(XXHASH_INSTALL)
    install(FILES xxhash.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
    if(__x86_dispatch)
        install(FILES xxh_x86dispatch.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
    endif()
endif()
