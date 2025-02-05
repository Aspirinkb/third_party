# Copyright (c) 2021, NVIDIA CORPORATION. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required(VERSION 3.17)
project(tritonthirdparty LANGUAGES C CXX)

#
# ExternalProject Dependencies
#
include(ExternalProject)

if(NOT TRITON_THIRD_PARTY_INSTALL_PREFIX)
  message(FATAL_ERROR "TRITON_THIRD_PARTY_INSTALL_PREFIX must be set")
endif() # TRITON_THIRD_PARTY_INSTALL_PREFIX

# If CMAKE_TOOLCHAIN_FILE is set, propagate that hint path to the external
# projects.
set(_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE "")
if (CMAKE_TOOLCHAIN_FILE)
  set(_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE "-DCMAKE_TOOLCHAIN_FILE:PATH=${CMAKE_TOOLCHAIN_FILE}")
endif()

# If VCPKG_TARGET_TRIPLET is set, propagate that hint path to the external
# projects.
set(_CMAKE_ARGS_VCPKG_TARGET_TRIPLET "")
if (VCPKG_TARGET_TRIPLET)
  set(_CMAKE_ARGS_VCPKG_TARGET_TRIPLET "-DVCPKG_TARGET_TRIPLET:STRING=${VCPKG_TARGET_TRIPLET}")
endif()

# If OPENSSL_ROOT_DIR is set, propagate that hint path to the external
# projects with OpenSSL dependency.
set(_CMAKE_ARGS_OPENSSL_ROOT_DIR "")
if (OPENSSL_ROOT_DIR)
  set(_CMAKE_ARGS_OPENSSL_ROOT_DIR "-DOPENSSL_ROOT_DIR:PATH=${OPENSSL_ROOT_DIR}")
endif()

set(ENV{HTTP_PROXY}  "http://127.0.0.1:7890")
set(ENV{HTTPS_PROXY}  "http://127.0.0.1:7890")
set(ENV{ALL_PROXY}  "socks5://127.0.0.1:7890")
#
# Build libcurl
#
ExternalProject_Add(curl
  PREFIX curl
  GIT_REPOSITORY "https://github.com/curl/curl.git"
  GIT_TAG "curl-7_66_0"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/curl/src/curl"
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DBUILD_TESTING:BOOL=OFF
    -DBUILD_CURL_EXE:BOOL=OFF
    -DBUILD_SHARED_LIBS:BOOL=OFF
    -DCURL_STATICLIB:BOOL=ON
    -DHTTP_ONLY:BOOL=ON
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/curl
)

#
# Get the protobuf and grpc source used for the GRPC endpoint. We must
# use v1.25.0 because later GRPC has significant performance
# regressions (e.g. resnet50 bs128).
#
ExternalProject_Add(grpc-repo
  PREFIX grpc-repo
  GIT_REPOSITORY "https://github.com/grpc/grpc.git"
  GIT_TAG "v1.25.0"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo/src/grpc"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  INSTALL_COMMAND ""
  TEST_COMMAND ""
)

#
# The GCS build requires a abseil newer than the one found in grpc
# v1.25.0 so grab a newer grpc just for abseil.
#
ExternalProject_Add(grpc-repo-new
  PREFIX grpc-repo-new
  GIT_REPOSITORY "https://github.com/grpc/grpc.git"
  GIT_TAG "v1.29.1"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo-new/src/grpc"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  INSTALL_COMMAND ""
  TEST_COMMAND ""
)

#
# Build nlohmann/json
#
ExternalProject_Add(nlohmann-json
  PREFIX nlohmann-json
  GIT_REPOSITORY "https://github.com/nlohmann/json.git"
  GIT_TAG "v3.9.0"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/json"
  CMAKE_CACHE_ARGS
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=TRUE
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/nlohmann_json
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DBUILD_SHARED_LIBS:STRING=yes
    -DBUILD_TESTING:BOOL=OFF
    -H. -Bcmake-out
)

#
# Build protobuf project from grpc-repo
#
# Use -DCMAKE_BUILD_TYPE:STRING=RELEASE to workaround the protobuf build issue
# described in https://stackoverflow.com/a/60218066/2091555
ExternalProject_Add(protobuf
  PREFIX protobuf
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo/src/grpc/third_party/protobuf/cmake"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -Dprotobuf_BUILD_TESTS:BOOL=OFF
    -Dprotobuf_WITH_ZLIB:BOOL=OFF
    -Dprotobuf_MSVC_STATIC_RUNTIME:BOOL=OFF
    -DCMAKE_BUILD_TYPE:STRING=RELEASE
    -DBUILD_SHARED_LIBS:STRING=no
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/protobuf
  DEPENDS grpc-repo
)

#
# Build absl (abseil) project from grpc-repo-new (see comment above).
#
ExternalProject_Add(absl
  PREFIX absl
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo-new/src/grpc/third_party/abseil-cpp"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=TRUE
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/absl
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DBUILD_TESTING:BOOL=OFF
  DEPENDS grpc-repo-new
)

#
# Build googletest project from grpc-repo
#
ExternalProject_Add(googletest
  PREFIX protobuf
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo/src/grpc/third_party/googletest"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/googletest
  DEPENDS grpc-repo
)

# Location where protobuf-config.cmake will be installed varies by
# platform
if (WIN32)
  set(_FINDPACKAGE_PROTOBUF_CONFIG_DIR "${TRITON_THIRD_PARTY_INSTALL_PREFIX}/protobuf/cmake")
else()
  set(_FINDPACKAGE_PROTOBUF_CONFIG_DIR "${TRITON_THIRD_PARTY_INSTALL_PREFIX}/protobuf/lib/cmake/protobuf")
endif()

#
# Build c-ares project from grpc-repo
#
ExternalProject_Add(c-ares
  PREFIX c-ares
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo/src/grpc/third_party/cares/cares"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCARES_SHARED:BOOL=OFF
    -DCARES_STATIC:BOOL=ON
    -DCARES_STATIC_PIC:BOOL=ON
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/c-ares
  DEPENDS grpc-repo
)

#
# Build GRPC
#
ExternalProject_Add(grpc
  PREFIX grpc
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/grpc-repo/src/grpc"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    -DgRPC_INSTALL:BOOL=ON
    -DgRPC_BUILD_TESTS:BOOL=OFF
    -DgRPC_PROTOBUF_PROVIDER:STRING=package
    -DgRPC_PROTOBUF_PACKAGE_TYPE:STRING=CONFIG
    -DgRPC_ZLIB_PROVIDER:STRING=package
    -DgRPC_CARES_PROVIDER:STRING=package
    -DgRPC_SSL_PROVIDER:STRING=package
    -Dc-ares_DIR:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/c-ares/lib/cmake/c-ares
    -DProtobuf_DIR:PATH=${_FINDPACKAGE_PROTOBUF_CONFIG_DIR}
    ${_CMAKE_ARGS_OPENSSL_ROOT_DIR}
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/grpc
  DEPENDS grpc-repo c-ares protobuf
)

#
# Build libevent
#
ExternalProject_Add(libevent
  PREFIX libevent
  GIT_REPOSITORY "https://github.com/libevent/libevent.git"
  GIT_TAG "release-2.1.8-stable"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/libevent/src/libevent"
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/libevent
)

#
# Build patched libevhtp
#
if (WIN32)
# FIXME evhtp can't be built successfully in native Windows environment
# CMAKE_PREFIX_PATH is needed to find libevent
ExternalProject_Add(libevhtp
  PREFIX libevhtp
  SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/libevhtp/libevhtp"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DEVHTP_DISABLE_REGEX:BOOL=ON
    -DEVHTP_DISABLE_SSL:BOOL=ON
    -DEVHTP_TRITON_ENABLE_HTTP_CONTIGUOUS:BOOL=ON
    -DEVHTP_TRITON_ENABLE_TRACING:BOOL=${TRITON_ENABLE_TRACING}
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DCMAKE_PREFIX_PATH:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/libevent
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/libevhtp
  DEPENDS libevent
)
else()
ExternalProject_Add(libevhtp
  PREFIX libevhtp
  SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/libevhtp/libevhtp"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DEVHTP_DISABLE_REGEX:BOOL=ON
    -DEVHTP_DISABLE_SSL:BOOL=ON
    -DEVHTP_TRITON_ENABLE_HTTP_CONTIGUOUS:BOOL=ON
    -DEVHTP_TRITON_ENABLE_TRACING:BOOL=${TRITON_ENABLE_TRACING}
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DLibevent_DIR:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/libevent/lib/cmake/libevent
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/libevhtp
  DEPENDS libevent
)
endif()

#
# Build Prometheus C++ library
#
ExternalProject_Add(prometheus-cpp
  PREFIX prometheus-cpp
  URL "https://github.com/jupp0r/prometheus-cpp/archive/v0.7.0.tar.gz"
  URL_HASH SHA256=93907d937fa7eab9605cba786123d3eba4e87e3dca8ecec93ff9eae4eef8de5a
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/prometheus-cpp/src/prometheus-cpp"
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DENABLE_PUSH:BOOL=OFF
    -DENABLE_PULL:BOOL=OFF
    -DENABLE_TESTING:BOOL=OFF
    -DUSE_THIRDPARTY_LIBRARIES:BOOL=OFF
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/prometheus-cpp
)

#
# Build crc32c
#
ExternalProject_Add(crc32c
  PREFIX crc32c
  GIT_REPOSITORY "https://github.com/google/crc32c.git"
  GIT_TAG "1.1.1"
  SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/crc32c/src/crc32c"
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCRC32C_BUILD_TESTS:BOOL=OFF
    -DBUILD_SHARED_LIBS:STRING=no
    -DCRC32C_BUILD_BENCHMARKS:BOOL=OFF
    -DCRC32C_USE_GLOG:BOOL=OFF
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/crc32c
)

# This is necessary because google-cloud-cpp
# does not pass protobuf_DIR and gRPC_DIR etc to its
# external projects but expects them in CMAKE_PREFIX_PATH.
#
# absl must preceed grpc to ensure that the absl from the newer grpc
# repo is used and not the absl from the older grpc.
set(GCS_CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH}
   ${TRITON_THIRD_PARTY_INSTALL_PREFIX}/c-ares/lib/cmake/c-ares
   ${TRITON_THIRD_PARTY_INSTALL_PREFIX}/absl/lib/cmake/absl
   ${TRITON_THIRD_PARTY_INSTALL_PREFIX}/crc32c/lib/cmake/Crc32c
   ${TRITON_THIRD_PARTY_INSTALL_PREFIX}/grpc/lib/cmake/grpc
   ${TRITON_THIRD_PARTY_INSTALL_PREFIX}/nlohmann_json/lib/cmake/nlohmann_json
   ${_FINDPACKAGE_PROTOBUF_CONFIG_DIR})

#
# Build google-cloud-cpp
#
# ExternalProject_Add(google-cloud-cpp
#   PREFIX google-cloud-cpp
#   GIT_REPOSITORY "https://github.com/googleapis/google-cloud-cpp.git"
#   GIT_TAG "v1.22.0"
#   SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/google-cloud-cpp/src/google-cloud-cpp"
#   CMAKE_CACHE_ARGS
#     ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
#     ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
#     -DGOOGLE_CLOUD_CPP_DEPENDENCY_PROVIDER:STRING=package
#     -DBUILD_TESTING:BOOL=OFF
#     -DCMAKE_PREFIX_PATH:PATH=${GCS_CMAKE_PREFIX_PATH}
#     -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
#     -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/google-cloud-cpp
#     -Dnlohmann_json_DIR:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/nlohmann_json/lib/cmake/nlohmann_json
#     -DProtobuf_DIR:PATH=${_FINDPACKAGE_PROTOBUF_CONFIG_DIR}
#     -DCrc32c_DIR:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/crc32c/lib/cmake/Crc32c
#   DEPENDS grpc c-ares protobuf crc32c absl nlohmann-json
# )

#
# Build azure-storage-cpplite
#
# ExternalProject_Add(azure-storage-cpplite
#   PREFIX azure-storage-cpplite
#   GIT_REPOSITORY "https://github.com/Azure/azure-storage-cpplite.git"
#   GIT_TAG "v0.3.0"
#   SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/azure-storage-cpplite/src/azure-storage-cpplite"
#   CMAKE_CACHE_ARGS
#     ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
#     ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
#     -DBUILD_SHARED_LIBS:BOOL=OFF
#     -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
#     -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
#     -DCURL_INCLUDE_DIR:STRING=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/curl/include
#     -DCURL_LIBRARY:STRING=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/curl/lib
#     -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/azure-storage-cpplite
#   DEPENDS curl
# )

#
# Build CNMeM (CUDA memory management library)
#
ExternalProject_Add(cnmem
  PREFIX cnmem
  SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/cnmem/cnmem"
  DOWNLOAD_COMMAND ""
  CMAKE_CACHE_ARGS
    ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
    ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
    -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
    -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/cnmem
)

#
# Build aws-sdk-cpp
#
# ExternalProject_Add(aws-sdk-cpp
#   PREFIX aws-sdk-cpp
#   GIT_REPOSITORY "https://github.com/aws/aws-sdk-cpp.git"
#   GIT_TAG "1.7.129"
#   SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/aws-sdk-cpp/src/aws-sdk-cpp"
#   CMAKE_CACHE_ARGS
#     ${_CMAKE_ARGS_CMAKE_TOOLCHAIN_FILE}
#     ${_CMAKE_ARGS_VCPKG_TARGET_TRIPLET}
#     -DBUILD_ONLY:STRING=s3
#     -DBUILD_SHARED_LIBS:BOOL=OFF
#     -DMINIMIZE_SIZE:BOOL=ON
#     -DENABLE_TESTING:BOOL=OFF
#     -DCMAKE_POSITION_INDEPENDENT_CODE:BOOL=ON
#     -DCMAKE_INSTALL_PREFIX:PATH=${TRITON_THIRD_PARTY_INSTALL_PREFIX}/aws-sdk-cpp
# )
