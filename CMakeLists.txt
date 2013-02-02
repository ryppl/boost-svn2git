# Created by Daniel Pfeifer <daniel@pfeifer-mail.de>
#
# Distributed under the Boost Software License, Version 1.0.
# See accompanying file LICENSE_1_0.txt or copy at
#   http://www.boost.org/LICENSE_1_0.txt

cmake_minimum_required(VERSION 2.8.7 FATAL_ERROR)
project(BoostSVN2GIT)

set(ruleset_file "${CMAKE_BINARY_DIR}/svn2git-ruleset")
file(REMOVE "${ruleset_file}")

# build svn2git
find_package(Qt4 REQUIRED QtCore)
file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/svn2git")
add_custom_command(OUTPUT "${CMAKE_BINARY_DIR}/svn2git/Makefile"
  COMMAND ${QT_QMAKE_EXECUTABLE} "${CMAKE_SOURCE_DIR}/svn2git"
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/svn2git"
  )
add_custom_command(OUTPUT "${CMAKE_BINARY_DIR}/svn2git/svn-all-fast-export"
  COMMAND make
  DEPENDS "${CMAKE_BINARY_DIR}/svn2git/Makefile"
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/svn2git"
  )
add_custom_target(svn2git
  DEPENDS "${CMAKE_BINARY_DIR}/svn2git/svn-all-fast-export"
  )

# collect all rulesets
file(GLOB rulesets
  "${CMAKE_SOURCE_DIR}/rulesets/libs/*"
  "${CMAKE_SOURCE_DIR}/rulesets/tools/*"
  )

set(repositories)
foreach(ruleset IN LISTS rulesets)
  get_filename_component(name "${ruleset}" NAME)
  file(APPEND "${ruleset_file}"
    "create repository ${name}\n"
    "end repository\n\n"
    "include ${ruleset}\n\n"
    )
  list(APPEND repositories "${name}")
endforeach()

# put all unmatched files to svn2git-fallback
file(APPEND "${ruleset_file}"
  "create repository svn2git-fallback\n"
  "end repository\n\n"
  "match /\n"
  "  repository svn2git-fallback\n"
  "end match\n\n"
  )
list(APPEND repositories "svn2git-fallback")

# TODO: perform conversion

# TODO: push all ${repositories} to bitbucket and github

