# Created by Daniel Pfeifer <daniel@pfeifer-mail.de>
#
# Distributed under the Boost Software License, Version 1.0.
# See accompanying file LICENSE_1_0.txt or copy at
#   http://www.boost.org/LICENSE_1_0.txt

cmake_minimum_required(VERSION 2.8.7 FATAL_ERROR)
project(BoostSVN2GIT)

set(BOOST_SVN_REPOSITORY "/home/svnsync/svn/boost" CACHE PATH
  "path to (the local mirror of) Boost's Subversion repository"
  )

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

# collect all rulesets
file(GLOB rulesets RELATIVE "${CMAKE_BINARY_DIR}"
  "${CMAKE_SOURCE_DIR}/rulesets/libs/*"
  "${CMAKE_SOURCE_DIR}/rulesets/tools/*"
  "${CMAKE_SOURCE_DIR}/rulesets/website"
  )
list(SORT rulesets)

# make sure that sandbox is at the end of the list!
# we want to make sure that the rulesets of libraries with history in the
# sandbox take higher precedence.
set(sandbox "${CMAKE_SOURCE_DIR}/rulesets/sandbox")
file(RELATIVE_PATH sandbox "${CMAKE_BINARY_DIR}" "${sandbox}")
list(APPEND rulesets "${sandbox}")

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
  "  branch master\n"
  "end match\n\n"
  "match /\n"
  "end match\n\n"
  )
list(APPEND repositories "svn2git-fallback")

# clean
file(REMOVE_RECURSE "${CMAKE_BINARY_DIR}/conversion")
file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/conversion")

# perform conversion
add_custom_target(conversion
  COMMAND 
    "${CMAKE_BINARY_DIR}/svn2git/svn-all-fast-export"
    --identity-map "${CMAKE_SOURCE_DIR}/authors.txt"
    --rules "${ruleset_file}"
    --resume-from 56354             # TODO: find reason for segfault !!
    "${BOOST_SVN_REPOSITORY}"
  DEPENDS
    "${CMAKE_BINARY_DIR}/svn2git/svn-all-fast-export"
  WORKING_DIRECTORY
    "${CMAKE_BINARY_DIR}/conversion"
  )

# push all ${repositories} to bitbucket
foreach(repo IN LISTS repositories)
  add_custom_target(push_${repo} ALL
    COMMAND git push --mirror git@bitbucket.org:boostorg/${repo}.git
    DEPENDS conversion
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/conversion/${repo}"
    )
endforeach()

