cmake_minimum_required(VERSION 3.20)
project(noisefloor LANGUAGES CXX VERSION 0.1.0)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif()

option(NOISEFLOOR_ENABLE_SANITIZERS "Build with AddressSanitizer + UndefinedBehaviorSanitizer" OFF)
option(NOISEFLOOR_BUILD_TESTS "Build the Catch2 test suite" ON)

# MSVC doesn't understand -Wall/-Wextra (fails with D8021), it uses /W4
if(MSVC)
  set(NOISEFLOOR_WARNINGS /W4)
else()
  set(NOISEFLOOR_WARNINGS -Wall -Wextra -Wpedantic)
endif()

add_library(noisefloor_core STATIC
  src/text.cpp
  src/bank.cpp
  src/scoring.cpp
  src/daily.cpp
  src/session.cpp
  src/problems.cpp
)
target_include_directories(noisefloor_core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_compile_options(noisefloor_core PRIVATE ${NOISEFLOOR_WARNINGS})
# The share line and some aliases are UTF-8 string literals. MSVC otherwise
# reads source files in the local code page and can mangle them.
if(MSVC)
  target_compile_options(noisefloor_core PUBLIC /utf-8)
endif()

if(NOISEFLOOR_ENABLE_SANITIZERS)
  target_compile_options(noisefloor_core PUBLIC -fsanitize=address,undefined -fno-omit-frame-pointer -g)
  target_link_options(noisefloor_core PUBLIC -fsanitize=address,undefined)
endif()

add_executable(noisefloor cli/main.cpp)
target_link_libraries(noisefloor PRIVATE noisefloor_core)
target_compile_options(noisefloor PRIVATE ${NOISEFLOOR_WARNINGS})
target_compile_definitions(noisefloor PRIVATE NOISEFLOOR_BANKS_DIR="${CMAKE_CURRENT_SOURCE_DIR}/banks")

if(NOISEFLOOR_BUILD_TESTS)
  enable_testing()
  include(FetchContent)
  FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v3.7.1
  )
  FetchContent_MakeAvailable(Catch2)

  add_executable(noisefloor_tests
    tests/test_text.cpp
    tests/test_bank.cpp
    tests/test_scoring.cpp
    tests/test_daily.cpp
    tests/test_session.cpp
    tests/test_banks_content.cpp
    tests/test_problems.cpp
  )
  target_link_libraries(noisefloor_tests PRIVATE noisefloor_core Catch2::Catch2WithMain)
  target_compile_options(noisefloor_tests PRIVATE ${NOISEFLOOR_WARNINGS})
  target_compile_definitions(noisefloor_tests PRIVATE NOISEFLOOR_BANKS_DIR="${CMAKE_CURRENT_SOURCE_DIR}/banks")

  include(${catch2_SOURCE_DIR}/extras/Catch.cmake)
  catch_discover_tests(noisefloor_tests)
endif()
