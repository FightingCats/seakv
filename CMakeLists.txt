cmake_minimum_required(VERSION 3.11)

project(
  seakv
  VERSION 1.0.0
  DESCRIPTION "kv db"
  LANGUAGES CXX C)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/")

set(CMAKE_CXX_STANDARD "17")

option(WITH_DPDK "Enable DPDK messaging" OFF)
if(WITH_DPDK)
  find_package(dpdk)
  if(NOT DPDK_FOUND)
    include(BuildDPDK)
    build_dpdk(${CMAKE_BINARY_DIR}/dpdk)
  endif()
  set(HAVE_DPDK TRUE)
endif()

CMAKE_DEPENDENT_OPTION(WITH_SPDK "Enable SPDK" OFF
  "CMAKE_SYSTEM_PROCESSOR MATCHES i386|i686|amd64|x86_64|AMD64|aarch64" OFF)
if(WITH_SPDK)
  if(NOT WITH_BLUESTORE)
    message(SEND_ERROR "Please enable WITH_BLUESTORE for using SPDK")
  endif()
  include(BuildSPDK)
  build_spdk()
  set(HAVE_SPDK TRUE)
endif(WITH_SPDK)



option(WITH_SEASTAR "Build seastar components" OFF)
if(WITH_SEASTAR)
  find_package(c-ares 1.13.0 QUIET)
  if(NOT c-ares_FOUND)
    message(STATUS "Could not find c-ares, will build it")
    include(Buildc-ares)
    build_c_ares()
  endif()
  macro(find_package name)
    if(NOT TARGET ${name})
      _find_package(${ARGV})
    endif()
  endmacro ()
  set(Seastar_API_LEVEL "6" CACHE STRING "" FORCE)
  set(Seastar_HWLOC OFF CACHE BOOL "" FORCE)
  set(Seastar_STD_OPTIONAL_VARIANT_STRINGVIEW ON CACHE BOOL "" FORCE)
  if(Seastar_DPDK)
    find_package(dpdk QUIET)
    if(NOT DPDK_FOUND)
      include(BuildDPDK)
      build_dpdk(${CMAKE_BINARY_DIR}/src/dpdk)
    endif()
  endif()
  list(APPEND Seastar_CXX_FLAGS
    "-DSEASTAR_NO_EXCEPTION_HACK"
    "-Wno-error"
    "-Wno-sign-compare"
    "-Wno-attributes"
    "-Wno-pessimizing-move"
    "-Wno-address-of-packed-member"
    "-Wno-non-virtual-dtor")
  set(Seastar_CXX_FLAGS "${Seastar_CXX_FLAGS}" CACHE STRING "" FORCE)
  add_subdirectory(seastar)
  # create the directory so cmake won't complain when looking at the imported
  # target: Seastar exports this directory created at build-time
  file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/seastar/gen/include")
endif()

option(WITH_RDMA "Enable RDMA in async messenger" OFF)
if(WITH_RDMA)
  find_package(verbs REQUIRED)
  set(HAVE_VERBS ${VERBS_FOUND})
  find_package(rdmacm REQUIRED)
  set(HAVE_RDMACM ${RDMACM_FOUND})
  set(HAVE_RDMA TRUE)
endif()

# Rocksdb
option(WITH_SYSTEM_ROCKSDB "require and build with system rocksdb" OFF)
if (WITH_SYSTEM_ROCKSDB)
  find_package(RocksDB 5.14 REQUIRED)
endif()

add_executable(seakv main.cc)


