cmake_minimum_required (VERSION 3.10.2)
project (isotree VERSION 0.6.1)
set(CMAKE_BUILD_TYPE Release)

if ("${CMAKE_CXX_COMPILE_FEATURES}" MATCHES "cxx_std_17")
    set(CMAKE_CXX_STANDARD 17)
else()
    if ("${CMAKE_CXX_COMPILE_FEATURES}" MATCHES "cxx_std_14")
        set(CMAKE_CXX_STANDARD 14)
    else()
        set(CMAKE_CXX_STANDARD 11)
    endif()
endif()
set(CMAKE_CXX_STANDARD_REQUIRED False)

set(SRC_FILES ${PROJECT_SOURCE_DIR}/src/instantiate_template_headers.cpp
              ${PROJECT_SOURCE_DIR}/src/oop_interface.cpp
              ${PROJECT_SOURCE_DIR}/src/c_interface.cpp
              ${PROJECT_SOURCE_DIR}/src/indexer.cpp
              ${PROJECT_SOURCE_DIR}/src/merge_models.cpp
              ${PROJECT_SOURCE_DIR}/src/subset_models.cpp
              ${PROJECT_SOURCE_DIR}/src/serialize.cpp
              ${PROJECT_SOURCE_DIR}/src/sql.cpp
              ${PROJECT_SOURCE_DIR}/src/formatted_exporters.cpp)
set(BUILD_SHARED_LIBS True)
add_library(isotree SHARED ${SRC_FILES})
target_include_directories(isotree PRIVATE ${PROJECT_SOURCE_DIR}/src)
set(HEADER_FILES "${PROJECT_SOURCE_DIR}/include/isotree.hpp"
                 "${PROJECT_SOURCE_DIR}/include/isotree_oop.hpp"
                 "${PROJECT_SOURCE_DIR}/include/isotree_c.h")
set_target_properties(isotree PROPERTIES PUBLIC_HEADER "${HEADER_FILES}" SOVERSION 0 VERSION ${PROJECT_VERSION})
add_definitions(-DISOTREE_COMPILE_TIME)

## set to OFF to use the system's default RNG engine
option(USE_XOSHIRO "Use the xoshiro RNG family" ON)
if (USE_XOSHIRO)
    message(STATUS "Using the Xoshiro random engine family")
    add_definitions(-D_USE_XOSHIRO)
else()
    message(STATUS "Using the compiler's default random engine")
endif()

## set to ON to avoid compiling with 'long double' type
## (its usage is optional and selectable at runtime, but makes
##  the compiled binaries much heavier)
option(NO_LONG_DOUBLE "Avoid adding support for 'long double' type" OFF)
if (NO_LONG_DOUBLE)
    message(STATUS "Avoid adding support for 'long double' type.")
    add_definitions(-DNO_LONG_DOUBLE)
endif()

## set to ON to compile for only 'double' and 'int' types
## (otherwise, compiles also for 'float', 'int64_t', 'size_t')
option(NO_TEMPLATED_VERSIONS "Don't create multiple templated versions for different data types" OFF)
if (NO_TEMPLATED_VERSIONS)
    message(STATUS "Building with support for only 'double' and 'int' types.")
    add_definitions(-DNO_TEMPLATED_VERSIONS)
endif()

## set to OFF to export all symbols
include(CheckCXXSourceCompiles)
option(HIDE_INTERNAL_SYMBOLS "Set hidden visibility for non-exported symbols" ON)
if (HIDE_INTERNAL_SYMBOLS AND NOT WIN32)
    set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
    set(CMAKE_REQUIRED_FLAGS " -fvisibility=hidden")
    check_cxx_source_compiles(
        "
        [[gnu::visibility(\"default\")]]
        int myfun() {
            return 0;
        }

        int main(int argc, char **argv)
        {
            return myfun();
        }
        "
        SUPPORTS_FVISIBILITY_HIDDEN
    )
    set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
    if (SUPPORTS_FVISIBILITY_HIDDEN)
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fvisibility=hidden")
	add_definitions(-DEXPLICITLTY_EXPORT_SYMBOLS)
    else()
        message(STATUS "Hidden symbols visibility not supported by the compiler, will export all symbols.")
    endif()
else()
    message(STATUS "Making all symbols in the library visible.")
endif()

## https://cliutils.gitlab.io/modern-cmake/chapters/packages/OpenMP.html
find_package(OpenMP)
if (OpenMP_CXX_FOUND)
    target_link_libraries(isotree PUBLIC OpenMP::OpenMP_CXX)
else()
    message(STATUS "OpenMP not found - will compile without multi-threading support")
endif()

# check if the robin-map library is available under /src
set(CMAKE_REQUIRED_INCLUDES ${PROJECT_SOURCE_DIR}/src)
check_cxx_source_compiles(
    "
    #include \"robinmap/include/tsl/robin_growth_policy.h\"
    int main(int argc, char **argv)
    {
        return 0;
    }
    "
    HAS_ROBIN_MAP
)
if (HAS_ROBIN_MAP)
    add_definitions(-D_USE_ROBIN_MAP)
else()
    # if it's not under /src, check if there's a system install
    find_package(tsl-robin-map)
    if (tsl-robin-map_FOUND)
        target_link_libraries(isotree PRIVATE tsl::robin_map)
	add_definitions(-D_USE_SYSTEM_ROBIN)
    else()
        message(WARNING "Robin-Map Headers not found - will use STL hashmaps which are slower.")
    endif()
endif()

# For handling large files with MinGW
if (WIN32)
    if (CMAKE_SIZEOF_VOID_P GREATER_EQUAL 8 AND (MSYS OR MINGW OR GCC))
        add_definitions(-D_FILE_OFFSET_BITS=64)
    endif()
endif()

# Compiler optimizations
check_cxx_source_compiles(
    "
    #include <cstddef>
    int main(int argc, char **argv)
    {
        double *__restrict x = nullptr;
        return 0;
    }
    "
    SUPPORTS_RESTRICT
)
if (SUPPORTS_RESTRICT)
    add_definitions(-DSUPPORTS_RESTRICT=1)
endif()

if (MSVC)
    if (NOT (${CMAKE_CXX_FLAGS_RELEASE} MATCHES "/O2"))
        set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
        set(CMAKE_REQUIRED_FLAGS "/O2")
        check_cxx_source_compiles(
            "
            int main(int argc, char **argv)
            {
                return 0;
            }
            "
            SUPPORTS_O2
        )
        set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
        if (SUPPORTS_O2)
            set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /O2")
        endif()
    endif()
    if (NOT (${CMAKE_CXX_FLAGS_RELEASE} MATCHES "/fp:except-"))
        set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
        set(CMAKE_REQUIRED_FLAGS "/fp:except-")
        check_cxx_source_compiles(
            "
            int main(int argc, char **argv)
            {
                return 0;
            }
            "
            SUPPORTS_FPNOEX
        )
        set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
        if (SUPPORTS_FPNOEX)
            set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /fp:except-")
        endif()
    endif()
else()
    if (NOT (${CMAKE_CXX_FLAGS_RELEASE} MATCHES "-O3 "))
        set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
        set(CMAKE_REQUIRED_FLAGS "-O3")
        check_cxx_source_compiles(
            "
            int main(int argc, char **argv)
            {
                return 0;
            }
            "
            SUPPORTS_O3
        )
        set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
        if (SUPPORTS_O3)
            set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")
        else()
            set(CMAKE_REQUIRED_FLAGS "-O2")
            check_cxx_source_compiles(
                "
                int main(int argc, char **argv)
                {
                    return 0;
                }
                "
                SUPPORTS_O2
            )
            set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
            if (SUPPORTS_O2)
                set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O2")
            endif()
        endif()
    endif()
    if (NOT (${CMAKE_CXX_FLAGS_RELEASE} MATCHES "-fno-trapping-math"))
        set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
        set(CMAKE_REQUIRED_FLAGS "-fno-trapping-math")
        check_cxx_source_compiles(
            "
            int main(int argc, char **argv)
            {
                return 0;
            }
            "
            SUPPORTS_FNTM
        )
        set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
        if (SUPPORTS_FNTM)
            set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fno-trapping-math")
        endif()
    endif()
    if (NOT (${CMAKE_CXX_FLAGS_RELEASE} MATCHES "-fno-math-errno"))
        set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
        set(CMAKE_REQUIRED_FLAGS "-fno-math-errno")
        check_cxx_source_compiles(
            "
            int main(int argc, char **argv)
            {
                return 0;
            }
            "
            SUPPORTS_FNE
        )
        set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
        if (SUPPORTS_FNE)
            set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fno-math-errno")
        endif()
    endif()
endif()

option(USE_MARCH_NATIVE "Build with -march=native" OFF)
if (USE_MARCH_NATIVE AND NOT MSVC)
    set(OLD_FLAGS ${CMAKE_REQUIRED_FLAGS})
    set(CMAKE_REQUIRED_FLAGS "-march=native")
    check_cxx_source_compiles(
        "
        int main(int argc, char **argv)
        {
            return 0;
        }
        "
        SUPPORTS_MARCH_NATIVE
    )
    set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
    if (SUPPORTS_MARCH_NATIVE)
        message(STATUS "Adding flag -march=native.")
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native")
    else()
        set(CMAKE_REQUIRED_FLAGS "-mcpu=native")
        check_cxx_source_compiles(
            "
            int main(int argc, char **argv)
            {
                return 0;
            }
            "
            SUPPORTS_MCPU_NATIVE
        )
        set(CMAKE_REQUIRED_FLAGS ${OLD_FLAGS})
        if (SUPPORTS_MCPU_NATIVE)
            message(STATUS "Adding flag -mcpu=native.")
            set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -mcpu=native")
        else()
            message(WARNING "Flags -march=native and -mcpu=native not supported by the compiler.")
        endif()
    endif()
endif()

# Link-time optimization if supported
# https://stackoverflow.com/questions/31355692/how-do-i-enable-link-time-optimization-lto-with-cmake
include(CheckIPOSupported)
check_ipo_supported(RESULT LTO_SUPPORTED)
if (LTO_SUPPORTED)
    set_property(TARGET isotree PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()


include(GNUInstallDirs)

if(NOT CMAKE_INSTALL_LIBDIR)
    set(INSTALL_LIBDIR lib)
else()
    set(INSTALL_LIBDIR ${CMAKE_INSTALL_LIBDIR})
endif()

if(NOT CMAKE_INSTALL_INCLUDEDIR)
    set(INSTALL_INCLUDEDIR include)
else()
    set(INSTALL_INCLUDEDIR ${CMAKE_INSTALL_INCLUDEDIR})
endif()

install(
    TARGETS isotree
    ARCHIVE DESTINATION ${INSTALL_LIBDIR}
    LIBRARY DESTINATION ${INSTALL_LIBDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

configure_file(isotree.pc.in isotree.pc @ONLY)
install(FILES ${CMAKE_BINARY_DIR}/isotree.pc DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/pkgconfig)

# uninstall target
if (NOT TARGET uninstall)
    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
        IMMEDIATE @ONLY
    )
    add_custom_target(
        uninstall
        COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
    )
endif()
