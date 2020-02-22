
include (CTest)

include_directories(..)

set(XXLIBS  ws2_32.lib)

set(VCEFLAGS "${VCEFLAGS} -D__PtW32NoCatchWarn")

macro(add_testcase test cleanupstyle def buildtype)

    if(${buildtype} STREQUAL "shared")
        set(lib_test test-dll-${test}${cleanupstyle}${PTW32_VER})
        set(lib_lib pthread${cleanupstyle}${PTW32_VER})
    elseif(${buildtype} STREQUAL "static")
        set(lib_test test-${test}${cleanupstyle}${PTW32_VER})
        set(lib_lib libpthread${cleanupstyle}${PTW32_VER})
    endif()

    set(c_dep "")
    if(${test} MATCHES "benchtest")
        set(c_dep "benchlib.c")
    endif()

    if(${test} MATCHES "openmp1")
        if(MSVC)
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /openmp")
        endif()
    endif()

    add_executable(${lib_test} ${test}.c ${c_dep})
    target_link_libraries(${lib_test} ${lib_lib} ${XXLIBS})
    target_compile_definitions(${lib_test} PUBLIC -D_CONSOLE -D_MBCS "-D${def}")
    add_dependencies(${lib_test} ${lib_lib})

    if(${CMAKE_GENERATOR} MATCHES "Visual Studio")
        install(FILES ${CMAKE_BINARY_DIR}/${CMAKE_BUILD_TYPE}/tests/${lib_test}.exe DESTINATION ${TESTDEST})
    else()
        install(FILES ${CMAKE_BINARY_DIR}/tests/${lib_test}.exe DESTINATION ${TESTDEST})
    endif()

    if(${cleanupstyle} MATCHES "VCE")
        set_target_properties(${lib_test} PROPERTIES COMPILE_FLAGS ${VCEFLAGS})
    endif()

    add_test(NAME ${lib_test} COMMAND ${lib_test})

endmacro()


file(GLOB TESTS *.c)

foreach(t ${TESTS})

    get_filename_component(test ${t} NAME)
    string(REPLACE ".c" "" test "${test}")

    # exclusions
    if(${test} STREQUAL "benchlib")
        list(REMOVE_ITEM TESTS ${t})
        continue()
    elseif(${test} STREQUAL "context2")    # SEGFAULT
        continue()
    elseif(${test} STREQUAL "tryentercs2") # SEGFAULT
        continue()
    endif()

    if(BUILD_SHARED_LIBS)
        if(${CLEANUP_STYLE} STREQUAL "VCE")
            shared_lib ( VCE __PTW32_CLEANUP_CXX shared )
        elseif(${CLEANUP_STYLE} STREQUAL "VSE")
            shared_lib ( VSE __PTW32_CLEANUP_SEH shared )
        elseif(${CLEANUP_STYLE} STREQUAL "VC")
            shared_lib ( VC  __PTW32_CLEANUP_C shared )
        endif()
    else()
        if(${CLEANUP_STYLE} STREQUAL "VCE")
            add_testcase(${test} VCE __PTW32_CLEANUP_CXX static )
        elseif(${CLEANUP_STYLE} STREQUAL "VSE")
            add_testcase(${test} VSE __PTW32_CLEANUP_SEH static )
        elseif(${CLEANUP_STYLE} STREQUAL "VC")
            add_testcase(${test} VC  __PTW32_CLEANUP_C static )
        endif()
    endif()

endforeach(t)
