# Test tree — populated in Phase 1+.
# A placeholder smoke test keeps `ctest` green during the bootstrap phase.

find_package(GTest CONFIG REQUIRED)
include(GoogleTest)

add_executable(smoke_test smoke_test.cpp)
target_link_libraries(smoke_test PRIVATE GTest::gtest GTest::gtest_main)
gtest_discover_tests(smoke_test)
