cmake_minimum_required(VERSION 3.15)
project(soloud LANGUAGES C CXX)

# === Options ===
option(SOLOUD_BUILD_STATIC "Build static library" ON)
option(SOLOUD_BUILD_SHARED "Build shared library" OFF)
option(SOLOUD_BUILD_DEMOS "Build SoLoud demo applications" OFF)
option(SOLOUD_GENERATE_GLUE "Generate glue code for language bindings" OFF)
option(SOLOUD_C_API "Enable C API" OFF)

option(SOLOUD_BACKEND_NULL "Enable Null backend" OFF)
option(SOLOUD_BACKEND_SDL2 "Enable SDL2 backend" OFF)
option(SOLOUD_BACKEND_ALSA "Enable ALSA backend" OFF)
option(SOLOUD_BACKEND_COREAUDIO "Enable CoreAudio backend (macOS only)" OFF)
option(SOLOUD_BACKEND_OPENSLES "Enable OpenSLES backend (Android only)" OFF)
option(SOLOUD_BACKEND_XAUDIO2 "Enable XAudio2 backend (Windows)" OFF)
option(SOLOUD_BACKEND_WINMM "Enable WinMM backend (Windows)" OFF)
option(SOLOUD_BACKEND_WASAPI "Enable WASAPI backend (Windows)" OFF)

# === Paths ===
set(SOLOUD_ROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(SOLOUD_INCLUDE_DIR ${SOLOUD_ROOT}/include)
set(SOLOUD_SRC_DIR ${SOLOUD_ROOT}/src)

file(GLOB_RECURSE SOLOUD_HEADERS CONFIGURE_DEPENDS "${SOLOUD_INCLUDE_DIR}/soloud/*.h")
file(GLOB_RECURSE SOLOUD_SOURCES CONFIGURE_DEPENDS
  "${SOLOUD_SRC_DIR}/core/*.cpp"
  "${SOLOUD_SRC_DIR}/filter/*.cpp"
  "${SOLOUD_SRC_DIR}/audiosource/**/*.cpp"
)

set(BACKEND_SOURCES "")
set(EXTRA_LIBS "")

# === Backend Selection ===
if(SOLOUD_BACKEND_NULL)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/null/soloud_null.cpp)
  add_compile_definitions(WITH_NULL)
endif()

if(SOLOUD_BACKEND_SDL2)
  find_package(SDL2 REQUIRED)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/sdl2_static/soloud_sdl2_static.cpp)
  list(APPEND EXTRA_LIBS SDL2::SDL2)
  add_compile_definitions(WITH_SDL2_STATIC)
endif()

if(SOLOUD_BACKEND_ALSA)
  find_library(ALSA_LIBRARY asound REQUIRED)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/alsa/soloud_alsa.cpp)
  list(APPEND EXTRA_LIBS ${ALSA_LIBRARY})
  add_compile_definitions(WITH_ALSA)
endif()

if(SOLOUD_BACKEND_COREAUDIO AND APPLE)
  find_library(AUDIOTOOLBOX_FRAMEWORK AudioToolbox REQUIRED)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/coreaudio/soloud_coreaudio.cpp)
  list(APPEND EXTRA_LIBS ${AUDIOTOOLBOX_FRAMEWORK})
  add_compile_definitions(WITH_COREAUDIO)
endif()

if(SOLOUD_BACKEND_OPENSLES)
  find_library(OPENSLES_LIBRARY OpenSLES REQUIRED)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/opensles/soloud_opensles.cpp)
  list(APPEND EXTRA_LIBS ${OPENSLES_LIBRARY})
  add_compile_definitions(WITH_OPENSLES)
endif()

if(SOLOUD_BACKEND_XAUDIO2)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/xaudio2/soloud_xaudio2.cpp)
  add_compile_definitions(WITH_XAUDIO2)
endif()

if(SOLOUD_BACKEND_WINMM)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/winmm/soloud_winmm.cpp)
  add_compile_definitions(WITH_WINMM)
endif()

if(SOLOUD_BACKEND_WASAPI)
  list(APPEND BACKEND_SOURCES ${SOLOUD_SRC_DIR}/backend/wasapi/soloud_wasapi.cpp)
  add_compile_definitions(WITH_WASAPI)
endif()

# === C API ===
if(SOLOUD_C_API)
  list(APPEND SOLOUD_SOURCES ${SOLOUD_SRC_DIR}/c_api/soloud_c.cpp)
  list(APPEND SOLOUD_HEADERS ${SOLOUD_INCLUDE_DIR}/soloud/soloud_c.h)
endif()

# === Final sources ===
set(FINAL_SOURCES ${SOLOUD_SOURCES} ${BACKEND_SOURCES} ${SOLOUD_HEADERS})

# === Library Targets ===
if(SOLOUD_BUILD_STATIC)
  add_library(soloud_static STATIC ${FINAL_SOURCES})
  target_compile_definitions(soloud_static PRIVATE _CRT_SECURE_NO_WARNINGS)
  target_include_directories(soloud_static PUBLIC ${SOLOUD_INCLUDE_DIR} ${SOLOUD_SRC_DIR})
  target_link_libraries(soloud_static PRIVATE ${EXTRA_LIBS})
  add_library(soloud::soloud ALIAS soloud_static)
endif()

if(SOLOUD_BUILD_SHARED)
  add_library(soloud_shared SHARED ${FINAL_SOURCES})
  target_compile_definitions(soloud_shared PRIVATE _CRT_SECURE_NO_WARNINGS)
  target_include_directories(soloud_shared PUBLIC ${SOLOUD_INCLUDE_DIR} ${SOLOUD_SRC_DIR})
  target_link_libraries(soloud_shared PRIVATE ${EXTRA_LIBS})
  add_library(soloud::soloud ALIAS soloud_shared)
endif()

# === Optional Demos ===
if(SOLOUD_BUILD_DEMOS)
  add_subdirectory(${SOLOUD_ROOT}/demos demos)
endif()

# === Optional Glue ===
if(SOLOUD_GENERATE_GLUE)
  add_subdirectory(${SOLOUD_SRC_DIR}/glue glue)
endif()

# === Install ===
install(DIRECTORY ${SOLOUD_INCLUDE_DIR}/soloud DESTINATION include)
if(SOLOUD_BUILD_STATIC)
  install(TARGETS soloud_static EXPORT soloudTargets ARCHIVE DESTINATION lib)
endif()
if(SOLOUD_BUILD_SHARED)
  install(TARGETS soloud_shared EXPORT soloudTargets RUNTIME DESTINATION bin LIBRARY DESTINATION lib)
endif()

# === CMake Package ===
install(EXPORT soloudTargets NAMESPACE soloud:: DESTINATION lib/cmake/soloud)
include(CMakePackageConfigHelpers)
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/SoloudConfigVersion.cmake VERSION 1.0 COMPATIBILITY AnyNewerVersion)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/SoloudConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/SoloudConfig.cmake @ONLY)
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/SoloudConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/SoloudConfigVersion.cmake
  DESTINATION lib/cmake/soloud
)

message(STATUS "\n✅ SoLoud is configured and ready to sing 🎶")
