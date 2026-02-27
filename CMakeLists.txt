set(BUNGEE_VERSION 0.0.0 CACHE STRING "Bungee version string")
set(BUNGEE_SELF_TEST 0 CACHE STRING "Enable Bungee self tests (0=off, 1=fast, 2=full)")
option(BUNGEE_BUILD_SHARED_LIBRARY "Build shared, dynamic Bungee library in addition to a static libary" ON)
option(BUNGEE_INSTALL_FRAMEWORK "Install as a framework" OFF)
set(BUNGEE_PRESET "" CACHE STRING "Name of the preset that we're building")

if (BUNGEE_BUILD_SHARED_LIBRARY)
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

cmake_minimum_required(VERSION 3.30...3.31)
include(CheckCXXCompilerFlag)

project(bungee VERSION ${BUNGEE_VERSION})
set(CMAKE_CXX_STANDARD 20)

if (WIN32)
  set(BUNGEE_EXPORT "__declspec(dllexport)")
else()
  set(BUNGEE_EXPORT "__attribute__((visibility(\"default\")))")
endif()

# Bungee library target: "bungee_library"
file(GLOB BUNGEE_LIBRARY_HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/bungee/*.h")
file(GLOB BUNGEE_SOURCE_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
if (BUNGEE_BUILD_SHARED_LIBRARY)
  add_library(bungee_library SHARED ${BUNGEE_LIBRARY_HEADERS} ${BUNGEE_SOURCE_FILES})
else()
  add_library(bungee_library STATIC ${BUNGEE_LIBRARY_HEADERS} ${BUNGEE_SOURCE_FILES})
endif()  
target_include_directories(bungee_library PRIVATE  submodules/eigen submodules .)
target_compile_definitions(bungee_library PRIVATE
  BUNGEE_VISIBILITY=${BUNGEE_EXPORT}
  BUNGEE_SELF_TEST=${BUNGEE_SELF_TEST}
  eigen_assert=BUNGEE_ASSERT1
  EIGEN_DONT_PARALLELIZE=1
  $<$<CONFIG:Debug>:EIGEN_RUNTIME_NO_MALLOC=1>
)
set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/src/version.cpp PROPERTIES COMPILE_DEFINITIONS BUNGEE_VERSION=\"${BUNGEE_VERSION}\")
target_compile_options(bungee_library PRIVATE $<$<CXX_COMPILER_ID:GNU,Clang>:-fwrapv>)
target_link_libraries(bungee_library PRIVATE pffft)
if (ANDROID)
  target_link_libraries(bungee_library PRIVATE log)
endif()
set_target_properties(bungee_library PROPERTIES OUTPUT_NAME bungee)

# Bungee command-line executable target: "bungee_executable"
add_executable(bungee_executable cmd/main.cpp)
target_include_directories(bungee_executable PRIVATE submodules/cxxopts/include .)
target_link_libraries(bungee_executable PRIVATE bungee_library)
set_target_properties(bungee_executable PROPERTIES 
  OUTPUT_NAME bungee
  INSTALL_RPATH "${CMAKE_INSTALL_RPATH};${CMAKE_INSTALL_PREFIX}"
)

# PFFFT as a static library
add_library(pffft EXCLUDE_FROM_ALL STATIC
  submodules/pffft/pffft.c
  submodules/pffft/fftpack.c
)
target_compile_options(pffft PRIVATE -ffast-math -fno-finite-math-only -fno-exceptions)

# Installation
if (BUNGEE_BUILD_SHARED_LIBRARY)
  if (BUNGEE_INSTALL_FRAMEWORK)
    set_target_properties(bungee_library PROPERTIES
      FRAMEWORK TRUE
      XCODE_ATTRIBUTE_CODE_SIGN_IDENTITY "iPhone Developer"
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${BUNGEE_VERSION}
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${BUNGEE_VERSION}
      VERSION ${BUNGEE_VERSION}
    )
    set_source_files_properties(
      ${BUNGEE_LIBRARY_HEADERS} 
      PROPERTIES
      MACOSX_PACKAGE_LOCATION Headers/bungee
    )
  endif()
  install(FILES 
    ${BUNGEE_LIBRARY_HEADERS}
    DESTINATION ${CMAKE_INSTALL_PREFIX}/include/bungee
  )
  install(TARGETS bungee_library bungee_executable
    ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/${BUNGEE_PRESET}
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/${BUNGEE_PRESET}
    RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/${BUNGEE_PRESET}
    BUNDLE DESTINATION ${CMAKE_INSTALL_PREFIX}/${BUNGEE_PRESET}
    FRAMEWORK DESTINATION ${CMAKE_INSTALL_PREFIX}/${BUNGEE_PRESET}
  )
  install(FILES 
    cmd/main.cpp  
    submodules/cxxopts/include/cxxopts.hpp
    DESTINATION ${CMAKE_INSTALL_PREFIX}/sample
  )

get_property(BUNGEE_LIBRARY_OUTPUT_NAME TARGET bungee_library PROPERTY OUTPUT_NAME)

if (APPLE)
  set(PKGCONFIG_LINK_PATH -F)
  set(PKGCONFIG_LINK_FLAG "-framework ")
else()
  set(PKGCONFIG_LINK_PATH -L)
  set(PKGCONFIG_LINK_FLAG "-l")
endif()
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libbungee.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libbungee.pc @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libbungee.pc DESTINATION ${CMAKE_INSTALL_PREFIX}/${BUNGEE_PRESET}/pkgconfig)

endif()
