cmake_minimum_required(VERSION 3.3)
project(boost_cmake)

if(NOT COMMAND find_host_package)
  macro(find_host_package)
    find_package(${ARGN})
  endmacro()
endif()

# Tag
find_package(Git REQUIRED)

set(BOOST_VERSION 1.60.0)
set(BOOST_ROOT "${CMAKE_BINARY_DIR}/boost_${BOOST_VERSION}")

if(NOT EXISTS ${CMAKE_BINARY_DIR}/boost)
  execute_process(
    COMMAND "${GIT_EXECUTABLE}" clone https://github.com/boostorg/boost.git "${CMAKE_BINARY_DIR}/boost"
    )
  execute_process(
    COMMAND "${GIT_EXECUTABLE}" -C "${CMAKE_BINARY_DIR}/boost" submodule update --init
    )
  execute_process(
    COMMAND "${GIT_EXECUTABLE}" -C "${CMAKE_BINARY_DIR}/boost" submodule foreach '"${GIT_EXECUTABLE}" checkout refs/tags/boost-${BOOST_VERSION} || :' 
    )
endif()

if(NOT EXISTS ${CMAKE_BINARY_DIR}/boost/b2)
  if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(BOOST_BOOTSTRAP_COMMAND bootstrap.bat)
  elseif(${CMAKE_SYSTEM_NAME} MATCHES "((Linux)|(Darwin))")
    set(BOOST_BOOTSTRAP_COMMAND ./bootstrap.sh)
  else()
    message(SEND_ERROR "Unsupported system: CMAKE_SYSTEM_NAME should be ${CMAKE_SYSTEM_NAME}")
  endif()
  execute_process(
    COMMAND ${BOOST_BOOTSTRAP_COMMAND}
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )

  if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    message(STATUS "CMAKE_CXX_COMPILER_ID =" "${CMAKE_CXX_COMPILER_ID}")
    set(BOOST_COMPILER_RULE
      "using msvc ;"
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(BOOST_COMPILER_RULE
      "using gcc : 4.9.3 : : <archiver>/usr/bin/gcc-ar <ranlib>/usr/bin/gcc-ranlib ;"
      )
  else()
    message(SEND_ERROR "Unsupported compiler: CMAKE_CXX_COMPILER_ID should be ${CMAKE_CXX_COMPILER_ID}")
  endif()

  # configure_file(
  #   "${CMAKE_CURRENT_SOURCE_DIR}/project-config.jam.in"
  #   "${CMAKE_BINARY_DIR}/boost/project-config.jam"
  #   @ONLY
  #   )
endif()

if(NOT EXISTS ${BOOST_ROOT})
  execute_process(
    COMMAND ${CMAKE_COMMAND} -E make_directory Build
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )
  execute_process(
    COMMAND ./b2 headers
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )

  if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    set(B2Args
      "\
b2.exe --prefix=${BOOST_ROOT} --build-dir=${CMAKE_BINARY_DIR}/boost/Build \
--with-atomic --with-chrono --with-date_time \
--with-filesystem --with-log --with-program_options \
--with-regex --with-system --with-thread \
toolset=msvc variant=release link=static \
threading=multi runtime-link=static address-model=64 \
install"
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(B2Args
      ./b2 
      --prefix=${BOOST_ROOT}
      --build-dir=${CMAKE_BINARY_DIR}/boost/Build
      --with-atomic --with-chrono --with-date_time
      --with-filesystem --with-log --with-program_options
      --with-regex --with-system --with-thread
      toolset=gcc-4.9.3 variant=release link=static
      threading=multi runtime-link=static address-model=64
      cxxflags=-std=c++14 cxxflags=-flto cxxflags=-fPIC cxxflags=-fuse-linker-plugin
      linkflags=-flto linkflags=-fuse-linker-plugin
      install
      )
  else()
    message(SEND_ERROR "Unsupported compiler: CMAKE_CXX_COMPILER_ID should be ${CMAKE_CXX_COMPILER_ID}")
  endif()

  # separate_arguments(B2Args)
  message(STATUS "B2Args =" "${B2Args}")
  execute_process(
    COMMAND ${B2Args}
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )
endif()

add_library(boost_cmake INTERFACE)

set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME ON)
set(Boost_USE_DEBUG_RUNTIME ON)
set(BOOST_NO_SYSTEM_PATHS ON)

find_host_package(
  Boost
  ${BOOST_VERSION}
  REQUIRED
  COMPONENTS
  chrono
  date_time
  filesystem
  log
  atomic
  program_options
  regex
  system
  thread
  )

target_include_directories(
  boost_cmake
  INTERFACE
  ${Boost_INCLUDE_DIRS}
  )

target_link_libraries(
  boost_cmake
  INTERFACE
  ${Boost_CHRONO_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_LOG_LIBRARY}
  ${Boost_PROGRAM_OPTIONS_LIBRARY}
  ${Boost_REGEX_LIBRARY}
  ${Boost_DATE_TIME_LIBRARY}
  ${Boost_SYSTEM_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_THREAD_LIBRARY}
  )
