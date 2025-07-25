#
# Amalgamation
#
set(SINGLEHEADER_FILES
  ${CMAKE_CURRENT_BINARY_DIR}/ada.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/ada.h
  ${CMAKE_CURRENT_BINARY_DIR}/ada_c.h
  ${CMAKE_CURRENT_BINARY_DIR}/demo.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/demo.c
  ${CMAKE_CURRENT_BINARY_DIR}/README.md
)
set_source_files_properties(${SINGLEHEADER_FILES} PROPERTIES GENERATED TRUE)

# In theory, this is unneeded, because the tests module does the same test:
find_package (Python3 COMPONENTS Interpreter)

if (Python3_Interpreter_FOUND)
  MESSAGE( STATUS "Python found, we are going to amalgamate.py." )

  add_custom_command(
    OUTPUT ${SINGLEHEADER_FILES}
    COMMAND ${CMAKE_COMMAND} -E env
      AMALGAMATE_SOURCE_PATH=${PROJECT_SOURCE_DIR}/src
      AMALGAMATE_INPUT_PATH=${PROJECT_SOURCE_DIR}/include
      AMALGAMATE_OUTPUT_PATH=${CMAKE_CURRENT_BINARY_DIR}
      ${Python3_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/amalgamate.py
      #
      # This is the best way I could find to make amalgamation trigger whenever source files or
      # header files change: since the "ada" library has to get rebuilt when that happens, we
      # take a dependency on the generated library file (even though we're not using it). Depending
      # on ada-source doesn't do the trick because DEPENDS here can only depend on an
      # *artifact*--it won't scan source and include files the way a concrete library or executable
      # will.
      #
      # It sucks that we have to build the actual library to make it happen, but it's better than\
      # nothing!
      #
      DEPENDS amalgamate.py ada
  )
  add_custom_target(ada-singleheader-files DEPENDS ${SINGLEHEADER_FILES})

  #
  # Include this if you intend to #include "ada.cpp" in your own .cpp files.
  #
  add_library(ada-singleheader-include-source INTERFACE)
  target_include_directories(ada-singleheader-include-source INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
  if (ADA_USE_SIMDUTF)
    target_include_directories(ada-singleheader-include-source INTERFACE ${simdutf_SOURCE_DIR}/include)
  endif()
  add_dependencies(ada-singleheader-include-source ada-singleheader-files)

  add_library(ada-singleheader-source INTERFACE)
  target_sources(ada-singleheader-source INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/ada.cpp>)
  target_link_libraries(ada-singleheader-source INTERFACE ada-singleheader-include-source)
  if (ADA_USE_SIMDUTF)
    target_link_libraries(ada-singleheader-source INTERFACE simdutf)
  endif()

  if (ADA_TESTING OR ADA_BUILD_SINGLE_HEADER_LIB)
    add_library(ada-singleheader-lib STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/ada.cpp>)
    if (ADA_USE_SIMDUTF)
      target_link_libraries(ada-singleheader-lib simdutf)
      target_include_directories(ada-singleheader-lib PRIVATE ${simdutf_SOURCE_DIR}/include)
    endif()
  endif(ADA_TESTING OR ADA_BUILD_SINGLE_HEADER_LIB)

  if (ADA_TESTING)
    add_executable(demo $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/demo.cpp>)
    target_link_libraries(demo ada-singleheader-include-source)
    if (ADA_USE_SIMDUTF)
      target_link_libraries(demo simdutf)
    endif()

    add_executable(demo_no_url_pattern $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/demo.cpp>)
    target_link_libraries(demo_no_url_pattern ada-singleheader-include-source)
    if (ADA_USE_SIMDUTF)
      target_link_libraries(demo_no_url_pattern simdutf)
    endif()
    target_compile_definitions(demo_no_url_pattern PRIVATE ADA_INCLUDE_URL_PATTERN=0)

    if (CMAKE_CROSSCOMPILING_EMULATOR)
      add_test(demo_no_url_pattern ${CMAKE_CROSSCOMPILING_EMULATOR} demo_no_url_pattern)
    else()
      add_test(demo_no_url_pattern demo_no_url_pattern)
    endif()

    add_executable(cdemo $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/demo.c>)
    target_link_libraries(cdemo ada-singleheader-lib)
    if (CMAKE_CROSSCOMPILING_EMULATOR)
      add_test(cdemo ${CMAKE_CROSSCOMPILING_EMULATOR} cdemo)
    else()
      add_test(cdemo cdemo)
    endif()
  endif()
else()
  MESSAGE( STATUS "Python not found, we are unable to test amalgamate.py." )
endif()
