project(jom)
cmake_minimum_required(VERSION 2.8.11)

set(CMAKE_AUTOMOC ON)

option(JOM_ENABLE_TESTS "Enable unit-testing for jom" OFF)

if(JOM_ENABLE_TESTS)
    enable_testing()
endif(JOM_ENABLE_TESTS)


# where to look first for cmake modules
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules;${CMAKE_MODULE_PATH}")

find_package(Qt5Core)

include_directories(
  ${CMAKE_BINARY_DIR}
  src/jomlib
)
if(MSVC)
    add_definitions(
      -D_CRT_SECURE_NO_DEPRECATE
    )
   # Qt disables the native wchar_t type, do it too to avoid linking issues
   set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Zc:wchar_t-" )
endif(MSVC)
add_definitions(
  -DQT_NO_CAST_TO_ASCII
  -DQT_USE_FAST_CONCATENATION
  -DQT_USE_FAST_OPERATOR_PLUS
  -DUNICODE
)

# in subdirectory src:
#
set(JOM_MOCS
    src/jomlib/targetexecutor.h
    src/jomlib/jomprocess.h
    src/jomlib/commandexecutor.h
    src/jomlib/jobclient.h
    src/jomlib/jobclientacquirehelper.h
)

set(JOM_SRCS
    src/jomlib/commandexecutor.cpp
    src/jomlib/dependencygraph.cpp
    src/jomlib/exception.cpp
    src/jomlib/fastfileinfo.cpp
    src/jomlib/filetime.cpp
    src/jomlib/helperfunctions.cpp
    src/jomlib/jobclient.cpp
    src/jomlib/jobclientacquirehelper.cpp
    src/jomlib/jobserver.cpp
    src/jomlib/macrotable.cpp
    src/jomlib/makefile.cpp
    src/jomlib/makefilefactory.cpp
    src/jomlib/makefilelinereader.cpp
    src/jomlib/options.cpp
    src/jomlib/parser.cpp
    src/jomlib/ppexpr_grammar.cpp
    src/jomlib/ppexprparser.cpp
    src/jomlib/preprocessor.cpp
    src/jomlib/targetexecutor.cpp
    src/jomlib/dependencygraph.h
    src/jomlib/exception.h
    src/jomlib/fastfileinfo.h
    src/jomlib/filetime.h
    src/jomlib/helperfunctions.h
    src/jomlib/macrotable.h
    src/jomlib/makefile.h
    src/jomlib/makefilefactory.h
    src/jomlib/makefilelinereader.h
    src/jomlib/options.h
    src/jomlib/parser.h
    src/jomlib/ppexpr_grammar_p.h
    src/jomlib/ppexprparser.h
    src/jomlib/preprocessor.h
    src/jomlib/stable.h
)

 # run moc on all headers and add the moc files to the SRCS
 list(APPEND JOM_SRCS ${JOM_GENERATED_SRCS} ${JOM_MOCS})

if(WIN32)
    list(APPEND JOM_SRCS
        src/jomlib/iocompletionport.h
        src/jomlib/iocompletionport.cpp
        src/jomlib/jomprocess.cpp
    )
else()
    add_definitions(
      -DUSE_QPROCESS
    )
    list(APPEND JOM_SRCS
        src/jomlib/jomprocess_qt.cpp
    )
endif()

 set(JOM_APP_MOCS
    src/app/application.h
)
 set(JOM_APP_SRCS
    src/app/main.cpp
    src/app/application.cpp
)
 list(APPEND JOM_APP_SRCS ${JOM_APP_GENERATED_SRCS} ${JOM_APP_MOCS})
 source_group("Generated Sources" FILES ${JOM_GENERATED_SRCS} ${JOM_APP_GENERATED_SRCS})

 add_executable(jom ${JOM_APP_SRCS} ${JOM_SRCS})
 if(WIN32)
    target_link_libraries(jom Qt5::Core ws2_32)
 else(WIN32)
    target_link_libraries(jom Qt5::Core)
 endif(WIN32)

 # install binaries to bin/, libraries to lib/ and import libraries to lib/ too
 install(TARGETS jom  RUNTIME DESTINATION bin
                      LIBRARY DESTINATION lib
                      ARCHIVE DESTINATION lib)

 # if we want to use tests:
 if(JOM_ENABLE_TESTS)
     # in subdirectory tests:
     set(TESTS_SRCS tests/tests.cpp tests/tests.h)
     add_executable(jom-test ${TESTS_SRCS} ${JOM_SRCS})
     target_link_libraries(jom-test Qt5::Core Qt5::Test)

     # copy the data directory 'makefiles' over into the build directory as the tests should be run from there
     file(GLOB_RECURSE JOM_TEST_DATA RELATIVE ${CMAKE_SOURCE_DIR}/tests/makefiles/ "tests/makefiles/*")
     file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/makefiles)
     foreach(_file ${JOM_TEST_DATA})
         configure_file(${CMAKE_SOURCE_DIR}/tests/makefiles/${_file} ${CMAKE_BINARY_DIR}/makefiles/${_file} COPY_ONLY)
     endforeach(_file ${JOM_TEST_DATA})

     # add one unit test per function:
     # you can run the unittests all at once using 'make test' from the build directory
     # to produce the list: jom-test --functions | sed 's|[(][)]||'
     set(TEST_NAMES
        includeFiles
        includeCycle
        macros
        invalidMacros
        preprocessorExpressions
        preprocessorDivideByZero
        preprocessorInvalidExpressions
        conditionals
        dotDirectives
        descriptionBlocks
        inferenceRules
        cycleInTargets
        dependentsWithSpace
        multipleTargets
        comments
        fileNameMacros
        fileNameMacrosInDependents
        windowsPathsInTargetName
        caseInsensitiveDependents
        environmentVariables
        ignoreExitCodes
        inlineFiles
        unicodeFiles
        builtin_cd
        suffixes
        nonexistentDependent
        outOfDateCheck
     )
     foreach(TEST_NAME ${TEST_NAMES})
        add_test(${TEST_NAME} jom-test ${TEST_NAME})
     endforeach()

 endif(JOM_ENABLE_TESTS)

