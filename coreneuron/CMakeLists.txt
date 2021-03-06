# Copyright (c) 2016, Blue Brain Project
# All rights reserved.

# Redistribution and use in source and binary forms, with or without modification,
# are permitted provided that the following conditions are met:
# 1. Redistributions of source code must retain the above copyright notice,
#    this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright notice,
#    this list of conditions and the following disclaimer in the documentation
#    and/or other materials provided with the distribution.
# 3. Neither the name of the copyright holder nor the names of its contributors
#    may be used to endorse or promote products derived from this software
#    without specific prior written permission.

# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF
# THE POSSIBILITY OF SUCH DAMAGE.


include_directories(utils/randoms)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${MPI_INCLUDE_PATH})

FILE(GLOB_RECURSE coreneuron_all_headers "*.h")
FILE(GLOB_RECURSE coreneuron_all_templates "*.ipp")
FILE(GLOB_RECURSE coreneuron_all_c_files "*.c*")
file(GLOB_RECURSE coreneuron_cuda_files "*.cu")

# Compile and include MOD files

find_program(MOD2C mod2c_core)
if(MOD2C STREQUAL "MOD2C-NOTFOUND")
message(FATAL_ERROR "Unable to locate mod2c_core executable")
endif()

# mod2c_core wants MODLUNIT set to point to its units file,
# but CMake does not support setting an environment for custom_commands.

get_filename_component(mod2c_install_prefix ${MOD2C} DIRECTORY)
find_file(MOD2C_UNITS nrnunits.lib HINT "${mod2c_install_prefix}" PATH_SUFFIXES "share")
if(NOT MOD2C_UNITS STREQUAL "MOD2C_UNITS-NOTFOUND")
set(MOD2C env "MODLUNIT=${MOD2C_UNITS}" ${MOD2C})
endif()

# Macro sets up build rule for .c files from .mod files.
# Parameters:
#    name     An arbitrary name to keep track of output .c files
#    input    The path to the mod file
#
# Because mod2c_core wants to write the .c file in the same directory
# as the mod file, we copy the mod file to the binary directory first
#
# The macro appends the names of the output files to MOD2C_${name}_OUTPUTS
# and the names of the mod files (without directories) to MOD2C_${name}_MODS

macro(mod2c_target name input)
    get_filename_component(mod2c_source_ ${input} ABSOLUTE)
    get_filename_component(mod2c_modname_ ${input} NAME)
    string(REGEX REPLACE "\\.mod$" ".c" mod2c_cname_ "${mod2c_modname_}")

    set(mod2c_output_ "${CMAKE_CURRENT_BINARY_DIR}/${mod2c_cname_}")
    list(APPEND MOD2C_${name}_OUTPUTS "${mod2c_output_}")
    list(APPEND MOD2C_${name}_MODS "${mod2c_modname_}")

    add_custom_command(OUTPUT "${mod2c_output_}"
      DEPENDS ${input} "${mod2c_install_prefix}/mod2c_core"
      COMMAND ${CMAKE_COMMAND} -E copy "${mod2c_source_}" "${CMAKE_CURRENT_BINARY_DIR}"
      COMMAND ${MOD2C} "${mod2c_modname_}"
      WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
endmacro()

# Macro mod2c_from_file attempts to add modules specified in a file.
# Parameters:
#     name        Key for mod2c_target
#     modlist     Path to the file containing the list of mod files.
#     searchpath  List of directories to search for mod files.
#
# This has to be a macro in order to properly accumulate the MOD2C_xxx_OUTPUTS
# variables. Go CMake! You rock!

macro(mod2c_from_file name modlist searchpath)
    unset(tmp_path_ CACHE)
    if(NOT EXISTS "${modlist}")
	string(REGEX REPLACE ":" ";" spath "${searchpath}")
	foreach(path_ ${spath})
            file(GLOB_RECURSE mod_files "${path_}/*.mod")
            list(APPEND mods_ "${mod_files};")
        endforeach()
    else()
	file(READ "${modlist}" mods_)
    endif()
    string(REGEX REPLACE "[ \t\n]+" ";" mods_ "${mods_}")
    list(REMOVE_ITEM mods_ "")

    foreach(mod_ ${mods_})
        unset(modpath_)
        if(NOT IS_ABSOLUTE "${mod_}")
            find_path(tmp_path_ "${mod_}" PATH ${searchpath} NO_DEFAULT_PATH)
	    if(tmp_path_)
	        set(modpath_ "${tmp_path_}/${mod_}")
	    endif()
	    unset(tmp_path_ CACHE)
        else()
            set(modpath_ "${mod_}")
	endif()
	if((NOT EXISTS "${modpath_}") OR (IS_DIRECTORY "${modpath_}"))
	    unset(modpath_)
        endif()

        if(modpath_)
            mod2c_target(${name} "${modpath_}")
            list(APPEND MOD_PATHS ${modpath_})
        else()
	    message(WARNING "Unable to locate MOD file: ${mod_}")
        endif()
    endforeach()
endmacro()

# Standard MOD files are always built

set(STDMODLIBPATH "${CMAKE_CURRENT_SOURCE_DIR}/mech/modfile")
set(MECH_STDMODS expsyn.mod  hh.mod  netstim.mod  passive.mod  pattern.mod stim.mod)

foreach(MOD ${MECH_STDMODS})
  mod2c_target(STDMECH "${STDMODLIBPATH}/${MOD}")
endforeach()

set(MECH_FILE_LISTS)

# Additional optional mechanisms specified in ADDITIONAL_MECHS
# and ADDITIONAL_MECHPATH (set as a PATH cache variable, but
# can in fact be a list of paths.)
# If ADDITIONAL_MECHS is not provided, the entire ADDITIONAL_MECHPATH
# will be checked for the mod files.

if(EXISTS ${ADDITIONAL_MECHS})
    if(IS_DIRECTORY "${modlist}")
        message(WARNING "Unable to read MOD list file: ${modlist}")
    else()
        list(APPEND MECH_FILE_LISTS "${ADDITIONAL_MECHS}")
    endif()
endif()
mod2c_from_file(OPTMECH "${ADDITIONAL_MECHS}" "${ADDITIONAL_MECHPATH}")

# For 'non-standard' mod files, need to generate registering function in mod_func.c

set(MOD_FUNC_C    "${CMAKE_CURRENT_BINARY_DIR}/mod_func.c")
set(MOD_FUNC_C_PL "${CMAKE_CURRENT_SOURCE_DIR}/mech/mod_func.c.pl")

# ... pass as argument to the perl script the list of mod file names;
add_custom_command(OUTPUT "${MOD_FUNC_C}"
    DEPENDS ${MECH_FILE_LISTS} "${MOD_FUNC_C_PL}"
    COMMAND perl "${MOD_FUNC_C_PL}" ${MOD2C_OPTMECH_MODS} > "${MOD_FUNC_C}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")

# ... and add a reconfigure dependency on the files that listed them

foreach(depfile ${MECH_FILE_LISTS})
    configure_file("${depfile}" "${CMAKE_CURRENT_BINARY_DIR}/dummy_reconfigure_dep")
endforeach()

# to work around no acc ability to pass function pointers as arguments,
# some translated c files depend on a _kinderiv.h file that is constructed
# by kinderiv.py

set(KINDERIV_PY "${CMAKE_CURRENT_SOURCE_DIR}/kinderiv.py")
set(KINDERIV_H "${CMAKE_CURRENT_BINARY_DIR}/_kinderiv.h")

add_custom_command(OUTPUT "${KINDERIV_H}"
    DEPENDS ${MOD2C_OPTMECH_OUTPUTS} ${MOD2C_STDMECH_OUTPUTS} "${KINDERIV_PY}"
    COMMAND python "${KINDERIV_PY}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")

if (EXPORT_MECHS_FUNCTIONS)
  # Create C file with all "get function pointers" methods
  set(MOD_FUNC_PTRS_C    "${CMAKE_CURRENT_BINARY_DIR}/mod_func_ptrs.c")
  set(MOD_FUNC_PTRS_C_PL "${CMAKE_CURRENT_SOURCE_DIR}/mech/mod_func_ptrs.c.pl")

  # ... pass as argument to the perl script the list of mods full paths;
  add_custom_command(OUTPUT "${MOD_FUNC_PTRS_C}"
      DEPENDS ${MECH_FILE_LISTS} "${MOD_FUNC_PTRS_C_PL}"
      COMMAND perl "${MOD_FUNC_PTRS_C_PL}" ${MOD_PATHS} > "${MOD_FUNC_PTRS_C}"
      WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
endif()

# Add generated sources from MOD files

set(GENERATED_MECH_C_FILES ${MOD_FUNC_C} ${MOD_FUNC_PTRS_C} ${MOD2C_STDMECH_OUTPUTS} ${MOD2C_OPTMECH_OUTPUTS})

# artificial cells must be on cpu, defaul nrnran123.c is for cpu, nrn_setup.cpp uses nrnran123 for only memory calculation purpose which should use cpu version of nrnran123
set(NOACC_MECH_C_FILES ${CMAKE_CURRENT_BINARY_DIR}/netstim.c ${CMAKE_CURRENT_BINARY_DIR}/netstim_inhpoisson.c ${CMAKE_CURRENT_BINARY_DIR}/pattern.c ${CMAKE_CURRENT_SOURCE_DIR}/utils/randoms/nrnran123.c ${CMAKE_CURRENT_SOURCE_DIR}/nrniv/nrn_setup.cpp)

if(ENABLE_OPENACC)
    set_source_files_properties(${GENERATED_MECH_C_FILES} PROPERTIES COMPILE_FLAGS "")
    set_source_files_properties(${NOACC_MECH_C_FILES} PROPERTIES COMPILE_FLAGS "-DDISABLE_OPENACC")
    if(${CMAKE_C_COMPILER_ID} STREQUAL "PGI")
        set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/scopmath_core/sparse_thread.c PROPERTIES COMPILE_FLAGS "-ta=tesla:nollvm")
    endif()
endif()

set(coreneuron_all_c_files ${coreneuron_all_c_files} ${GENERATED_MECH_C_FILES})

#to indent generated c/cpp files
if(CLANG_FORMAT_FOUND)
    add_custom_target(formatbuild
        COMMAND ${CMAKE_COMMAND}
            -DSOURCE_FILES:STRING="${GENERATED_MECH_C_FILES}"
            -DCLANG_FORMAT_EXECUTABLE=${CLANG_FORMAT_EXECUTABLE}
            -P "${PROJECT_SOURCE_DIR}/CMake/ClangFormatUtils.cmake"
        DEPENDS ${GENERATED_MECH_C_FILES}
        )
endif()


add_library(coreneuron ${COMPILE_LIBRARY_TYPE} ${coreneuron_all_headers} ${coreneuron_all_templates} ${coreneuron_all_c_files} ${KINDERIV_H})

set(link_cudacoreneuron)
set(link_reportinglib)

#@TODO: CMake should have option for arch
if(ENABLE_OPENACC)
    cuda_add_library("cudacoreneuron" ${coreneuron_cuda_files} OPTIONS -arch=sm_20)
    set(link_cudacoreneuron cudacoreneuron)
endif()

if(ENABLE_REPORTINGLIB)
  set(link_reportinglib ${REPORTINGLIB_LIBRARIES})
endif()
target_link_libraries(coreneuron ${MPI_C_LIBRARIES}
    ${link_reportinglib} ${link_cudacoreneuron} ${CUDA_LIBRARIES} )


set_target_properties(coreneuron PROPERTIES
                            VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}
                            SOVERSION ${VERSION_MAJOR}
                            CLEAN_DIRECT_OUTPUT 1)

install(TARGETS coreneuron
             LIBRARY DESTINATION ${LIB_INSTALL_DIR}
             ARCHIVE DESTINATION ${LIB_INSTALL_DIR} )
