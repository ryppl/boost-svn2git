# Created by Daniel Pfeifer <daniel@pfeifer-mail.de>
#
# Distributed under the Boost Software License, Version 1.0.
# See accompanying file LICENSE_1_0.txt or copy at
#   http://www.boost.org/LICENSE_1_0.txt

cmake_minimum_required(VERSION 2.8.7 FATAL_ERROR)
project(BoostSVN2GIT)

add_subdirectory(svn2git)

set(BOOST_SVN_REPOSITORY "/home/svnsync/svn/boost" CACHE PATH
  "path to (the local mirror of) Boost's Subversion repository"
  )

set(BOOST_GIT_REPOSITORY "/mnt/ramdisk/conversion" CACHE PATH
  "where to generate the local Git repositories"
  )

# clean
file(REMOVE_RECURSE "${BOOST_GIT_REPOSITORY}")
file(MAKE_DIRECTORY "${BOOST_GIT_REPOSITORY}")

set(ruleset_file "${CMAKE_BINARY_DIR}/svn2git-ruleset")
file(REMOVE "${ruleset_file}")

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

# perform conversion
add_custom_target(conversion
  COMMAND
    $<TARGET_FILE:svn2git>
    --identity-map "${CMAKE_SOURCE_DIR}/authors.txt"
    --rules "${ruleset_file}"
    --resume-from 6
    "${BOOST_SVN_REPOSITORY}"
  DEPENDS
    svn2git
  WORKING_DIRECTORY
    "${BOOST_GIT_REPOSITORY}"
  )

# push all ${repositories} to bitbucket
foreach(repo IN LISTS repositories)
  add_custom_target(push_${repo} ALL
    COMMAND git push --mirror git@bitbucket.org:boostorg/${repo}.git
    DEPENDS conversion
    WORKING_DIRECTORY "${BOOST_GIT_REPOSITORY}/${repo}"
    )
endforeach()

