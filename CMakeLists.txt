set(PACKAGE protobuf-c)
set(PACKAGE_NAME protobuf-c)
set(PACKAGE_VERSION 1.3.0)

cmake_minimum_required(VERSION 3.1 FATAL_ERROR)

project(protobuf-c VERSION ${PACKAGE_VERSION})

include(CPack)
include(CTest)

option(BUILD_PROTO3 "BUILD_PROTO3" ON)

include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)

set(PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")

set(defs
  PACKAGE_VERSION="${PACKAGE_VERSION}"
  PACKAGE_STRING="${PACKAGE_STRING}"
  WORDS_BIGENDIAN=${WORDS_BIGENDIAN})

if(MSVC)
  # using Visual Studio C++
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4267 /wd4244")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267 /wd4244")
endif()

add_library(protobuf-c
  protobuf-c/protobuf-c.c
  protobuf-c/protobuf-c.h)
target_compile_definitions(protobuf-c PUBLIC ${defs})
target_include_directories(protobuf-c PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/protobuf-c>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)

find_package(Protobuf REQUIRED)

if(MSVC AND NOT BUILD_SHARED_LIBS)
	# In case we are building static libraries, link also the runtime library statically
	# so that MSVCR*.DLL is not required at runtime.
	# https://msdn.microsoft.com/en-us/library/2kzt1wy3.aspx
	# This is achieved by replacing msvc option /MD with /MT and /MDd with /MTd
	# http://www.cmake.org/Wiki/CMake_FAQ#How_can_I_build_my_MSVC_application_with_a_static_runtime.3F
	foreach(flag_var
		CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
		CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
	  if(${flag_var} MATCHES "/MD")
		string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
	  endif(${flag_var} MATCHES "/MD")
	endforeach(flag_var)
endif()
set(PROTOC_GEN_C_SRC
    protoc-c/c_bytes_field.cc
    protoc-c/c_bytes_field.h
    protoc-c/c_enum.cc
    protoc-c/c_enum_field.cc
    protoc-c/c_enum_field.h
    protoc-c/c_enum.h
    protoc-c/c_extension.cc
    protoc-c/c_extension.h
    protoc-c/c_field.cc
    protoc-c/c_field.h
    protoc-c/c_file.cc
    protoc-c/c_file.h
    protoc-c/c_generator.cc
    protoc-c/c_generator.h
    protoc-c/c_helpers.cc
    protoc-c/c_helpers.h
    protoc-c/c_message.cc
    protoc-c/c_message_field.cc
    protoc-c/c_message_field.h
    protoc-c/c_message.h
    protoc-c/c_primitive_field.cc
    protoc-c/c_primitive_field.h
    protoc-c/c_service.cc
    protoc-c/c_service.h
    protoc-c/c_string_field.cc
    protoc-c/c_string_field.h
    protoc-c/main.cc)
add_executable(protoc-gen-c ${PROTOC_GEN_C_SRC})
target_include_directories(protoc-gen-c PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/protoc-c>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/protobuf-c>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_compile_definitions(protoc-gen-c PUBLIC ${defs})
target_link_libraries(protoc-gen-c PUBLIC
  ${PROTOBUF_PROTOC_LIBRARY}
  ${PROTOBUF_LIBRARY})

if(BUILD_PROTO3)
  #target_compile_definitions(protobuf-c PUBLIC HAVE_PROTO3)
  target_compile_definitions(protoc-gen-c PUBLIC HAVE_PROTO3)
endif()

if(CMAKE_HOST_UNIX)
  add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
                     COMMAND ln -sf protoc-gen-c protoc-c
                     DEPENDS protoc-gen-c)
endif()

function(generate_test_sources PROTO_FILE SRC HDR)
	add_custom_command(
    OUTPUT ${SRC} ${HDR}
    COMMAND ${PROTOBUF_PROTOC_EXECUTABLE}
    ARGS --plugin=$<TARGET_FILE:protoc-gen-c>
         -I${CMAKE_CURRENT_SOURCE_DIR}
         --c_out=${CMAKE_CURRENT_BINARY_DIR}
         ${PROTO_FILE}
    DEPENDS protoc-gen-c)
endfunction()


if(BUILD_TESTING)
  generate_test_sources(t/test.proto t/test.pb-c.c t/test.pb-c.h)

  add_executable(test-generated-code
    t/generated-code/test-generated-code.c
    t/test.pb-c.c t/test.pb-c.h )
  target_link_libraries(test-generated-code PUBLIC protobuf-c)

  add_custom_command(
    OUTPUT t/test-full.pb.cc t/test-full.pb.h
    COMMAND ${PROTOBUF_PROTOC_EXECUTABLE}
    ARGS --cpp_out=${CMAKE_CURRENT_BINARY_DIR}
         -I${CMAKE_CURRENT_SOURCE_DIR}
         t/test-full.proto)

  generate_test_sources(
    t/test-full.proto t/test-full.pb-c.c t/test-full.pb-c.h)

  add_executable(cxx-generate-packed-data
    t/generated-code2/cxx-generate-packed-data.cc
    t/test-full.pb.h t/test-full.pb.cc)
  target_include_directories(cxx-generate-packed-data PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
  target_link_libraries(cxx-generate-packed-data ${PROTOBUF_LIBRARY})

  file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/t/generated-code2)
  add_custom_command(OUTPUT t/generated-code2/test-full-cxx-output.inc
    COMMAND ${CMAKE_CURRENT_BINARY_DIR}/cxx-generate-packed-data
            >t/generated-code2/test-full-cxx-output.inc
    DEPENDS cxx-generate-packed-data)

  generate_test_sources(
    t/test-optimized.proto
    t/test-optimized.pb-c.c
    t/test-optimized.pb-c.h)

  add_executable(test-generated-code2
    t/generated-code2/test-generated-code2.c
    t/generated-code2/test-full-cxx-output.inc
    t/test-full.pb-c.h
    t/test-full.pb-c.c
    t/test-optimized.pb-c.h
    t/test-optimized.pb-c.c)
  target_link_libraries(test-generated-code2 PUBLIC protobuf-c)

  generate_test_sources(
    t/issue220/issue220.proto
    t/issue220/issue220.pb-c.c
    t/issue220/issue220.pb-c.h)
  add_executable(
    test-issue220
    t/issue220/issue220.c
    t/issue220/issue220.pb-c.c
    t/issue220/issue220.pb-c.h)
  target_link_libraries(test-issue220 PUBLIC protobuf-c)

  generate_test_sources(
    t/issue251/issue251.proto
    t/issue251/issue251.pb-c.c
    t/issue251/issue251.pb-c.h)
  add_executable(
    test-issue251
    t/issue251/issue251.c
    t/issue251/issue251.pb-c.c
    t/issue251/issue251.pb-c.h)
  target_link_libraries(test-issue251 protobuf-c)

  add_executable(test-version t/version/version.c)
  target_link_libraries(test-version PUBLIC protobuf-c)

  generate_test_sources(
    t/test-proto3.proto
    t/test-proto3.pb-c.c
    t/test-proto3.pb-c.h)
  add_executable(
    test-generated-code3
    t/generated-code/test-generated-code.c
    t/test-proto3.pb-c.c
    t/test-proto3.pb-c.h)
  target_compile_definitions(test-generated-code3 PUBLIC PROTO3)
  target_link_libraries(test-generated-code3 PUBLIC protobuf-c)

  add_test(test-generated-code test-generated-code)
  add_test(test-generated-code2 test-generated-code2)
  add_test(test-generated-code3 test-generated-code3)
  add_test(test-issue220 test-issue220)
  add_test(test-issue251 test-issue251)
  add_test(test-version test-version)
endif()

install(TARGETS protoc-gen-c protobuf-c RUNTIME DESTINATION bin LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
install(FILES protobuf-c/protobuf-c.h DESTINATION include/protobuf-c)
install(FILES protobuf-c/protobuf-c.h DESTINATION include)

if(CMAKE_HOST_UNIX)
  install(CODE "execute_process(
                  COMMAND ln -sf protoc-gen-c protoc-c
                  WORKING_DIRECTORY ${CMAKE_INSTALL_PREFIX}/bin
                )")
endif()
