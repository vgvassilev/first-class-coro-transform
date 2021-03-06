# Test runner infrastructure for fcct (copied and adapted from Clang). This
# configures the fcct test trees for use by Lit, and delegates to LLVM's lit
# test handlers.

if (CMAKE_CFG_INTDIR STREQUAL ".")
  set(LLVM_BUILD_MODE ".")
else ()
  set(LLVM_BUILD_MODE "%(build_mode)s")
endif ()

# Needed for '%shlibext'
set(TARGET_SHLIBEXT "${CMAKE_SHARED_LIBRARY_SUFFIX}")

# Define variables which cmake will export for lit. Typically in an installation
# they are not defined or defined in the wrong way.
if(FCCT_BUILT_STANDALONE)
  set(LLVM_SOURCE_DIR "")
  # cmake will fall back to LLVM-config (asking llvm-config --obj-root) and this
  # will return a build folder. This information will force lit to look for
  # utils such as FileCheck in /usr/local/... instead of the true location in
  # /usr/lib/llvm-5.0/
  set(LLVM_BINARY_DIR ${LLVM_INSTALL_PREFIX})
  set(LLVM_TOOLS_DIR ${LLVM_INSTALL_PREFIX})
  set(LLVM_LIBS_DIR ${LLVM_INSTALL_PREFIX}/lib/)
endif()

configure_lit_site_cfg(
  ${CMAKE_CURRENT_SOURCE_DIR}/lit.site.cfg.in
  ${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg
  )

option(CLANG_TEST_USE_VG "Run Clang tests under Valgrind" OFF)
if(CLANG_TEST_USE_VG)
  set(CLANG_TEST_EXTRA_ARGS ${CLANG_TEST_EXTRA_ARGS} "--vg")
endif ()

list(APPEND FCCT_TEST_DEPS fcct)
# Try to append dependencies only if we are building in-tree.
if(NOT FCCT_BUILT_STANDALONE)
  list(APPEND FCCT_TEST_DEPS llvm-config FileCheck clang opt count not)
endif()

set(FCCT_TEST_PARAMS
  fcct_site_config=${CMAKE_CURRENT_BINARY_DIR}/lit.site.cfg
  )

add_custom_target(fcct-test-depends DEPENDS ${FCCT_TEST_DEPS})
set_target_properties(fcct-test-depends PROPERTIES FOLDER "FCCT tests")

# Set the main llvm source directory. Than can happen if
# FCCT_PATH_TO_LLVM_SOURCE is not set. This helps finding lit.py
if(NOT LLVM_MAIN_SRC_DIR)
  set(LLVM_MAIN_SRC_DIR ${LLVM_BUILD_MAIN_SRC_DIR})
endif()
add_lit_testsuite(check-fcct "Running the FCCT regression tests"
  ${CMAKE_CURRENT_BINARY_DIR}
  #LIT ${LIT_COMMAND}
  PARAMS ${FCCT_TEST_PARAMS}
  DEPENDS ${FCCT_TEST_DEPS}
  ARGS ${FCCT_TEST_EXTRA_ARGS} --verbose --succinct
  )
set_target_properties(check-fcct PROPERTIES FOLDER "FCCT tests")

add_lit_testsuites(FCCT ${CMAKE_CURRENT_SOURCE_DIR}
  PARAMS ${FCCT_TEST_PARAMS}
  DEPENDS ${FCCT_TEST_DEPS}
)
