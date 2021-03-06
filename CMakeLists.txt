cmake_minimum_required (VERSION 3.12)
project (game_overlay_sdk)

macro(configure_msvc_runtime)
  if(MSVC)
    # Default to statically-linked runtime.
    if("${MSVC_RUNTIME}" STREQUAL "")
      set(MSVC_RUNTIME "static")
    endif()
    # Set compiler options.
    set(variables
      CMAKE_C_FLAGS_DEBUG
      CMAKE_C_FLAGS_MINSIZEREL
      CMAKE_C_FLAGS_RELEASE
      CMAKE_C_FLAGS_RELWITHDEBINFO
      CMAKE_CXX_FLAGS_DEBUG
      CMAKE_CXX_FLAGS_MINSIZEREL
      CMAKE_CXX_FLAGS_RELEASE
      CMAKE_CXX_FLAGS_RELWITHDEBINFO
    )
    if(${MSVC_RUNTIME} STREQUAL "static")
      message(STATUS
        "MSVC -> forcing use of statically-linked runtime."
      )
      foreach(variable ${variables})
        if(${variable} MATCHES "/MD")
          string(REGEX REPLACE "/MD" "/MT" ${variable} "${${variable}}")
        endif()
      endforeach()
    else()
      message(STATUS
        "MSVC -> forcing use of dynamically-linked runtime."
      )
      foreach(variable ${variables})
        if(${variable} MATCHES "/MT")
          string(REGEX REPLACE "/MT" "/MD" ${variable} "${${variable}}")
        endif()
      endforeach()
    endif()
  endif()
endmacro()

configure_msvc_runtime ()

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
  MESSAGE("64 bits compiler detected")
  SET(EX_PLATFORM 64)
  SET(EX_PLATFORM_NAME "x64")

  SET(GAME_OVERLAY_NAME "GameOverlay64")
  SET(INJECTOR_NAME "DLLInjection64")
  SET(VULKAN_OVERLAY "VK_LAYER_OCAT_overlay64")

  find_library(Vulkan_LIBRARY
    NAMES vulkan-1
    PATHS
      "$ENV{VULKAN_SDK}/Bin"
      "$ENV{VULKAN_SDK}/Lib"
  )
else(CMAKE_SIZEOF_VOID_P EQUAL 8)
  MESSAGE("32 bits compiler detected")
  SET(EX_PLATFORM 32)
  SET(EX_PLATFORM_NAME "x86")

  SET(GAME_OVERLAY_NAME "GameOverlay32")
  SET(INJECTOR_NAME "DLLInjection32")
  SET(VULKAN_OVERLAY "VK_LAYER_OCAT_overlay32")

  find_library(Vulkan_LIBRARY
    NAMES vulkan-1
    PATHS
      "$ENV{VULKAN_SDK}/Bin32"
      "$ENV{VULKAN_SDK}/Lib32"
  )

endif(CMAKE_SIZEOF_VOID_P EQUAL 8)

IF(EX_PLATFORM EQUAL 64)
MESSAGE("Outputting to lib64 and bin64")

SET (CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE
  ${CMAKE_SOURCE_DIR}/python/game_overlay_sdk/lib
  CACHE PATH
  "Single Directory for all Libraries"
)

SET (CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE
  ${CMAKE_SOURCE_DIR}/python/game_overlay_sdk/lib
  CACHE PATH
  "Single Directory for all Executables."
)

SET (CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE
  ${CMAKE_SOURCE_DIR}/lib/64
  CACHE PATH
  "Single Directory for all static libraries."
)

ELSE( EX_PLATFORM EQUAL 64 )

SET (CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE
  ${CMAKE_SOURCE_DIR}/python/game_overlay_sdk/lib
  CACHE PATH
  "Single Directory for all Libraries"
)

SET (CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE
  ${CMAKE_SOURCE_DIR}/python/game_overlay_sdk/lib
  CACHE PATH
  "Single Directory for all Executables."
)

SET (CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE
   ${CMAKE_SOURCE_DIR}/lib/32
   CACHE PATH
   "Single Directory for all static libraries."
)
ENDIF( EX_PLATFORM EQUAL 64 )

find_path(Vulkan_INCLUDE_DIR
  NAMES vulkan/vulkan.h
  PATHS
    "$ENV{VULKAN_SDK}/Include"
)

################################### NOTE ###############################################
## Injector is compiled without unicode while everything else - with unicode defined! ##
########################################################################################

#---------------- INJECTOR START -----------------#
add_library (
  ${INJECTOR_NAME} SHARED
  src/Utility/Win32Handle.cpp
  src/Utility/StringUtils.cpp
  src/Utility/SuspendThreads.cpp
  src/Utility/FileUtils.cpp
  src/DLLInjection/DLLInjection.cpp
  src/DLLInjection/Monitor.cpp
  src/DLLInjection/MonitorProcessCreation.cpp
)

target_include_directories (${INJECTOR_NAME} PUBLIC
  inc
  src/Utility
  src/DLLInjection/inc
)

#----------------- INJECTOR END -------------------#

#------------------ COMMON START ------------------#
file(GLOB_RECURSE CommonSrc
  "src/DLLToInject/Commons/*.cpp"
  "src/Utility/*.cpp"
)

add_library (
  Commons STATIC
  ${CommonSrc}
)

target_include_directories (Commons PUBLIC
  src
  ${Vulkan_INCLUDE_DIR}
)

target_compile_definitions(Commons PRIVATE
  VK_USE_PLATFORM_WIN32_KHR
  _UNICODE
  UNICODE
)

target_link_libraries(Commons ${Vulkan_LIBRARY})
#-------------------- COMMON END --------------------#

#-------------------- MINHOOK START -----------------#
file(GLOB_RECURSE MinHookSrc "src/DLLToInject/MinHook/*.c")

add_library(
  MinHook STATIC
  ${MinHookSrc}
)
#-------------------- MINHOOK END -------------------#

#----------------- GameOverlay START ----------------#
file(GLOB_RECURSE GameOverlayD3DSrc "src/DLLToInject/GameOverlay/d3d/*.cpp")

add_library (
  ${GAME_OVERLAY_NAME} SHARED
  ${GameOverlayD3DSrc}
  src/DLLToInject/GameOverlay/d3d/exports.def
  src/DLLToInject/GameOverlay/d3d/res/resource.rc
)

target_link_libraries (${GAME_OVERLAY_NAME} MinHook Commons d3d12 d2d1 dwrite ${Vulkan_LIBRARY} comdlg32 advapi32 odbc32 odbccp32)

target_include_directories ( ${GAME_OVERLAY_NAME} PUBLIC
  src
  src/DLLToInject/Commons
  src/DLLToInject/MinHook/include
  src/DLLToInject/LibOVR/include
  src/DLLToInject/openvr/headers
  src/DLLToInject/GameOverlay/d3d/source
  src/DLLToInject/GameOverlay/d3d/deps/d3dx12
)

target_compile_definitions(${GAME_OVERLAY_NAME} PRIVATE
  VK_USE_PLATFORM_WIN32_KHR
  _UNICODE
  UNICODE
)

set_source_files_properties(src/DLLToInject/GameOverlay/d3d/exports.def PROPERTIES HEADER_FILE_ONLY TRUE)

set_target_properties(${GAME_OVERLAY_NAME} PROPERTIES LINK_FLAGS "/DELAYLOAD:d3d12.dll /DEF:\"${CMAKE_SOURCE_DIR}/src/DLLToInject/GameOverlay/d3d/exports.def\"" )

#----------------- GameOverlay END ------------------#

#----------------- VULKAN LAYER START ------------------#
file(GLOB_RECURSE VulkanOverlaySrc "src/DLLToInject/GameOverlay/vulkan/*.cpp")

add_library (
  ${VULKAN_OVERLAY} SHARED
  ${VulkanOverlaySrc}
  src/DLLToInject/GameOverlay/vulkan/VK_LAYER_OCAT_overlay.def
  src/DLLToInject/GameOverlay/vulkan/VK_LAYER_OCAT_overlay.rc
)

target_link_libraries (${VULKAN_OVERLAY} Commons ${GAME_OVERLAY_NAME} d2d1 dwrite comdlg32 advapi32 odbc32 odbccp32)

target_include_directories (${VULKAN_OVERLAY} PUBLIC
  src
  src/DLLToInject/Commons
  ${Vulkan_INCLUDE_DIR}
  "$ENV{VULKAN_SDK}/Source/layers"
  src/DLLToInject/openvr/headers
  src/DLLToInject/GameOverlay/d3d/source
  src/DLLToInject/LibOVR/include
)

target_compile_definitions(${VULKAN_OVERLAY} PRIVATE
  VK_USE_PLATFORM_WIN32_KHR
  _CRT_SECURE_NO_WARNINGS
  API_NAME="Vulkan"
)

set_source_files_properties(src/DLLToInject/GameOverlay/vulkan/VK_LAYER_OCAT_overlay.def PROPERTIES HEADER_FILE_ONLY TRUE)

set_target_properties(${VULKAN_OVERLAY} PROPERTIES LINK_FLAGS "/DEF:\"${CMAKE_SOURCE_DIR}/src/DLLToInject/GameOverlay/vulkan/VK_LAYER_OCAT_overlay.def\"" )

#----------------- VULKAN LAYER END ------------------#