# Generate version file
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/project_version.h.in
               ${CMAKE_CURRENT_BINARY_DIR}/project_version.h @ONLY)

add_library(bergamot-translator STATIC
    byte_array_util.cpp
    text_processor.cpp
    translation_model.cpp 
    request.cpp 
    batching_pool.cpp
    aggregate_batching_pool.cpp
    response_builder.cpp
    quality_estimator.cpp
    batch.cpp
    annotation.cpp
    service.cpp
    parser.cpp
    response.cpp
    html.cpp
    xh_scanner.cpp
)
if (USE_WASM_COMPATIBLE_SOURCE)
  # Using wasm compatible sources should include this compile definition;
  # Has to be done here because we are including marian headers + some sources
  # in local repository use these definitions
  target_compile_definitions(bergamot-translator PUBLIC USE_SSE2 WASM_COMPATIBLE_SOURCE)
endif()

if(COMPILE_WASM)
  target_compile_definitions(bergamot-translator PUBLIC WASM)
  # Enable code that is required for generating JS bindings
  target_compile_definitions(bergamot-translator PRIVATE WASM_BINDINGS)
  target_compile_options(bergamot-translator PRIVATE ${WASM_COMPILE_FLAGS})
  target_link_options(bergamot-translator PRIVATE ${WASM_LINK_FLAGS})
endif(COMPILE_WASM)

if(ENABLE_CACHE_STATS)
    target_compile_definitions(bergamot-translator PUBLIC ENABLE_CACHE_STATS)
endif(ENABLE_CACHE_STATS)

find_package(PkgConfig QUIET)

find_package(sentencepiece CONFIG QUIET)
if(sentencepiece_FOUND)
  set(SENTENCEPIECE_TARGET sentencepiece::sentencepiece)
elseif(NOT WIN32)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(sentencepiece REQUIRED IMPORTED_TARGET sentencepiece)
  set(SENTENCEPIECE_TARGET PkgConfig::sentencepiece)
else()
  find_path(SENTENCEPIECE_INCLUDE_DIR
    NAMES sentencepiece_processor.h
    PATHS ${CMAKE_PREFIX_PATH}
    PATH_SUFFIXES include
  )

  find_library(SENTENCEPIECE_LIBRARY
    NAMES sentencepiece
    PATHS ${CMAKE_PREFIX_PATH}
    PATH_SUFFIXES lib
  )

  find_library(SENTENCEPIECE_TRAIN_LIBRARY
      NAMES sentencepiece_train
  	PATHS ${CMAKE_PREFIX_PATH}
  	PATH_SUFFIXES lib
  )

  if(SENTENCEPIECE_INCLUDE_DIR AND SENTENCEPIECE_LIBRARY)
    add_library(sentencepiece::sentencepiece UNKNOWN IMPORTED)
    set_target_properties(sentencepiece::sentencepiece PROPERTIES
      IMPORTED_LOCATION "${SENTENCEPIECE_LIBRARY}"
      INTERFACE_INCLUDE_DIRECTORIES "${SENTENCEPIECE_INCLUDE_DIR}"
    )

    if(SENTENCEPIECE_TRAIN_LIBRARY)
      set_property(TARGET sentencepiece::sentencepiece APPEND PROPERTY
      INTERFACE_LINK_LIBRARIES "${SENTENCEPIECE_TRAIN_LIBRARY}"
    )
    endif(SENTENCEPIECE_TRAIN_LIBRARY)

    set(SENTENCEPIECE_TARGET sentencepiece::sentencepiece)
    set(SENTENCEPIECE_INCLUDE_DIRS "${SENTENCEPIECE_INCLUDE_DIR}")
  else()
    if(PkgConfig_FOUND)
      pkg_check_modules(sentencepiece QUIET IMPORTED_TARGET sentencepiece)
      if(sentencepiece_FOUND)
        set(SENTENCEPIECE_TARGET PkgConfig::sentencepiece)
      else()
        message(FATAL_ERROR "sentencepiece not found. Please install it via vcpkg.")
      endif()
    else()
      message(FATAL_ERROR "sentencepiece not found. Please install it via vcpkg.")
    endif()
  endif()
endif()

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

find_package(intgemm CONFIG QUIET)
find_package(simd_utils REQUIRED CONFIG)
find_package(ruy REQUIRED CONFIG)
find_package(marian-dev REQUIRED CONFIG)
find_package(ssplit REQUIRED CONFIG)
find_package(protobuf REQUIRED CONFIG)
find_package(absl CONFIG QUIET)

target_link_libraries(bergamot-translator marian-dev::marian-dev ssplit::ssplit)

target_include_directories(bergamot-translator
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/..>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/bergamot-translator>
)

install(TARGETS bergamot-translator
    EXPORT bergamot_translatorTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(EXPORT bergamot_translatorTargets
    NAMESPACE bergamot-translator::
    DESTINATION share/bergamot-translator
    FILE bergamot_translatorTargets.cmake)

configure_package_config_file(
    "${CMAKE_SOURCE_DIR}/bergamot_translatorConfig.cmake.in"
    "${CMAKE_SOURCE_DIR}/bergamot-translatorConfig.cmake"
    INSTALL_DESTINATION share/bergamot-translator)

write_basic_package_version_file(
    "${CMAKE_SOURCE_DIR}/bergamot-translatorConfigVersion.cmake"
    VERSION 1.0.0
    COMPATIBILITY SameMajorVersion)

install(FILES
    "${CMAKE_SOURCE_DIR}/bergamot-translatorConfig.cmake"
    "${CMAKE_SOURCE_DIR}/bergamot-translatorConfigVersion.cmake"
    DESTINATION share/bergamot-translator)

install(
    DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/bergamot-translator/translator
    FILES_MATCHING
        PATTERN "*.h"
	PATTERN "threadsafe_batching_pool.cpp"
)

