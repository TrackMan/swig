# CMakeLists.txt for C# test-suite
# Modern .NET Core/dotnet implementation

cmake_minimum_required(VERSION 3.12)

# Find required tools
find_program(DOTNET_EXECUTABLE NAMES dotnet)

if(NOT DOTNET_EXECUTABLE)
    message(WARNING "dotnet not found - C# tests will not be available")
    return()
else()
    message(STATUS "dotnet found: ${DOTNET_EXECUTABLE}")

    # Check dotnet version
    execute_process(
        COMMAND ${DOTNET_EXECUTABLE} --version
        OUTPUT_VARIABLE DOTNET_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    message(STATUS "dotnet version: ${DOTNET_VERSION}")
endif()

# C++ test cases
set(CPP_TEST_CASES
    complextest
    csharp_argument_defaults_feature
    csharp_attributes
    csharp_director_typemaps
    csharp_exceptions
    csharp_features
    csharp_lib_arrays
    csharp_lib_arrays_bool
    csharp_namespace_system_collision
    csharp_prepost
    csharp_swig2_compatibility
    csharp_typemaps
    director_wstring
    enum_thorough_simple
    enum_thorough_typesafe
    exception_partial_info
    intermediary_classname
    nested_scope
    li_boost_intrusive_ptr
    li_std_list
)

# C++11 test cases
set(CPP11_TEST_CASES
    cpp11_shared_ptr_const
    cpp11_shared_ptr_crtp_upcast
    cpp11_shared_ptr_nullptr_in_containers
    cpp11_shared_ptr_overload
    cpp11_shared_ptr_template_upcast
    cpp11_shared_ptr_upcast
    cpp11_std_unordered_map
    cpp11_std_unordered_set
    cpp11_strongly_typed_enumerations_simple
)

# Set up paths
set(SWIG_EXECUTABLE ${CMAKE_BINARY_DIR}/swig)
set(SWIG_LIB_DIR ${CMAKE_SOURCE_DIR}/Lib)
set(SWIGWARN_FILE ${CMAKE_BINARY_DIR}/swigwarn.swg)
set(TEST_SUITE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/..)
set(LIBRARY_CSPROJ_TEMPLATE ${CMAKE_CURRENT_SOURCE_DIR}/library.csproj.in)
set(RUNME_CSPROJ_TEMPLATE ${CMAKE_CURRENT_SOURCE_DIR}/runme.csproj.in)

# Helper function to add a C# test case
function(add_csharp_test_case TEST_NAME)
    # Determine source files
    set(INTERFACE_FILE "${TEST_SUITE_DIR}/${TEST_NAME}.i")
    set(RUNME_FILE "${CMAKE_CURRENT_SOURCE_DIR}/${TEST_NAME}_runme.cs")

    # Create output directory
    set(TEST_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/${TEST_NAME}")
    file(MAKE_DIRECTORY "${TEST_OUTPUT_DIR}")

    # Generated files
    set(WRAP_CXX_FILE "${TEST_OUTPUT_DIR}/${TEST_NAME}_wrap.cxx")
    set(LIBRARY_CSPROJ_FILE "${TEST_OUTPUT_DIR}/${TEST_NAME}.csproj")
    set(RUNME_CSPROJ_FILE "${TEST_OUTPUT_DIR}/runme/${TEST_NAME}_runme.csproj")

    # Base SWIG options
    set(SWIG_OPTS -csharp -c++ -namespace ${TEST_NAME}Namespace -Werror)

    # Test-specific options
    set(ALLOW_UNSAFE "false")
    set(NEEDS_NUMERICS "false")

    if("${TEST_NAME}" STREQUAL "csharp_swig2_compatibility")
        list(APPEND SWIG_OPTS -DSWIG2_CSHARP)
    endif()
    if("${TEST_NAME}" STREQUAL "intermediary_classname")
        list(APPEND SWIG_OPTS -dllimport intermediary_classname)
    endif()
    if("${TEST_NAME}" STREQUAL "complextest")
        set(NEEDS_NUMERICS "true")
    endif()
    if("${TEST_NAME}" STREQUAL "csharp_lib_arrays" OR "${TEST_NAME}" STREQUAL "csharp_lib_arrays_bool")
        set(ALLOW_UNSAFE "true")
    endif()

    # Run SWIG to generate wrapper
    add_custom_command(
        OUTPUT ${WRAP_CXX_FILE}
        COMMAND ${CMAKE_COMMAND} -E env "SWIG_LIB=${SWIG_LIB_DIR}"
            ${SWIG_EXECUTABLE}
            ${SWIG_OPTS}
            -I${CMAKE_BINARY_DIR}
            -I${TEST_SUITE_DIR}
            -o ${WRAP_CXX_FILE}
            -outdir ${TEST_OUTPUT_DIR}
            ${INTERFACE_FILE}
        DEPENDS swig ${SWIGWARN_FILE} ${INTERFACE_FILE}
        COMMENT "Running SWIG for ${TEST_NAME}"
        VERBATIM
    )

    # Create shared library
    add_library(${TEST_NAME}_module MODULE ${WRAP_CXX_FILE})
    set_target_properties(${TEST_NAME}_module PROPERTIES
        OUTPUT_NAME ${TEST_NAME}
        PREFIX ""
        CXX_STANDARD 11
    )
    target_include_directories(${TEST_NAME}_module PRIVATE ${TEST_SUITE_DIR})

    # On macOS, use .dylib extension for .NET compatibility
    if(APPLE)
        set_target_properties(${TEST_NAME}_module PROPERTIES
            SUFFIX ".dylib"
        )
    endif()

    # Copy the library to the test output directory after building
    add_custom_command(TARGET ${TEST_NAME}_module POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy
            $<TARGET_FILE:${TEST_NAME}_module>
            ${TEST_OUTPUT_DIR}/$<TARGET_FILE_NAME:${TEST_NAME}_module>
        COMMENT "Copying ${TEST_NAME} library to test directory"
    )

    # Configure the library .csproj file
    configure_file(
        ${LIBRARY_CSPROJ_TEMPLATE}
        ${LIBRARY_CSPROJ_FILE}
        @ONLY
    )

    # Build target for the C# wrapper library
    add_custom_target(${TEST_NAME}_build
        COMMAND ${CMAKE_COMMAND} -E echo "Building C# library for ${TEST_NAME}..."
        COMMAND ${DOTNET_EXECUTABLE} build ${LIBRARY_CSPROJ_FILE} -c Release
        DEPENDS ${TEST_NAME}_module
        WORKING_DIRECTORY ${TEST_OUTPUT_DIR}
        COMMENT "Building C# library: ${TEST_NAME}"
        USES_TERMINAL
        VERBATIM
    )

    # If there's a runme file, create a separate runme project
    if(EXISTS ${RUNME_FILE})
        # Create runme subdirectory
        file(MAKE_DIRECTORY "${TEST_OUTPUT_DIR}/runme")

        # Configure the runme .csproj file
        configure_file(
            ${RUNME_CSPROJ_TEMPLATE}
            ${RUNME_CSPROJ_FILE}
            @ONLY
        )

        # Copy runme file to runme directory
        add_custom_command(
            OUTPUT ${TEST_OUTPUT_DIR}/runme/${TEST_NAME}_runme.cs
            COMMAND ${CMAKE_COMMAND} -E copy ${RUNME_FILE} ${TEST_OUTPUT_DIR}/runme/
            DEPENDS ${RUNME_FILE}
            COMMENT "Copying runme file for ${TEST_NAME}"
        )

        # Build and run target
        add_custom_target(${TEST_NAME}
            COMMAND ${CMAKE_COMMAND} -E echo "Building runme for ${TEST_NAME}..."
            COMMAND ${DOTNET_EXECUTABLE} build ${RUNME_CSPROJ_FILE} -c Release
            COMMAND ${CMAKE_COMMAND} -E echo "Running ${TEST_NAME}..."
            COMMAND ${DOTNET_EXECUTABLE} run --project ${RUNME_CSPROJ_FILE} --no-build -c Release
            DEPENDS ${TEST_NAME}_build ${TEST_OUTPUT_DIR}/runme/${TEST_NAME}_runme.cs
            WORKING_DIRECTORY ${TEST_OUTPUT_DIR}/runme
            COMMENT "Running C# test: ${TEST_NAME}"
            USES_TERMINAL
            VERBATIM
        )

        # Separate run-only target (doesn't rebuild)
        add_custom_target(${TEST_NAME}_run
            COMMAND ${CMAKE_COMMAND} -E echo "Running ${TEST_NAME}..."
            COMMAND ${DOTNET_EXECUTABLE} run --project ${RUNME_CSPROJ_FILE} --no-build -c Release
            DEPENDS ${TEST_NAME}
            WORKING_DIRECTORY ${TEST_OUTPUT_DIR}/runme
            COMMENT "Running C# test: ${TEST_NAME} (no rebuild)"
            USES_TERMINAL
            VERBATIM
        )

        # Add CTest test
        add_test(
            NAME csharp_${TEST_NAME}
            COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target ${TEST_NAME}
        )
    else()
        # No runme file - the build target is the main target
        add_custom_target(${TEST_NAME}
            DEPENDS ${TEST_NAME}_build
            COMMENT "Building C# test: ${TEST_NAME}"
        )
    endif()
endfunction()

# Create a target for all tests
add_custom_target(check-csharp-test-suite
    COMMENT "Running all C# test suite cases"
)

# Add all test cases
foreach(TEST_CASE ${CPP_TEST_CASES} ${CPP11_TEST_CASES})
    add_csharp_test_case(${TEST_CASE})
    add_dependencies(check-csharp-test-suite ${TEST_CASE})
endforeach()

