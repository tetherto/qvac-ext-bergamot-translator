set(COMPILE_TESTS OFF)

if(COMPILE_WASM)
  # This is a bad way of adding compilation flags. Will be improved soon.
  add_compile_options(${WASM_COMPILE_FLAGS})
  add_link_options(${WASM_LINK_FLAGS})
endif(COMPILE_WASM)

add_subdirectory(ssplit-cpp EXCLUDE_FROM_ALL)

get_property(INCLUDE_DIRECTORIES DIRECTORY ssplit-cpp/src PROPERTY INCLUDE_DIRECTORIES)
target_include_directories(ssplit PUBLIC ${INCLUDE_DIRECTORIES})

set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} PARENT_SCOPE)    
set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} PARENT_SCOPE)    

if(COMPILE_PYTHON)
  add_subdirectory(pybind11)
endif(COMPILE_PYTHON)
