################################################################################
# Copyright (c) 2017, 2018 IBM Corp. and others
#
# This program and the accompanying materials are made available under
# the terms of the Eclipse Public License 2.0 which accompanies this
# distribution and is available at https://www.eclipse.org/legal/epl-2.0/
# or the Apache License, Version 2.0 which accompanies this distribution and
# is available at https://www.apache.org/licenses/LICENSE-2.0.
#
# This Source Code may also be made available under the following
# Secondary Licenses when the conditions for such availability set
# forth in the Eclipse Public License, v. 2.0 are satisfied: GNU
# General Public License, version 2 with the GNU Classpath
# Exception [1] and GNU General Public License, version 2 with the
# OpenJDK Assembly Exception [2].
#
# [1] https://www.gnu.org/software/classpath/license.html
# [2] http://openjdk.java.net/legal/assembly-exception.html
#
# SPDX-License-Identifier: EPL-2.0 OR Apache-2.0 OR GPL-2.0 WITH Classpath-exception-2.0 OR LicenseRef-GPL-2.0 WITH Assembly-exception
################################################################################

# This CMakeLists is included by the VM CMake lists, and works after composition
# has occurred.
#
# Relies on a few pieces of OMR, and a few pieces of the VM CMakeLists.
#
# Longer term, this will of course have to collapse into the VM builds.

omr_add_tracegen(env/j9jit.tdf)

# this is a workarround because the jit code is inconsistent about how it includes the tracegen files
# It only really works if we put it in the source tree
add_custom_command(
	OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/env/ut_j9jit.h
	DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/ut_j9jit.h
	COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_CURRENT_BINARY_DIR}/ut_j9jit.h" "${CMAKE_CURRENT_SOURCE_DIR}/env/ut_j9jit.h"
)
add_custom_target(j9jit_tracegen DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/env/ut_j9jit.h)

set(J9SRC   ../../)
# Absolutize the J9SRC mostly to make it easier to read the paths.
get_filename_component(J9SRC ${J9SRC} ABSOLUTE)


# The Makefiles specified the OMR in the VM tree, and this does the same
# does the same.
set(OMR_ROOT ${omr_SOURCE_DIR})
# Absolutize the OMR_ROOT, which is required
# for MASM2GAS.
get_filename_component(OMR_ROOT ${OMR_ROOT} ABSOLUTE)

set(CMAKE_MODULE_PATH "${OMR_ROOT}/cmake/modules" ${CMAKE_MODULE_PATH})

#include(${OMR_ROOT}/cmake/compiler_support.cmake)
include(OmrCompilerSupport)

# Override masm2gas with J9JIT version until we've
# resolved the divergence.
set(masm2gas_path  build/scripts/masm2gas.pl )
get_filename_component(masm2gas_path ${masm2gas_path} ABSOLUTE)
set(MASM2GAS_PATH  ${masm2gas_path} CACHE INTERNAL "MASM2GAS PATH")

# The list of files that are added to the compiler in addition
# To the defaults provided by create_omr_compiler_library
set(J9JIT_FILES "" CACHE INTERNAL "The computed list of j9jit files")

# Used inside added subdirectories to help
# fill in the object list
macro(j9jit_files)
	set(J9JIT_FILES ${J9JIT_FILES} ${ARGN} CACHE INTERNAL "The computed list of j9jit files")
endmacro(j9jit_files)

j9jit_files(${CMAKE_CURRENT_BINARY_DIR}/ut_j9jit.c)
# Add our subdirs.
add_subdirectory(codegen)
add_subdirectory(compile)
add_subdirectory(control)
add_subdirectory(env)
add_subdirectory(il)
add_subdirectory(ilgen)
add_subdirectory(infra)
add_subdirectory(optimizer)
add_subdirectory(ras)
add_subdirectory(runtime)

# Should this be conditional?
if(TR_HOST_ARCH STREQUAL "x" OR TR_TARGET_ARCH STREQUAL "x")
	add_subdirectory(x)
endif()
if(TR_HOST_ARCH STREQUAL "p" OR TR_TARGET_ARCH STREQUAL "p")
	add_subdirectory(p)
endif()
if(TR_HOST_ARCH STREQUAL "z" OR TR_TARGET_ARCH STREQUAL "z")
	add_subdirectory(z)
endif()
if(TR_HOST_ARCH STREQUAL "arm" OR TR_TARGET_ARCH STREQUAL "arm")
	add_subdirectory(arm)
endif()

# Some files in OMR provide duplicate or extra functionality not needed
# in J9, so we need to remove them
set(REMOVED_OMR_FILES
	${OMR_ROOT}/compiler/control/CompileMethod.cpp
	${OMR_ROOT}/compiler/control/CompilationController.cpp
	${OMR_ROOT}/compiler/env/FEBase.cpp
	${OMR_ROOT}/compiler/optimizer/FEInliner.cpp
	${OMR_ROOT}/compiler/env/JitConfig.cpp
	${OMR_ROOT}/compiler/env/PersistentAllocator.cpp
	${OMR_ROOT}/compiler/env/SystemSegmentProvider.cpp
	${OMR_ROOT}/compiler/infra/OMRMonitor.cpp
	${OMR_ROOT}/compiler/runtime/Trampoline.cpp
	${OMR_ROOT}/compiler/runtime/Runtime.cpp
	${OMR_ROOT}/compiler/ilgen/IlBuilder.cpp
	${OMR_ROOT}/compiler/ilgen/IlValue.cpp
	${OMR_ROOT}/compiler/ilgen/IlInjector.cpp
	${OMR_ROOT}/compiler/ilgen/MethodBuilder.cpp
	${OMR_ROOT}/compiler/ilgen/BytecodeBuilder.cpp
	${OMR_ROOT}/compiler/ilgen/TypeDictionary.cpp
	${OMR_ROOT}/compiler/ilgen/ThunkBuilder.cpp
	${OMR_ROOT}/compiler/ilgen/VirtualMachineOperandStack.cpp
)

# Extra defines not provided by the create_omr_compiler_library call
set(TARGET_DEFINES
	J9_PROJECT_SPECIFIC
	_LONG_LONG #?
)

# When VM is built with CMake, we should just use the
# INTERFACE specification of target_link_libraries,
# though that would also involve teaching MASM2GAS and
# PASM2ASM about target includes.
#
# In the mean time, like the makefiles, this is is using
# the inlcudes from the SDK
set(J9_INCLUDES
	# From jitinclude.mk
	${omr_SOURCE_DIR}/thread
	${omr_SOURCE_DIR}/gc/include
	# endjitinclude.mk
	../ # Frustratingly there are some #include "trj9/frob/baz.hpp" refs  which require this.
	${J9SRC}/codert_vm
	${J9SRC}/gc_include
	${J9SRC}/gc_glue_java
	${J9SRC}/jit_vm
	${J9SRC}/nls
	${J9SRC}/oti
	${J9SRC}/include
	${J9SRC}/util
	${j9vm_BINARY_DIR}
	${omr_BINARY_DIR}
	${CMAKE_CURRENT_BINARY_DIR}/../../omr
	${CMAKE_CURRENT_BINARY_DIR}
)
# Platform specific list of flags, derived directly from the
# Makefiles
set(J9_SHAREDFLAGS
	-fno-strict-aliasing
	-Wno-deprecated
	-Wno-enum-compare
	-Wno-write-strings
	-fomit-frame-pointer
	-fasynchronous-unwind-tables
	-fno-dollars-in-identifiers
	-Wreturn-type
	-pthread
)

# Platform specific CXX flags, also derived from the
# Makefiles
set(J9_CXXFLAGS
	${J9_SHAREDFLAGS}
	-fno-rtti
	-fno-threadsafe-statics
	-Wno-invalid-offsetof
)

omr_stringify(J9_SHAREDLAGS_STR ${J9_SHAREDFLAGS})
omr_stringify(J9_CXXFLAGS_STR ${J9_CXX_FLAGS})

# Note: This is explicitly overriding what's provided by
#       the VM CMakeLists, as they pass -fno-exceptions
#       right now, and the JIT needs exceptions
set(CMAKE_CXX_FLAGS "${J9_CXXFLAGS_STR}")
set(CMAKE_C_FLAGS "${J9_SHAREDFLAGS_STR} -std=gnu89")

set(JIT_TARGET_LIBS
	j9avl
	j9codert_vm
	j9hashtable
	j9jit_vm
	j9pool
	j9stackmap
	j9util
	j9utilcore
	j9hookable
	j9thr
)


create_omr_compiler_library(NAME j9jit
	SHARED
	OBJECTS ${J9_FILES} ${J9JIT_FILES} ${NOT_EXFRRED}
	DEFINES J9_PROJECT_SPECIFIC ${TARGET_DEFINES}
	INCLUDES ${J9_INCLUDES}
	FILTER ${REMOVED_OMR_FILES}
)

add_dependencies(j9jit
	omrgc_hookgen
	j9jit_tracegen
)


target_link_libraries(j9jit
	PRIVATE
		j9vm_interface

		${JIT_TARGET_LIBS} ${JIT_TARGET_LIBS} ${JIT_TARGET_LIBS}
		m #maaaath
		${CMAKE_DL_LIBS}
)

set_property(TARGET j9jit APPEND_STRING PROPERTY
	LINK_FLAGS "  -Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/build/scripts/j9jit.linux.exp")
set_property(TARGET j9jit PROPERTY LINKER_LANGUAGE CXX)

install(
	TARGETS j9jit
	LIBRARY DESTINATION ${j9vm_SOURCE_DIR}
)
