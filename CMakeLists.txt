cmake_minimum_required(VERSION 3.3)
project(boost_cmake)

if(NOT COMMAND find_host_package)
  macro(find_host_package)
    find_package(${ARGN})
  endmacro()
endif()

# macro to find file on the host OS
if(ANDROID)
  macro( find_host_file )
    set( CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER )
    set( CMAKE_FIND_ROOT_PATH_MODE_LIBRARY NEVER )
    set( CMAKE_FIND_ROOT_PATH_MODE_INCLUDE NEVER )
    if( CMAKE_HOST_WIN32 )
      SET( WIN32 1 )
      SET( UNIX )
    elseif( CMAKE_HOST_APPLE )
      SET( APPLE 1 )
      SET( UNIX )
    endif()
    find_file( ${ARGN} )
    SET( WIN32 )
    SET( APPLE )
    SET( UNIX 1 )
    set( CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY )
    set( CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY )
    set( CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY )
  endmacro()
else()
  macro( find_host_file )
    find_file( ${ARGN} )
  endmacro()
endif(ANDROID)

# Tag
find_host_package(Git REQUIRED)

set(BOOST_VERSION 1.60.0)
set(BOOST_ROOT "${CMAKE_BINARY_DIR}/boost_${BOOST_VERSION}")

if(NOT EXISTS ${CMAKE_BINARY_DIR}/boost)
  execute_process(
    COMMAND "${GIT_EXECUTABLE}" clone https://github.com/boostorg/boost.git "${CMAKE_BINARY_DIR}/boost"
    )

  execute_process(
    COMMAND "${GIT_EXECUTABLE}" -C "${CMAKE_BINARY_DIR}/boost" checkout refs/tags/boost-${BOOST_VERSION}
    )

  execute_process(
    COMMAND "${GIT_EXECUTABLE}" -C "${CMAKE_BINARY_DIR}/boost" submodule update --init
    )

  if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    # On Windows, git submodule foreach does not recognize OR(||) syntax.
    execute_process(
      COMMAND "${CMAKE_COMMAND}" -E echo "\"${GIT_EXECUTABLE}\" checkout refs/tags/boost-${BOOST_VERSION} || echo \"checkout failed\""
      OUTPUT_FILE ${CMAKE_BINARY_DIR}/checkout.bat
      )

    execute_process(
      COMMAND "${GIT_EXECUTABLE}" -C "${CMAKE_BINARY_DIR}/boost" submodule foreach ${CMAKE_BINARY_DIR}/checkout.bat
      )
  elseif(${CMAKE_SYSTEM_NAME} MATCHES "((Linux)|(Darwin))")
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" -C "${CMAKE_BINARY_DIR}/boost" submodule foreach '"${GIT_EXECUTABLE}" checkout refs/tags/boost-${BOOST_VERSION} || :'
      )
  endif()
endif()

if(NOT EXISTS ${CMAKE_BINARY_DIR}/boost/b2 AND NOT EXISTS ${CMAKE_BINARY_DIR}/boost/b2.exe)
  if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(BOOST_BOOTSTRAP_COMMAND bootstrap.bat)
  elseif(IOS OR (${CMAKE_SYSTEM_NAME} MATCHES "Darwin"))
    set(BOOST_BOOTSTRAP_COMMAND ./bootstrap.sh --with-toolset=clang)
  elseif(${CMAKE_SYSTEM_NAME} MATCHES "((Linux)|(Android))")
    set(BOOST_BOOTSTRAP_COMMAND ./bootstrap.sh)
  else()
    message(SEND_ERROR "Unsupported CMAKE_SYSTEM_NAME : ${CMAKE_SYSTEM_NAME}")
  endif()
  execute_process(
    COMMAND ${BOOST_BOOTSTRAP_COMMAND}
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )

  if(ANDROID)
    set(BOOST_COMPILER_RULE
      "using gcc : arm : arm-linux-androideabi-g++ ;"
      )
  elseif(IOS OR (CMAKE_CXX_COMPILER_ID MATCHES "Clang"))
    set(BOOST_COMPILER_RULE
      " using clang ;"
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    set(BOOST_COMPILER_RULE
      "using msvc ;"
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(BOOST_COMPILER_RULE
      "using clang : 3.7 : : clang ;"
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(BOOST_COMPILER_RULE
      "using gcc : 4.9.3 : : <archiver>/usr/bin/gcc-ar <ranlib>/usr/bin/gcc-ranlib ;"
      )
  else()
    message(SEND_ERROR "Unsupported compiler CMAKE_CXX_COMPILER_ID: ${CMAKE_CXX_COMPILER_ID}")
  endif()

  configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/project-config.jam.in"
    "${CMAKE_BINARY_DIR}/boost/project-config.jam"
    @ONLY
    )
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

  if(ANDROID)
    set(B2Args
      ./b2
      --prefix=${BOOST_ROOT}
      --build-dir=${CMAKE_BINARY_DIR}/boost/Build
      --with-atomic --with-chrono --with-date_time
      --with-filesystem --with-log --with-program_options
      --with-regex --with-system --with-thread
      toolset=gcc-arm variant=release link=static
      threading=multi threadapi=pthread target-os=android
      define=BOOST_MATH_DISABLE_FLOAT128
      include=/home/mtakeda/etc/arm-linux-androideabi-4.9/include/c++/4.9
      include=$ANDROID_NDK_ROOT/sources/cxx-stl/gnu-libstdc++/4.9/include
      include=$ANDROID_NDK_ROOT/sources/cxx-stl/gnu-libstdc++/4.9/libs/armeabi/include
      include=$ANDROID_NDK_ROOT/platforms/android-19/arch-arm/usr/include
      install
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    set(B2Args
      b2.exe
      --prefix=${BOOST_ROOT}
      --build-dir=${CMAKE_BINARY_DIR}/boost/Build
      --with-atomic --with-chrono --with-date_time
      --with-filesystem --with-log --with-program_options
      --with-regex --with-system --with-thread
      toolset=msvc variant=debug,release link=static
      threading=multi runtime-link=static address-model=64
      install
      )
  elseif(IOS)
    # For IOS we need to build for both arm and simulator(x86).
    execute_process(
      COMMAND ${CMAKE_COMMAND} -E echo "\
#!/bin/sh
./b2 --prefix=${BOOST_ROOT} --build-dir=${CMAKE_BINARY_DIR}/boost/Build \
 --libdir=${BOOST_ROOT}/lib/iphone_arm \
 --with-atomic --with-chrono --with-date_time \
 --with-filesystem --with-log --with-program_options --with-regex --with-system --with-thread \
 toolset=clang cflags=\"-arch armv7 -arch armv7s -arch arm64\" \
 cxxflags=-std=c++11 cxxflags=-stdlib=libc++ cxxflags=-fvisibility=hidden cxxflags=-fvisibility-inlines-hidden \
 linkflags=-stdlib=libc++ link=static architecture=arm target-os=iphone threading=multi define=_LITTLE_ENDIAN \
 variant=release \
 include=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS7.1.sdk/usr/include \
 install

./b2 --prefix=${BOOST_ROOT} --build-dir=${CMAKE_BINARY_DIR}/boost/Build \
 --libdir=${BOOST_ROOT}/lib/ios \
 --with-atomic --with-chrono --with-date_time \
 --with-filesystem --with-log --with-program_options --with-regex --with-system --with-thread \
 toolset=clang cflags=\"-arch armv7 -arch armv7s -arch arm64\" \
 cxxflags=-std=c++11 cxxflags=-stdlib=libc++ cxxflags=-fvisibility=hidden cxxflags=-fvisibility-inlines-hidden \
 linkflags=-stdlib=libc++ link=static architecture=arm target-os=iphone threading=multi define=_LITTLE_ENDIAN \
 variant=release \
 include=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS7.1.sdk/usr/include \
 --clean

./b2 \
 --prefix=${BOOST_ROOT} \
 --build-dir=${CMAKE_BINARY_DIR}/boost/Build \
 --libdir=${BOOST_ROOT}/lib/iphone_sim \
 --with-atomic --with-chrono --with-date_time \
 --with-filesystem --with-log --with-program_options \
 --with-regex --with-system --with-thread \
 toolset=clang \
 cflags=\"-arch i386 -arch x86_64\" \
 cxxflags=-std=c++11 cxxflags=-stdlib=libc++ cxxflags=-fvisibility=hidden cxxflags=-fvisibility-inlines-hidden \
 linkflags=-stdlib=libc++ link=static architecture=arm target-os=iphone threading=multi define=_LITTLE_ENDIAN \
 variant=release \
 include=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneSimulator7.1.sdk/usr/include \
 install

./b2 \
 --prefix=${BOOST_ROOT} \
 --build-dir=${CMAKE_BINARY_DIR}/boost/Build \
 --libdir=${BOOST_ROOT}/lib/iphone_sim \
 --with-atomic --with-chrono --with-date_time \
 --with-filesystem --with-log --with-program_options \
 --with-regex --with-system --with-thread \
 toolset=clang \
 cflags=\"-arch i386 -arch x86_64\" \
 cxxflags=-std=c++11 cxxflags=-stdlib=libc++ cxxflags=-fvisibility=hidden cxxflags=-fvisibility-inlines-hidden \
 linkflags=-stdlib=libc++ link=static architecture=arm target-os=iphone threading=multi define=_LITTLE_ENDIAN \
 variant=release \
 include=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneSimulator7.1.sdk/usr/include \
 --clean

for li in `find ${BOOST_ROOT}/lib/iphone_arm -name \"*.a\"`; do
FILENAME=$(basename $li \"${BOOST_ROOT}/lib/iphone_arm\")
lipo -create \"${BOOST_ROOT}/lib/iphone_arm/$FILENAME\" \"${BOOST_ROOT}/lib/iphone_sim/$FILENAME\" -output \"${BOOST_ROOT}/lib/$FILENAME\"
done

rm -rf ${BOOST_ROOT}/lib/iphone_arm
rm -rf ${BOOST_ROOT}/lib/iphone_sim \
"
OUTPUT_FILE build_boost.sh
)

    execute_process(
      COMMAND chmod 744 build_boost.sh
      )
    set(B2Args
      ${CMAKE_BINARY_DIR}/build_boost.sh
      )
  elseif(CMAKE_SYSTEM_NAME MATCHES "Darwin")
    set(B2Args
      ./b2
      --prefix=${BOOST_ROOT}
      --build-dir=${CMAKE_BINARY_DIR}/boost/Build
      --with-atomic --with-chrono --with-date_time
      --with-filesystem --with-log --with-program_options
      --with-regex --with-system --with-thread
      toolset=clang
      cflags=-arch\ i386\ -arch\ x86_64
      cxxflags=-fvisibility=hidden cxxflags=-fvisibility-inlines-hidden
      cxxflags=-std=c++11 cxxflags=-stdlib=libc++ cxxflags=-ftemplate-depth=512
      linkflags="-stdlib=libc++" link=static
      install
      )
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(B2Args
      ./b2
      --prefix=${BOOST_ROOT}
      --build-dir=${CMAKE_BINARY_DIR}/boost/Build
      --with-atomic --with-chrono --with-date_time
      --with-filesystem --with-log --with-program_options
      --with-regex --with-system --with-thread
      toolset=clang-3.7 variant=release link=static
      threading=multi runtime-link=static address-model=64
      cxxflags=-std=c++14 cxxflags=-fPIC cxxflags=-stdlib=libstdc++
      install
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

  message(STATUS "B2Args =" "${B2Args}")
  execute_process(
    COMMAND ${B2Args}
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )
endif()

add_library(boost_cmake INTERFACE)

find_host_file(BOOST_INCLUDEDIR
  boost
  PATHS ${BOOST_ROOT}/include
  PATH_SUFFIXES
  boost-1_60 # On Windows, b2.exe install header files in {prefix}/include/boost_{version}
  NO_DEFAULT_PATH
  )

if(NOT EXISTS ${BOOST_INCLUDEDIR})
  message(SEND_ERROR "BOOST_INCLUDEDIR could not be found in the BOOST_ROOT, ${BOOST_ROOT}")
endif()

get_filename_component(
  BOOST_INCLUDEDIR
  ${BOOST_INCLUDEDIR}
  DIRECTORY
  )

message(STATUS "BOOST_INCLUDEDIR =" "${BOOST_INCLUDEDIR}")

set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME ON)
set(Boost_USE_DEBUG_RUNTIME ON)
set(Boost_NO_SYSTEM_PATHS ON)
set(BOOST_LIBRARYDIR "${BOOST_ROOT}/lib")

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
  ${Boost_THREAD_LIBRARY}
  )
