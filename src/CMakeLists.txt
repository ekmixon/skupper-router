##
## Licensed to the Apache Software Foundation (ASF) under one
## or more contributor license agreements.  See the NOTICE file
## distributed with this work for additional information
## regarding copyright ownership.  The ASF licenses this file
## to you under the Apache License, Version 2.0 (the
## "License"); you may not use this file except in compliance
## with the License.  You may obtain a copy of the License at
##
##   http://www.apache.org/licenses/LICENSE-2.0
##
## Unless required by applicable law or agreed to in writing,
## software distributed under the License is distributed on an
## "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
## KIND, either express or implied.  See the License for the
## specific language governing permissions and limitations
## under the License.
##

# Generate code from the management schema.
set(GENERATED_SOURCES
  schema_enum.h
  schema_enum.c
  )

set(GENERATOR_DEPENDS
  ${CMAKE_CURRENT_SOURCE_DIR}/schema_c.py
  ${CMAKE_SOURCE_DIR}/python/skupper_router/management/skrouter.json)
file (GLOB GENERATOR_SRC ${CMAKE_SOURCE_DIR}/python/skupper_router_internal/management/*.py)
set(GENERATOR_DEPENDS ${GENERATOR_DEPENDS} ${GENERATOR_SRC})
file (GLOB GENERATOR_SRC ${CMAKE_SOURCE_DIR}/python/skupper_router/management/*.py)
set(GENERATOR_DEPENDS ${GENERATOR_DEPENDS} ${GENERATOR_SRC})

add_custom_command (
  OUTPUT ${GENERATED_SOURCES}
  COMMAND ${Python_EXECUTABLE} ${CMAKE_BINARY_DIR}/tests/run.py -s ${CMAKE_CURRENT_SOURCE_DIR}/schema_c.py
  DEPENDS ${GENERATOR_DEPENDS})

# Build the skupper-router library.
set(qpid_dispatch_SOURCES
  adaptors/reference_adaptor.c
  adaptors/adaptor_utils.c
  adaptors/http_common.c
  adaptors/http1/http1_codec.c
  adaptors/http1/http1_adaptor.c
  adaptors/http1/http1_client.c
  adaptors/http1/http1_server.c
  adaptors/http1/http1_request_info.c
  adaptors/tcp_adaptor.c
  adaptors/test_adaptor.c
  alloc.c
  alloc_pool.c
  aprintf.c
  amqp.c
  atomic.c
  bitmask.c
  buffer.c
  enum.c
  error.c
  compose.c
  ctools.c
  connection_manager.c
  container.c
  delivery_state.c
  discriminator.c
  dispatch.c
  entity.c
  entity_cache.c
  failoverlist.c
  hash.c
  iterator.c
  log.c
  message.c
  parse.c
  parse_tree.c
  policy.c
  policy_spec.c
  protocol_adaptor.c
  proton_utils.c
  posix/threading.c
  python_embedded.c
  router_agent.c
  router_config.c
  platform.c
  protocol_log.c
  router.c
  router_core.c
  router_core/address_watch.c
  router_core/agent.c
  router_core/agent_address.c
  router_core/agent_config_address.c
  router_core/agent_config_auto_link.c
  router_core/agent_connection.c
  router_core/agent_link.c
  router_core/agent_router.c
  router_core/connections.c
  router_core/core_attach_address_lookup.c
  router_core/core_events.c
  router_core/core_link_endpoint.c
  router_core/core_client_api.c
  router_core/delivery.c
  router_core/error.c
  router_core/forwarder.c
  router_core/route_control.c
  router_core/router_core.c
  router_core/router_core_thread.c
  router_core/route_tables.c
  router_core/management_agent.c
  router_core/terminus.c
  router_core/transfer.c
  router_core/core_timer.c
  router_core/module.c
  router_core/modules/edge_router/module.c
  router_core/modules/edge_router/addr_proxy.c
  router_core/modules/edge_router/connection_manager.c
  router_core/modules/test_hooks/core_test_hooks.c
  router_core/modules/edge_addr_tracking/edge_addr_tracking.c
  router_core/modules/address_lookup_client/address_lookup_client.c
  router_core/modules/heartbeat_edge/heartbeat_edge.c
  router_core/modules/heartbeat_server/heartbeat_server.c
  router_core/modules/stuck_delivery_detection/delivery_tracker.c
  router_core/modules/mobile_sync/mobile.c
  router_core/modules/streaming_link_scrubber/streaming_link_scrubber.c
  router_node.c
  router_pynode.c
  schema_enum.c
  static_assert.c
  server.c
  timer.c
  trace_mask.c
  python_utils.c
  qd_asan_interface.c
  )

set(qpid_dispatch_INCLUDES
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR}/router_core
  ${CMAKE_CURRENT_BINARY_DIR}
  )

set(qpid_dispatch_LIBRARIES
  ${Proton_Proactor_LIBRARIES}
  ${Proton_Core_LIBRARIES}
  ${Proton_Tls_LIBRARIES}
  ${CMAKE_THREAD_LIBS_INIT}
  ${rt_lib}
  ${CMAKE_DL_LIBS}
  ${Python_LIBRARIES}
  )

# USE_LIBWEBSOCKETS is true only if LIBWEBSOCKETS_FOUND
if(USE_LIBWEBSOCKETS)
  list(APPEND qpid_dispatch_SOURCES http-libwebsockets.c)
  list(APPEND qpid_dispatch_INCLUDES ${LIBWEBSOCKETS_INCLUDE_DIRS})
  list(APPEND qpid_dispatch_LIBRARIES ${LIBWEBSOCKETS_LIBRARIES})
else(USE_LIBWEBSOCKETS)
  list(APPEND qpid_dispatch_SOURCES http-none.c)
endif(USE_LIBWEBSOCKETS)

if(USE_LIBNGHTTP2)
  list(APPEND qpid_dispatch_SOURCES adaptors/http2/http2_adaptor.c)
  list(APPEND qpid_dispatch_INCLUDES ${NGHTTP2_INCLUDE_DIRS})
  list(APPEND qpid_dispatch_LIBRARIES ${NGHTTP2_LIBRARIES})
else(USE_LIBNGHTTP2)
  list(APPEND qpid_dispatch_SOURCES adaptors/http2/http2_adaptor_none.c)
endif(USE_LIBNGHTTP2)

if ((DEFINED ASAN_LIBRARY) OR (DEFINED UBSAN_LIBRARY) OR (DEFINED TSAN_LIBRARY))
  set(USING_SANITIZERS TRUE)
endif()

add_library(skupper-router OBJECT ${qpid_dispatch_SOURCES})
target_include_directories(skupper-router PRIVATE ${qpid_dispatch_INCLUDES})
target_link_libraries(skupper-router PRIVATE ${qpid_dispatch_LIBRARIES})

# check for various function availability
check_symbol_exists(getrlimit sys/resource.h QD_HAVE_GETRLIMIT)

# https://stackoverflow.com/questions/54771452/expanding-a-variable-cmakedefine-and-generator-expression-in-template-file
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/config.h.in" CONFIG_H_IN)
string(CONFIGURE "${CONFIG_H_IN}" CONFIG_H_TMP)
file(GENERATE OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/config.h" CONTENT "${CONFIG_H_TMP}")
