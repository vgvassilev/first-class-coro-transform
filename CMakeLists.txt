# If we are not building as a part of LLVM, build FCCT as an
# standalone project, using LLVM as an external library:
if( CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR )
  project(first_class_coro_transform) # FCCT
  cmake_minimum_required(VERSION 3.4.3)
  find_package(Subversion)

  find_package(LLVM 5.0 REQUIRED CONFIG PATHS ${FCCT_PATH_TO_LLVM_BUILD} NO_DEFAULT_PATH)

  # We cannot use find_package(Clang) or include(ClangConfig) because we require
  # a specific version (currently 5.0) and ClangConfig.cmake tries to do
  # find_package(LLVM REQUIRED CONFIG...) which in certain cases resolves to the
  # wrong LLVM version.
  # FIXME: That may be a bug in ClangConfig.cmake.

  if(NOT LLVM_FOUND)
    message(FATAL "Please set FCCT_PATH_TO_LLVM_BUILD pointing to the llvm build or installation folder")
  endif()

  # In case this was a path to a build folder of llvm still try to find AddLLVM
  list(APPEND CMAKE_MODULE_PATH "${LLVM_DIR}")

  include(AddLLVM)
  include(HandleLLVMOptions)

  set(CMAKE_INCLUDE_CURRENT_DIR ON)

  # In rare cases we might want to have clang installed in a different place
  # than llvm and the header files should be found first (even though the
  # LLVM_INCLUDE_DIRS) contain clang headers, too.
  include_directories(${CLANG_INCLUDE_DIRS})
  include_directories(${LLVM_INCLUDE_DIRS})
  add_definitions(${LLVM_DEFINITIONS})

  # If the llvm sources are present add them with higher priority.
  if(LLVM_BUILD_MAIN_SRC_DIR)
    # LLVM_INCLUDE_DIRS contains the include paths to both LLVM's source and
    # build directories. Since we cannot just include ClangConfig.cmake (see
    # fixme above) we have to do a little more work to get the right include
    # paths for clang.
    #
    # FIXME: We only support in-tree builds of clang, that is clang being built
    # in llvm_src/tools/clang.
    include_directories(${LLVM_BUILD_MAIN_SRC_DIR}/tools/clang/include/)

    if(NOT LLVM_BUILD_BINARY_DIR)
      message(FATAL "LLVM_BUILD_* values should be available for the build tree")
    endif()

    include_directories(${LLVM_BUILD_BINARY_DIR}/tools/clang/include/)
  endif()

  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib/)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin/)

  set( FCCT_BUILT_STANDALONE 1 )
endif()

set(C_INCLUDE_DIRS "" CACHE STRING
  "Colon separated list of directories FCCT will search for headers.")

set(FCCT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(FCCT_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR})

if( CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR AND NOT MSVC_IDE )
  message(FATAL_ERROR "In-source builds are not allowed. CMake would overwrite "
"the makefiles distributed with LLVM. Please create a directory and run cmake "
"from there, passing the path to this source directory as the last argument. "
"This process created the file `CMakeCache.txt' and the directory "
"`CMakeFiles'. Please delete them.")
endif()

# Add appropriate flags for GCC
if (LLVM_COMPILER_IS_GCC_COMPATIBLE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-common -Woverloaded-virtual -Wcast-qual -fno-strict-aliasing -pedantic -Wno-long-long -Wall -W -Wno-unused-parameter -Wwrite-strings")
endif ()

if (APPLE)
  set(CMAKE_MODULE_LINKER_FLAGS "-Wl,-flat_namespace -Wl,-undefined -Wl,suppress")
endif ()

# FIXME: Use merge this with the content from the LLVMConfig and ClangConfig.
if (NOT FCCT_BUILT_STANDALONE)
include_directories(BEFORE
  ${CMAKE_CURRENT_BINARY_DIR}/../clang/include
  ${CMAKE_CURRENT_SOURCE_DIR}/../clang/include
  )
endif()

include_directories(BEFORE
  ${CMAKE_CURRENT_BINARY_DIR}/include
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  )

install(DIRECTORY include/
  DESTINATION include
  FILES_MATCHING
  PATTERN "*.def"
  PATTERN "*.h"
  PATTERN ".svn" EXCLUDE
  )

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/include/
  DESTINATION include
  FILES_MATCHING
  PATTERN "CMakeFiles" EXCLUDE
  PATTERN "*.inc"
  )

add_definitions( -D_GNU_SOURCE
  -DFCCT_SRCDIR_INCL="${FCCT_SOURCE_DIR}/include"
  -DFCCT_INSTDIR_INCL="${FCCT_BINARY_DIR}/include" )

include(VersionFromVCS)
# SVN_REVISION and GIT_COMMIT get set by the call to add_version_info_from_vcs.
# DUMMY_VAR contains a version string which we don't care about.
add_version_info_from_vcs(DUMMY_VAR)

option(FCCT_BUILD_STATIC_ONLY "Does not build shared libraries. Useful when we have LLVM_ENABLE_PLUGINS=OFF (eg. Win or CYGWIN)" OFF)
if (NOT FCCT_BUILD_STATIC_ONLY AND NOT LLVM_ENABLE_PLUGINS)
  message(FATAL_ERROR "LLVM_ENABLE_PLUGINS is set to OFF. Please build FCCT with -DFCCT_BUILD_STATIC_ONLY=ON.")
endif()

add_subdirectory(lib)
add_subdirectory(tools)

# TODO: docs.
if (NOT FCCT_BUILD_STATIC_ONLY)
  add_subdirectory(test)
endif()

# Workaround for MSVS10 to avoid the Dialog Hell
# FIXME: This could be removed with future version of CMake.
if( FCCT_BUILT_STANDALONE AND MSVC_VERSION EQUAL 1600 )
  set(FCCT_SLN_FILENAME "${CMAKE_CURRENT_BINARY_DIR}/FCCT.sln")
  if( EXISTS "${FCCT_SLN_FILENAME}" )
    file(APPEND "${FCCT_SLN_FILENAME}" "\n# This should be regenerated!\n")
  endif()
endif()
