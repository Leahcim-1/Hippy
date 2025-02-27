#
# Tencent is pleased to support the open source community by making
# Hippy available.
#
# Copyright (C) 2022 THL A29 Limited, a Tencent company.
# All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

cmake_minimum_required(VERSION 3.14)

project("core")

get_filename_component(PROJECT_ROOT_DIR "${PROJECT_SOURCE_DIR}/.." REALPATH)

include("${PROJECT_ROOT_DIR}/buildconfig/cmake/GlobalPackagesModule.cmake")
include("${PROJECT_ROOT_DIR}/buildconfig/cmake/compiler_toolchain.cmake")

set(CMAKE_VERBOSE_MAKEFILE on)
set(CMAKE_CXX_STANDARD 14)

# region library
add_library(${PROJECT_NAME} STATIC)
target_include_directories(${PROJECT_NAME} PUBLIC include)
target_compile_options(${PROJECT_NAME} PRIVATE ${COMPILE_OPTIONS})
target_link_libraries(${PROJECT_NAME} PRIVATE android log)
# endregion

# region vm
if ("${JS_ENGINE}" STREQUAL "V8")
  GlobalPackages_Add(v8)
  target_link_libraries(${PROJECT_NAME} PRIVATE v8)

  # The following definitions are used by PUBLIC header files
  target_compile_definitions(${PROJECT_NAME} PUBLIC "JS_V8")
  get_target_property(V8_WITHOUT_INSPECTOR v8 INTERFACE_V8_WITHOUT_INSPECTOR)
  if (V8_WITHOUT_INSPECTOR)
    target_compile_definitions(${PROJECT_NAME} PUBLIC "V8_WITHOUT_INSPECTOR")
  endif ()
elseif ("${JS_ENGINE}" STREQUAL "JSC")
  target_compile_definitions(${PROJECT_NAME} PUBLIC "JS_JSC")
endif ()
# endregion

# region base
add_subdirectory(third_party/base)
target_include_directories(${PROJECT_NAME} PUBLIC third_party/base/include)
target_link_libraries(${PROJECT_NAME} PUBLIC tdf_base)
# endregion

# region source set
set(SOURCE_SET
    src/base/file.cc
    src/base/js_value_wrapper.cc
    src/base/task.cc
    src/base/task_runner.cc
    src/base/thread.cc
    src/base/thread_id.cc
    src/engine.cc
    src/modules/console_module.cc
    src/modules/contextify_module.cc
    src/modules/module_register.cc
    src/modules/timer_module.cc
    src/napi/callback_info.cc
    src/napi/js_native_turbo.cc
    src/scope.cc
    src/task/common_task.cc
    src/task/javascript_task.cc
    src/task/javascript_task_runner.cc
    src/task/worker_task_runner.cc)
if ("${JS_ENGINE}" STREQUAL "V8")
  list(APPEND SOURCE_SET
      src/napi/v8/js_native_api_v8.cc
      src/napi/v8/js_native_turbo_v8.cc
      src/napi/v8/native_source_code_android.cc)
  if (NOT V8_WITHOUT_INSPECTOR)
    list(APPEND SOURCE_SET
            src/inspector/v8_channel_impl.cc
            src/inspector/v8_inspector_client_impl.cc)
  endif ()
elseif ("${JS_ENGINE}" STREQUAL "JSC")
  list(APPEND SOURCE_SET
      src/napi/jsc/js_native_api_jsc.cc
      src/napi/jsc/js_native_api_value_jsc.cc
      src/napi/jsc/js_native_jsc_helper.cc
      src/napi/jsc/js_native_turbo_jsc.cc
      src/napi/jsc/native_source_code_ios.cc)
endif ()
set(SOURCE_SET_STANDALONE
    src/modules/console_module.cc
    src/modules/contextify_module.cc
    src/modules/timer_module.cc)
target_sources(${PROJECT_NAME} PRIVATE ${SOURCE_SET})
# The linker treats `.a` and `.o` files are different:
#   * `.o` files from `.a` file only referenced `.o` object files are linked.
#   * `.o` files in the linker always go inside, referenced or not.
#
# The `SOURCE_SET_STANDALONE` including static initializers,
# so its visibility must be PUBLIC.
target_sources(${PROJECT_NAME} PUBLIC ${SOURCE_SET_STANDALONE})
# endregion
