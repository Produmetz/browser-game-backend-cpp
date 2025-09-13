cmake_minimum_required(VERSION 3.11)

project(game_server CXX)
set(CMAKE_CXX_STANDARD 20)

include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
conan_basic_setup(TARGETS)

set(THREADS_PREFER_PTHREAD_FLAG ON)

find_package(Boost 1.78.0 REQUIRED)
if(Boost_FOUND)
  include_directories(${Boost_INCLUDE_DIRS})
endif()

find_package(Threads REQUIRED)

add_library(common_lib STATIC
    src/tagged.h
    src/tagged_uuid.h
    src/tagged_uuid.cpp
    src/sdk.h
)
target_include_directories(common_lib PUBLIC 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# 2. Model core library
add_library(model_lib STATIC
    src/model.h
    src/model.cpp
    src/model_core.h
    src/model_core.cpp
    src/loot_generator.h
    src/loot_generator.cpp
    src/collision_detector.h
    src/collision_detector.cpp
)
target_link_libraries(model_lib PUBLIC
    common_lib
    CONAN_PKG::libpq
    CONAN_PKG::libpqxx
)
target_include_directories(model_lib PUBLIC 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# 3. Serialization library
add_library(serialization_lib STATIC
    src/extra_data.h
    src/extra_data.cpp
    src/boost_json.cpp
    src/json_loader.h
    src/json_loader.cpp
    src/model_serialization.h
)
target_link_libraries(serialization_lib PUBLIC
    model_lib
    common_lib
    CONAN_PKG::boost
)
target_include_directories(serialization_lib PUBLIC 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# 4. Database layer
add_library(db_lib STATIC
    src/save_manager.h
    src/DB_manager.h
)
target_link_libraries(db_lib PUBLIC
    model_lib
    serialization_lib
)

# 5. HTTP server library
add_library(http_lib STATIC
    src/http_server.h
    src/http_server.cpp
    src/request_handler.h
    src/request_handler.cpp
)
target_link_libraries(http_lib PUBLIC
    serialization_lib
    db_lib
    CONAN_PKG::boost
    Threads::Threads
)
target_include_directories(http_lib PUBLIC 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Main executable
add_executable(game_server src/main.cpp)
target_link_libraries(game_server
    http_lib
    db_lib
    serialization_lib
    model_lib
    common_lib
)