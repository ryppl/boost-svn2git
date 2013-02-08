# Created by Daniel Pfeifer <daniel@pfeifer-mail.de>
#
# Distributed under the Boost Software License, Version 1.0.
# See accompanying file LICENSE_1_0.txt or copy at
#   http://www.boost.org/LICENSE_1_0.txt

cmake_minimum_required(VERSION 2.8.7 FATAL_ERROR)
project(BoostSVN2GIT)

add_subdirectory(svn2git)

if(NOT BOOST_SVN)
  message(FATAL_ERROR "Please set BOOST_SVN to Boost's Subversion repository")
endif()

if(RAMDISK)
  set(git_repository "${RAMDISK}/conversion")
  set(svn_repository "${RAMDISK}/boost_svn")
  file(COPY "${BOOST_SVN}/" DESTINATION "${svn_repository}")
else()
  set(git_repository "${CMAKE_BINARY_DIR}/conversion")
  set(svn_repository "${BOOST_SVN}")
endif()

# clean
set(repositories_setup "${git_repository}/_setup")
file(GLOB ruleset_files "${CMAKE_SOURCE_DIR}/rulesets/*")
add_custom_command(OUTPUT "${repositories_setup}"
  COMMAND ${CMAKE_COMMAND} -E remove_directory "${git_repository}"
  COMMAND ${CMAKE_COMMAND} -E make_directory "${git_repository}"
  COMMAND ${CMAKE_COMMAND} -E touch "${repositories_setup}"
  DEPENDS ${ruleset_files} svn2git
  COMMENT "Rulesets changed. Cleaning temporary repositories."
  )

# collect all rulesets
file(GLOB rulesets RELATIVE "${CMAKE_BINARY_DIR}"
  "${CMAKE_SOURCE_DIR}/rulesets/libs/*"
  "${CMAKE_SOURCE_DIR}/rulesets/tools/*"
# "${CMAKE_SOURCE_DIR}/rulesets/recurse"
  "${CMAKE_SOURCE_DIR}/rulesets/website"
  )
list(SORT rulesets)

# make sure that sandbox is at the end of the list!
# we want to make sure that the rulesets of libraries with history in the
# sandbox take higher precedence.
set(sandbox "${CMAKE_SOURCE_DIR}/rulesets/sandbox")
file(RELATIVE_PATH sandbox "${CMAKE_BINARY_DIR}" "${sandbox}")
list(APPEND rulesets "${sandbox}")

set(ruleset_file "${CMAKE_BINARY_DIR}/svn2git-ruleset")
file(REMOVE "${ruleset_file}")

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
    --add-metadata
    --identity-map "${CMAKE_SOURCE_DIR}/authors.txt"
    --rules "${ruleset_file}"
    "${svn_repository}"
  DEPENDS
    ${repositories_setup}
  WORKING_DIRECTORY
    "${git_repository}"
  )

# push all ${repositories} to bitbucket
foreach(repo IN LISTS repositories)
  add_custom_target(push_${repo} ALL
    COMMAND git push --quiet --progress --mirror git@bitbucket.org:boostorg/${repo}.git
    COMMAND git push --quiet --progress --mirror git@github.com:boostorg/${repo}.git
    DEPENDS conversion
    WORKING_DIRECTORY "${git_repository}/${repo}"
    )
endforeach()

