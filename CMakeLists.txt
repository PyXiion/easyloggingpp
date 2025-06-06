cmake_minimum_required(VERSION 3.28)

project(Easyloggingpp CXX)

macro(require_cpp14)
        if (${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} GREATER 3.0)
                # CMake 3.1 has built-in CXX standard checks.
                message("-- Setting C++14")
                set(CMAKE_CXX_STANDARD 14)
                set(CMAKE_CXX_STANDARD_REQUIRED on)
        else()
                if (CMAKE_CXX_COMPILER_ID MATCHES "GCC")
                    message ("-- GNU CXX (-std=c++14)")
                    list(APPEND CMAKE_CXX_FLAGS "-std=c++14")
                elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
                    message ("-- CLang CXX (-std=c++14)")
                    list(APPEND CMAKE_CXX_FLAGS "-std=c++14")
                else()
                    message ("-- Easylogging++ requires C++14. Your compiler does not support it.")
                endif()
        endif()
endmacro()

option(test "Build all tests" OFF)
option(build_static_lib "Build easyloggingpp as a static library" OFF)
option(lib_utc_datetime "Build library with UTC date/time logging" OFF)

set(ELPP_MAJOR_VERSION "9")
set(ELPP_MINOR_VERSION "96")
set(ELPP_PATCH_VERSION "7")
set(ELPP_VERSION_STRING "${ELPP_MAJOR_VERSION}.${ELPP_MINOR_VERSION}.${ELPP_PATCH_VERSION}")

set(ELPP_INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "The directory the headers are installed in")
set(ELPP_PKGCONFIG_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

install(FILES
    src/easylogging++.h
    src/easylogging++.cc
    DESTINATION "${ELPP_INCLUDE_INSTALL_DIR}"
    COMPONENT dev)

configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/cmake/easyloggingpp.pc.cmakein
    ${CMAKE_CURRENT_BINARY_DIR}/easyloggingpp.pc @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/easyloggingpp.pc DESTINATION "${ELPP_PKGCONFIG_INSTALL_DIR}")

include(CheckIncludeFileCXX)
check_include_file_cxx("execinfo.h" HAVE_EXECINFO)
if (HAVE_EXECINFO)
	add_definitions(-DHAVE_EXECINFO)
endif()

if (build_static_lib)
        if (lib_utc_datetime)
                add_definitions(-DELPP_UTC_DATETIME)
        endif()

        require_cpp14()
        add_library(easyloggingpp STATIC src/easylogging++.cc)
        target_include_directories(easyloggingpp PUBLIC src)
        set_property(TARGET easyloggingpp PROPERTY POSITION_INDEPENDENT_CODE ON)

        install(TARGETS
            easyloggingpp
            ARCHIVE DESTINATION lib)
endif()

export(PACKAGE ${PROJECT_NAME})


########################################## Unit Testing ###################################
if (test)
    # We need C++14 (Google Test requirement)
    require_cpp14()
    set (CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

    find_package (GTest REQUIRED)
    include_directories (${GTEST_INCLUDE_DIRS})

    enable_testing()

    if (EMSCRIPTEN)
        set(CMAKE_EXE_LINKER_FLAGS "-O2 -s TOTAL_MEMORY=134217728")
    endif()

    add_executable(easyloggingpp-unit-tests
        src/easylogging++.cc
        test/main.cc
    )

    target_compile_definitions(easyloggingpp-unit-tests PUBLIC
        ELPP_FEATURE_ALL
        ELPP_LOGGING_FLAGS_FROM_ARG
        ELPP_NO_DEFAULT_LOG_FILE
        ELPP_FRESH_LOG_FILE
        ELPP_STOP_ON_FIRST_ASSERTION
        ELPP_STL_LOGGING
        ELPP_FORCE_ENV_VAR_FROM_BASH
        ELPP_ENABLE_THREADING
        ELPP_FEATURE_CRASH_LOG
        ELPP_SYSLOG
    )

    # Standard linking to gtest stuff.
    target_link_libraries(easyloggingpp-unit-tests ${GTEST_BOTH_LIBRARIES})

    add_test(NAME easyloggingppUnitTests COMMAND easyloggingpp-unit-tests -v)
endif()
