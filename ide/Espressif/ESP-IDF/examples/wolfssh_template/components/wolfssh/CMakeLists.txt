#  Espressif component/wolfssh/CMakeLists.txt
#
#  Copyright (C) 2006-2023 WOLFSSL Inc.
#
#  This file is part of WOLFSSH.
#
#  WOLFSSH is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation; either version 2 of the License, or
#  (at your option) any later version.
#
#  WOLFSSH is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#  along with this program; if not, write to the Free Software
#  Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1335, USA
#
# cmake for WOLFSSH Espressif projects
#
# See https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html
#

cmake_minimum_required(VERSION 3.16)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWOLFSSH_USER_SETTINGS")
set(CMAKE_CURRENT_SOURCE_DIR ".")

# Attention!
#
# When editing component CMake files, consider the following :
#
# NO Managed Componenets: Normal stand-alone app, "as cloned" from github.
#   There's no notion of staging names (e.g. mywolfmqtt) regardless of environment settings.
#   All of the component source is locall. See settings such s WOLFSSL_ROOT=[your path]
#
# Partially Managed Components. This one is tricky. When publishing a component with examples,
#    those examples will have a chicken-and-egg problem: the required component is not yet published.
#    Adding to the complexity is the notion of staging components, that are purposely prefixed with
#    "my" (e.g. mywolfmqtt) to distinguish from production, live components (e.g. wolfmqtt)
#
#    Partially Managed Component Examples are typically only encountered by the component publisher
#    and only at publish time, such as when performing the pre-publish build check.
#
#    A partially managed component may also be manually created, when adding a managed component to
#    and existing project. For example:
#
#       idf.py add-dependency "wolfssl/wolfssh^1.4.15-stable"
#
# Fully Managaged Componenets. This is the typical example as created from the Component Registry:
#    For example:
#
#       idf.py create-project-from-example "wolfssl/wolfssh^1.4.15-stable:wolfssh_server"
#
# In all cases, keep in mind that components other than wolfssl will depend on the wolfssl component.
#

# Component naming is only adjusted when using Managed Components, and only when using staging site.
if( "$ENV{IDF_COMPONENT_REGISTRY_URL}" STREQUAL "https://components-staging.espressif.com" )
    # TODO: Is checking these two variables really the best way to detect an active Component Manager?
    message(STATUS "component_manager_interface_version = ${component_manager_interface_version}")
    message(STATUS "managed_components = ${managed_components}")
    if( ("${managed_components}" STREQUAL "") AND ("${component_manager_interface_version}" STREQUAL "") )
        # We've found a staging component, but did not detect the component manager
        set(WOLFSSL_COMPONENT_NAME "wolfssl")
    else()
        set(WOLFSSL_COMPONENT_NAME "mywolfssl")
    endif()
else()
    set(WOLFSSL_COMPONENT_NAME "wolfssl")
endif()

set(COMPONENT_REQUIRES lwip "${WOLFSSL_COMPONENT_NAME}")

# COMPONENT_NAME = wolfssh
# The component name is the directory name. "No feature to change this".
# See https://github.com/espressif/esp-idf/issues/8978#issuecomment-1129892685

# set the root of WOLFSSH in top-level project CMakelists.txt:
#   set(WOLFSSH_ROOT  "C:/some path/with/spaces")
#   set(WOLFSSH_ROOT  "c:/workspace/WOLFSSH-[username]")
#   set(WOLFSSH_ROOT  "/mnt/c/some path/with/spaces")
#   or use this logic to assign value from Environment Variable WOLFSSH_ROOT,
#   or assume this is an example 7 subdirectories below:

# We are typically in [root]/IDE/Espressif/ESP-IDF/examples/WOLFSSH_test/components/WOLFSSH
# The root of WOLFSSH is 7 directories up from here:

if(CMAKE_BUILD_EARLY_EXPANSION)
    message(STATUS "WOLFSSH component CMAKE_BUILD_EARLY_EXPANSION:")
    idf_component_register(
                            REQUIRES "${COMPONENT_REQUIRES}"
                            PRIV_REQUIRES
                                esp_timer
                                driver
                                "${WOLFSSL_COMPONENT_NAME}" # either wolfssl or mywolfssl as a staging component
                           )

else()
    # not CMAKE_BUILD_EARLY_EXPANSION
    message(STATUS "************************************************************************************************")
    message(STATUS "wolfssh component config:")
    message(STATUS "************************************************************************************************")

    # Check to see if we're already in WOLFSSH, and only if WOLFSSH_ROOT not specified
    if ("${WOLFSSH_ROOT}" STREQUAL "")
        # WOLFSSH examples are 7 directories deep from WOLFSSH repo root
        #                        1  2  3  4  5  6  7
        set(THIS_RELATIVE_PATH "../../../../../../..")
        get_filename_component(THIS_SEARCH_PATH  "${THIS_RELATIVE_PATH}" ABSOLUTE)
        message(STATUS "Searching in path = ${THIS_SEARCH_PATH}")

        if (EXISTS "${THIS_SEARCH_PATH}/wolfcrypt/src")
            # we're already in WOLFSSH examples!
            get_filename_component(WOLFSSH_ROOT  "${THIS_SEARCH_PATH}" ABSOLUTE)
            message(STATUS "Using WOLFSSH example with root ${WOLFSSH_ROOT}")
        else()
            # We're in some other repo such as wolfssh, so we'll search for an
            # adjacent-level directory for WOLFSSH. (8 directories up, then down one)
            #
            # For example WOLFSSH examples:
            #   C:\workspace\WOLFSSH-gojimmypi\IDE\Espressif\ESP-IDF\examples\WOLFSSH_benchmark\components\WOLFSSH
            #
            # For example wolfSSH examples:
            #   C:\workspace\wolfssh-gojimmypi\ide\Espressif\ESP-IDF\examples\wolfssh_benchmark\components\WOLFSSH
            #
            #                        1  2  3  4  5  6  7  8
            set(THIS_RELATIVE_PATH "../../../../../../../..")
            get_filename_component(THIS_SEARCH_PATH  "${THIS_RELATIVE_PATH}" ABSOLUTE)
            message(STATUS "Searching next in path = ${THIS_SEARCH_PATH}")
        endif()
    endif()

    # search other possible locations
    if ("${WOLFSSH_ROOT}" STREQUAL "")
        # there's not a hard-coded WOLFSSH_ROOT value above, so let's see if we can find it.
        if( "$ENV{WOLFSSH_ROOT}" STREQUAL "" )
            message(STATUS "Environment Variable WOLFSSH_ROOT not set. Will search common locations.")

            message(STATUS "CMAKE_CURRENT_SOURCE_DIR = ${CMAKE_CURRENT_SOURCE_DIR}")
            get_filename_component(THIS_DIR "${CMAKE_CURRENT_SOURCE_DIR}" ABSOLUTE)
            message(STATUS "THIS_DIR = ${THIS_DIR}")

            # find the user name to search for possible "WOLFSSH-username"
            message(STATUS "USERNAME = $ENV{USERNAME}")
            if(  "$ENV{USER}" STREQUAL "" ) # the bash user
                if(  "$ENV{USERNAME}" STREQUAL "" ) # the Windows user
                    message(STATUS "could not find USER or USERNAME")
                else()
                    # the bash user is not blank, so we'll use it.
                    set(THIS_USER "$ENV{USERNAME}")
                endif()
            else()
                # the bash user is not blank, so we'll use it.
               set(THIS_USER "$ENV{USER}")
            endif()
            message(STATUS "THIS_USER = ${THIS_USER}")

            # This same makefile is used for both the WOLFSSH component, and other
            # components that may depend on WOLFSSH, such as wolfssh. Therefore
            # we need to determine if this makefile is in the WOLFSSH repo, or
            # some other repo.

            if(  "{THIS_USER}" STREQUAL "" )
                # This is highly unusual to not find a user name.
                # In this case, we'll just search for a "WOLFSSH" directory:
                message(STATUS "No username found!")
                get_filename_component(WOLFSSH_ROOT  "${THIS_RELATIVE_PATH}/WOLFSSH" ABSOLUTE)
            else()
                # We found an environment USER name!
                # The first place to look for WOLFSSH will be in a user-clone called "WOLFSSH-[username]"
                message(STATUS "Using [THIS_USER = ${THIS_USER}] to see if there's a [relative path]/WOLFSSH-${THIS_USER} directory.")
                get_filename_component(WOLFSSH_ROOT  "${THIS_RELATIVE_PATH}/WOLFSSH-${THIS_USER}" ABSOLUTE)

                if( EXISTS "${WOLFSSH_ROOT}" )
                    message(STATUS "Found WOLFSSH in user-suffix ${WOLFSSH_ROOT}")
                else()
                    # If there's not a user-clone called "WOLFSSH-[username]",
                    # perhaps there's simply a git clone called "WOLFSSH"?
                    message(STATUS "Did not find WOLFSSH-${THIS_USER}; continuing search...")
                    get_filename_component(WOLFSSH_ROOT  "${THIS_RELATIVE_PATH}/WOLFSSH" ABSOLUTE)

                    if( EXISTS "${WOLFSSH_ROOT}" )
                        message(STATUS "Found WOLFSSH in standard ${WOLFSSH_ROOT}")
                    else()
                        # Things are looking pretty bleak. We'll likely not be able to compile.
                        message(STATUS "Did not find WOLFSSH in ${WOLFSSH_ROOT}")
                    endif()
                endif()
            endif()

        else()
            # there's an environment variable, so use it.
            set(WOLFSSH_ROOT "$ENV{WOLFSSH_ROOT}")

            if( EXISTS "${WOLFSSH_ROOT}" )
                get_filename_component(WOLFSSH_ROOT  "$ENV{WOLFSSH_ROOT}" ABSOLUTE)
                message(STATUS "Found WOLFSSH_ROOT via Environment Variable:")
            else()
                message(FATAL_ERROR "WOLFSSH_ROOT Environment Variable defined, but path not found:")
                message(STATUS "$ENV{WOLFSSH_ROOT}")
            endif()
        endif()
        # end of search for WOLFSSH component root
    else()
        # There's already a value assigned; we won't search for anything else.
        message(STATUS "Found user-specified WOLFSSH_ROOT value.")
    endif() # WOLFSSH_ROOT user defined

    # After all the logic above, does our WOLFSSH_ROOT actually exist?
    if( EXISTS "${WOLFSSH_ROOT}" )
        message(STATUS "WOLFSSH_ROOT = ${WOLFSSH_ROOT}")
    else()
        # Abort. We need WOLFSSH _somewhere_.
        message(FATAL_ERROR "Could not find WOLFSSH in ${WOLFSSH_ROOT}. Try setting environment variable or git clone.")
    endif()


    set(INCLUDE_PATH ${WOLFSSH_ROOT})

    set(COMPONENT_SRCDIRS
            "\"${WOLFSSH_ROOT}/src/\""
    ) # COMPONENT_SRCDIRS
    message(STATUS "This COMPONENT_SRCDIRS = ${COMPONENT_SRCDIRS}")

    set(WOLFSSH_PROJECT_DIR "${CMAKE_HOME_DIRECTORY}/components/wolfssh")

    # Espressif may take several passes through this makefile. Check to see if we found IDF
    string(COMPARE EQUAL "${PROJECT_SOURCE_DIR}" "" WOLFSSH_FOUND_IDF)

    message(STATUS "IDF_PATH = $ENV{IDF_PATH}")
    message(STATUS "PROJECT_SOURCE_DIR = ${PROJECT_SOURCE_DIR}")
    message(STATUS "EXCLUDE_ASM = ${EXCLUDE_ASM}")

    #
    # Check to see if there's both a local copy and EDP-IDF copy of the WOLFSSH and/or wolfssh components.
    #
    if( EXISTS "${WOLFSSH_PROJECT_DIR}" AND EXISTS "$ENV{IDF_PATH}/components/WOLFSSH/" )
        #
        # WOLFSSH found in both ESP-IDF and local project - needs to be resolved by user
        #
        message(STATUS "")
        message(STATUS "**************************************************************************************")
        message(STATUS "")
        message(STATUS "Error: Found components/WOLFSSH in both local project and IDF_PATH")
        message(STATUS "")
        message(STATUS "To proceed: ")
        message(STATUS "")
        message(STATUS "Remove either the local project component: ${WOLFSSH_PROJECT_DIR} ")
        message(STATUS "or the Espressif shared component installed at: $ENV{IDF_PATH}/components/WOLFSSH/ ")
        message(STATUS "")
        message(FATAL_ERROR "Please use WOLFSSH in either local project or Espressif components, but not both.")
        message(STATUS "")
        message(STATUS "**************************************************************************************")
        message(STATUS "")

        # Optional: if you change the above FATAL_ERROR to STATUS you can warn at runtime with this macro definition:
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DWOLFSSH_MULTI_INSTALL_WARNING")

    else()
        if( EXISTS "$ENV{IDF_PATH}/components/WOLFSSH/" )
            #
            # WOLFSSH found in ESP-IDF components and is assumed to be already configured in user_settings.h via setup.
            #
            message(STATUS "")
            message(STATUS "Using components/WOLFSSH in IDF_PATH = $ENV{IDF_PATH}")
            message(STATUS "")
        else()
            #
            # WOLFSSH is not an ESP-IDF component.
            # We need to now determine if it is local and if so if it is part of the WOLFSSH repo,
            # or if  WOLFSSH is simply installed as a local component.
            #

            if( EXISTS "${WOLFSSH_PROJECT_DIR}" )
                #
                # WOLFSSH found in local project.
                #
                if( EXISTS "${WOLFSSH_PROJECT_DIR}/wolfcrypt/" )
                    message(STATUS "")
                    message(STATUS "Using installed project ./components/WOLFSSH in CMAKE_HOME_DIRECTORY = ${CMAKE_HOME_DIRECTORY}")
                    message(STATUS "")
                    #
                    # Note we already checked above and confirmed there's not another WOLFSSH installed in the ESP-IDF components.
                    #
                    # We won't do anything else here, as it will be assumed the original install completed successfully.
                    #
                else() # full WOLFSSH not installed in local project
                    #
                    # This is the developer repo mode. WOLFSSH will be assumed to be not installed to ESP-IDF nor local project
                    # In this configuration, we are likely running a WOLFSSH example found directly in the repo.
                    #
                    message(STATUS "")
                    message(STATUS "Using developer repo ./components/WOLFSSH in CMAKE_HOME_DIRECTORY = ${CMAKE_HOME_DIRECTORY}")
                    message(STATUS "")

                    message(STATUS "************************************************************************************************")
                    # When in developer mode, we are typically running WOLFSSH examples such as benchmark or test directories.
                    # However, the as-cloned or distributed WOLFSSH does not have the ./include/ directory, so we'll add it as needed.
                    #
                    # first check if there's a [root]/include/user_settings.h
                    if( EXISTS "${WOLFSSH_ROOT}/include/user_settings.h" )
                        message(FATAL_ERROR "Found stray WOLFSSH user_settings.h in "
                                            "${WOLFSSH_ROOT}/include/user_settings.h "
                                            " (please move it to ${WOLFSSH_PROJECT_DIR}/include/user_settings.h )")
                    else()
                        # we won't overwrite an existing user settings file, just note that we already have one:
                        if( EXISTS "${WOLFSSH_PROJECT_DIR}/include/user_settings.h" )
                            message(STATUS "Using existing WOLFSSH user_settings.h in "
                                           "${WOLFSSH_PROJECT_DIR}/include/user_settings.h")
                        else()
                            message(STATUS "Installing WOLFSSH user_settings.h to "
                                           "${WOLFSSH_PROJECT_DIR}/include/user_settings.h")
                            # file(COPY "${WOLFSSH_ROOT}/IDE/Espressif/ESP-IDF/user_settings.h"
                            # DESTINATION "${CMAKE_HOME_DIRECTORY}/WOLFSSH/include/")
                        endif()
                    endif() # user_settings.h

                    message(STATUS "************************************************************************************************")
                    message(STATUS "")
                endif()

            else()
                # we did not find a ./components/WOLFSSH/include/ directory from this pass of cmake.
                if($WOLFSSH_FOUND_IDF)
                    message(STATUS "")
                    message(STATUS "WARNING: WOLFSSH not found.")
                    message(STATUS "")
                else()
                    # probably needs to be re-parsed by Espressif
                    message(STATUS "WOLFSSH found IDF. Project Source:${PROJECT_SOURCE_DIR}")
                endif() # else we have not found ESP-IDF yet
            endif() # else not a local WOLFSSH component

        endif() #else not an ESP-IDF component
    endif() # else not local copy and EDP-IDF WOLFSSH


    # RTOS_IDF_PATH is typically:
    # "/Users/{username}/Desktop/esp-idf/components/freertos/include/freertos"
    # depending on the environment, we may need to swap backslashes with forward slashes
    string(REPLACE "\\" "/" RTOS_IDF_PATH "$ENV{IDF_PATH}/components/freertos/FreeRTOS-Kernel/include/freertos")

    string(REPLACE "\\" "/" WOLFSSH_ROOT ${WOLFSSH_ROOT})

    if(IS_DIRECTORY "${RTOS_IDF_PATH}")
        message(STATUS "Found current RTOS path: ${RTOS_IDF_PATH}")
    else()
        # ESP-IDF prior version 4.4x has a different RTOS directory structure
        string(REPLACE "\\" "/" RTOS_IDF_PATH "$ENV{IDF_PATH}/components/freertos/include/freertos")
        if(IS_DIRECTORY "${RTOS_IDF_PATH}")
            message(STATUS "Found legacy RTOS path: ${RTOS_IDF_PATH}")
        else()
            message(STATUS "Could not find RTOS path")
        endif()
    endif()


    set(COMPONENT_ADD_INCLUDEDIRS
        # "./include" # not used! See wolfSSL include/user_settings.h
        "\"${WOLFSSH_ROOT}/\""
        "\"${WOLFSSH_ROOT}/wolfssh/\""
        "\"${RTOS_IDF_PATH}/\""
        )


    if(IS_DIRECTORY ${IDF_PATH}/components/cryptoauthlib)
        list(APPEND COMPONENT_ADD_INCLUDEDIRS "../cryptoauthlib/lib")
    endif()

    list(APPEND COMPONENT_ADD_INCLUDEDIRS "\"${WOLFSSH_ROOT}/wolfssh/\"")



    set(COMPONENT_SRCEXCLUDE
        # wolfSSH
        # TODO: we likely need to check #if !defined(WOLFSSH_MISC_INCLUDED) && !defined(NO_INLINE) && !defined(WOLFSSH_IGNORE_FILE_WARN)
        #       here in cmake if we actually want to always exclude wolfssh misc.c file. (see source; ok for demo)
        "\"${WOLFSSH_ROOT}/src/misc.c\"" # misc.c does not need to be compiled when using inline (NO_INLINE not defined))
    )

    spaces2list(COMPONENT_REQUIRES)

    separate_arguments(COMPONENT_SRCDIRS NATIVE_COMMAND "${COMPONENT_SRCDIRS}")
    separate_arguments(COMPONENT_SRCEXCLUDE NATIVE_COMMAND "${COMPONENT_SRCEXCLUDE}")
    separate_arguments(COMPONENT_ADD_INCLUDEDIRS NATIVE_COMMAND "${COMPONENT_ADD_INCLUDEDIRS}")

    #
    # See https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/build-system.html#example-component-requirements
    #
    message(STATUS "COMPONENT_SRCDIRS = ${COMPONENT_SRCDIRS}")
    message(STATUS "COMPONENT_ADD_INCLUDEDIRS = ${COMPONENT_ADD_INCLUDEDIRS}")
    message(STATUS "COMPONENT_REQUIRES = ${COMPONENT_REQUIRES}")
    message(STATUS "COMPONENT_SRCEXCLUDE = ${COMPONENT_SRCEXCLUDE}")

    #
    # see https://docs.espressif.com/projects/esp-idf/en/stable/esp32/migration-guides/release-5.x/build-system.html?highlight=space%20path
    #
    set(EXTRA_COMPONENT_DIRS "${COMPONENT_SRCDIRS}")
    idf_component_register(
                            SRC_DIRS "${COMPONENT_SRCDIRS}"
                            INCLUDE_DIRS "${COMPONENT_ADD_INCLUDEDIRS}"
                            REQUIRES "${COMPONENT_REQUIRES}"
                            EXCLUDE_SRCS "${COMPONENT_SRCEXCLUDE}"
                            PRIV_REQUIRES
                            esp_timer
                            driver
                            "${WOLFSSL_COMPONENT_NAME}"  # either wolfssl or mywolfssl as a staging component
                           )
    # some optional diagnostics
    if (1)
        get_cmake_property(_variableNames VARIABLES)
        list (SORT _variableNames)
        message(STATUS "")
        message(STATUS "ALL VARIABLES BEGIN")
        message(STATUS "")
        foreach (_variableName ${_variableNames})
            message(STATUS "${_variableName}=${${_variableName}}")
        endforeach()
        message(STATUS "")
        message(STATUS "ALL VARIABLES END")
        message(STATUS "")
    endif()

    # target_sources(WOLFSSH PRIVATE  "\"${WOLFSSH_ROOT}/WOLFSSH/\""  "\"${WOLFSSH_ROOT}/WOLFSSH/wolfcrypt\"")
endif() # CMAKE_BUILD_EARLY_EXPANSION



# check to see if there's both a local copy and EDP-IDF copy of the WOLFSSH components
if( EXISTS "${WOLFSSH_PROJECT_DIR}" AND EXISTS "$ENV{IDF_PATH}/components/WOLFSSH/" )
    message(STATUS "")
    message(STATUS "")
    message(STATUS "********************************************************************")
    message(STATUS "WARNING: Found components/WOLFSSH in both local project and IDF_PATH")
    message(STATUS "********************************************************************")
    message(STATUS "")
endif()
# end multiple component check


#
# LIBWOLFSSH_SAVE_INFO(VAR_OUPUT THIS_VAR VAR_RESULT)
#
# Save the THIS_VAR as a string in a macro called VAR_OUPUT
#
# VAR_OUPUT:  the name of the macro to define
# THIS_VAR:   the OUTPUT_VARIABLE result from a execute_process()
# VAR_RESULT: the RESULT_VARIABLE from a execute_process(); "0" if successful.
#
function ( LIBWOLFSSH_SAVE_INFO VAR_OUPUT THIS_VAR VAR_RESULT )
    # is the RESULT_VARIABLE output value 0? If so, IS_VALID_VALUE is true.
    string(COMPARE EQUAL "${VAR_RESULT}" "0" IS_VALID_VALUE)

    # if we had a successful operation, save the THIS_VAR in VAR_OUPUT
    if(${IS_VALID_VALUE})
        # strip newline chars in THIS_VAR parameter and save in VAR_VALUE
        string(REPLACE "\n" ""  VAR_VALUE  ${THIS_VAR})

        # we'll could percolate the value to the parent for possible later use
        # set(${VAR_OUPUT} ${VAR_VALUE} PARENT_SCOPE)

        # but we're only using it here in this function
        set(${VAR_OUPUT} ${VAR_VALUE})

        # we'll print what we found to the console
        message(STATUS "Found ${VAR_OUPUT}=${VAR_VALUE}")

        # the interesting part is defining the VAR_OUPUT name a value to use in the app
        add_definitions(-D${VAR_OUPUT}=\"${VAR_VALUE}\")
    else()
        # if we get here, check the execute_process command and parameters.
        message(STATUS "LIBWOLFSSH_SAVE_INFO encountered a non-zero VAR_RESULT")
        set(${VAR_OUPUT} "Unknown")
    endif()
endfunction() # LIBWOLFSSH_SAVE_INFO

# create some programmatic #define values that will be used by ShowExtendedSystemInfo().
# see wolfcrypt\src\port\Espressif\esp32_utl.c
if(NOT CMAKE_BUILD_EARLY_EXPANSION)
    set (git_cmd "git")
    message(STATUS "Adding macro definitions:")

    # LIBWOLFSSH_VERSION_GIT_ORIGIN: git config --get remote.origin.url
    execute_process(WORKING_DIRECTORY ${WOLFSSH_ROOT} COMMAND ${git_cmd} "config" "--get" "remote.origin.url" OUTPUT_VARIABLE TMP_OUT RESULT_VARIABLE TMP_RES ERROR_QUIET  )
    LIBWOLFSSH_SAVE_INFO(LIBWOLFSSH_VERSION_GIT_ORIGIN "${TMP_OUT}" "${TMP_RES}")

    # LIBWOLFSSH_VERSION_GIT_BRANCH: git rev-parse --abbrev-ref HEAD
    execute_process(WORKING_DIRECTORY ${WOLFSSH_ROOT} COMMAND ${git_cmd} "rev-parse" "--abbrev-ref" "HEAD" OUTPUT_VARIABLE TMP_OUT RESULT_VARIABLE TMP_RES ERROR_QUIET  )
    LIBWOLFSSH_SAVE_INFO(LIBWOLFSSH_VERSION_GIT_BRANCH "${TMP_OUT}" "${TMP_RES}")

    # LIBWOLFSSH_VERSION_GIT_HASH: git rev-parse HEAD
    execute_process(WORKING_DIRECTORY ${WOLFSSH_ROOT} COMMAND ${git_cmd} "rev-parse" "HEAD" OUTPUT_VARIABLE TMP_OUT RESULT_VARIABLE TMP_RES ERROR_QUIET  )
    LIBWOLFSSH_SAVE_INFO(LIBWOLFSSH_VERSION_GIT_HASH "${TMP_OUT}" "${TMP_RES}")

    # LIBWOLFSSH_VERSION_GIT_SHORT_HASH: git rev-parse --short HEAD
    execute_process(WORKING_DIRECTORY ${WOLFSSH_ROOT} COMMAND ${git_cmd} "rev-parse" "--short" "HEAD" OUTPUT_VARIABLE TMP_OUT RESULT_VARIABLE TMP_RES ERROR_QUIET )
    LIBWOLFSSH_SAVE_INFO(LIBWOLFSSH_VERSION_GIT_SHORT_HASH "${TMP_OUT}" "${TMP_RES}")

    # LIBWOLFSSH_VERSION_GIT_HASH_DATE git show --no-patch --no-notes --pretty=\'\%cd\'
    execute_process(WORKING_DIRECTORY ${WOLFSSH_ROOT} COMMAND ${git_cmd} "show" "--no-patch" "--no-notes" "--pretty=\'\%cd\'" OUTPUT_VARIABLE TMP_OUT RESULT_VARIABLE TMP_RES  )
    LIBWOLFSSH_SAVE_INFO(LIBWOLFSSH_VERSION_GIT_HASH_DATE "${TMP_OUT}" "${TMP_RES}")

    message(STATUS "************************************************************************************************")
    message(STATUS "WOLFSSH component config complete!")
    message(STATUS "************************************************************************************************")
endif()
