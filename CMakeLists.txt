CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
PROJECT(libgraphqlparser C CXX)

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")

FIND_PACKAGE(PythonInterp)
IF (NOT PYTHON_VERSION_MAJOR EQUAL 2)
  MESSAGE(FATAL_ERROR "Python 2 is required.")
ENDIF()

FIND_PROGRAM(CTYPESGEN_FOUND ctypesgen.py)

FIND_PACKAGE(BISON 3)
FIND_PACKAGE(FLEX)
IF (BISON_FOUND)
  BISON_TARGET(graphqlparser parser.ypp ${CMAKE_CURRENT_SOURCE_DIR}/parser.tab.cpp)
ENDIF()

IF(FLEX_FOUND)
  FLEX_TARGET(GraphQLScanner lexer.lpp ${CMAKE_CURRENT_SOURCE_DIR}/lexer.cpp COMPILE_FLAGS "--header-file=lexer.h")
  IF (BISON_FOUND)
    ADD_FLEX_BISON_DEPENDENCY(GraphQLScanner graphqlparser)
  ENDIF()
ENDIF()

FILE(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/c)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

ADD_LIBRARY(graphqlparser SHARED
  JsonVisitor.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.h
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/AstVisitor.h
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAst.h
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAst.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAstForEachConcreteType.h
  c/GraphQLAstNode.cpp
  c/GraphQLAstToJSON.cpp
  c/GraphQLAstVisitor.h
  c/GraphQLAstVisitor.cpp
  c/GraphQLParser.cpp
  parser.tab.hpp
  parser.tab.cpp
  lexer.cpp
  lexer.h
  GraphQLParser.cpp)

# Enable this and remove CMAKE_CXX_FLAGS fiddle above when we are able
# to upgrade to CMake 2.8.12. Blocker seems to be Travis CI being on
# Ubuntu Precise; Trusty has 2.8.12.
# TARGET_COMPILE_OPTIONS(graphqlparser PUBLIC -std=gnu++11)

ADD_EXECUTABLE(dump_json_ast dump_json_ast.cpp)
TARGET_LINK_LIBRARIES(dump_json_ast graphqlparser)

FUNCTION(GENERATE_AST_FILE FILE_TYPE FILE_RELATIVE_PATH)
  ADD_CUSTOM_COMMAND(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${FILE_RELATIVE_PATH}
    COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.py ${FILE_TYPE} ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.ast > ${CMAKE_CURRENT_BINARY_DIR}/${FILE_RELATIVE_PATH}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.ast ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.py)
ENDFUNCTION(GENERATE_AST_FILE)

GENERATE_AST_FILE(cxx Ast.h)

GENERATE_AST_FILE(cxx_visitor AstVisitor.h)

GENERATE_AST_FILE(cxx_impl Ast.cpp)

GENERATE_AST_FILE(c c/GraphQLAst.h)

GENERATE_AST_FILE(c_impl c/GraphQLAst.cpp)

GENERATE_AST_FILE(c_visitor_impl c/GraphQLAstForEachConcreteType.h)

ADD_SUBDIRECTORY(python)

OPTION(test "Build tests." OFF)

INSTALL(DIRECTORY c ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR}/c DESTINATION include/graphqlparser
  FILES_MATCHING PATTERN "*.h"
  PATTERN "build" EXCLUDE)

INSTALL(FILES
  AstNode.h
  GraphQLParser.h
  JsonVisitor.h
  lexer.h
  location.hh
  parser.tab.hpp
  position.hh
  stack.hh
  syntaxdefs.h
  DESTINATION include/graphqlparser)
INSTALL(TARGETS graphqlparser
  LIBRARY DESTINATION lib)

IF (test)
  ADD_SUBDIRECTORY(test)
ENDIF()
