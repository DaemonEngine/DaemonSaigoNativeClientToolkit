# Author: Thomas Debesse
# License: CC0 1.0

cmake_minimum_required(VERSION 3.5)

include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)

project(SaigoNativeClientToolchain)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
	set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/prefix" CACHE PATH "Install path prefix, prepended onto install directories." FORCE)
endif ()

get_property(CMAKE_BUILD_TYPE_HELP CACHE CMAKE_BUILD_TYPE PROPERTY HELPSTRING)
set(CMAKE_BUILD_TYPE "Release" CACHE STRING "${CMAKE_BUILD_TYPE_HELP}" FORCE)
set(COMPILER_FLAGS "-O3")

option(USE_LTO "Enable link-time optimization." OFF)
option(USE_CCACHE "Enable cached compilation." ON)

if (USE_LTO)
	set(CONFIGURE_ARG_LTO --enable-lto)
endif()

if (USE_CCACHE)
	set(CCACHE_CMD "ccache")
endif()

option(BUILD_CLANG "Build the NaCl Saigo clang compiler." ON)
option(BUILD_BINUTILS "Build the NaCl binutils." ON)

set(GIT_REPOSITORY_CLANG "https://chromium.googlesource.com/native_client/nacl-llvm-project-v10.git" CACHE STRING "Git repository location.")
set(GIT_REPOSITORY_BINUTILS "https://chromium.googlesource.com/native_client/nacl-binutils.git" CACHE STRING "Git repository location.")

mark_as_advanced(GIT_REPOSITORY_CLANG)
mark_as_advanced(GIT_REPOSITORY_BINUTILS)

set(ARCH_PREFIXES x86_64;i686;arm)

set(EXTERNAL_PROJECT_BASE ExternalProjects)
set(EXTERNAL_PROJECT_SOURCES_DIR ${EXTERNAL_PROJECT_BASE}/Sources)

set_directory_properties(PROPERTIES EP_BASE ${CMAKE_CURRENT_BINARY_DIR}/${EXTERNAL_PROJECT_BASE})

macro(AddAliases targetName toolNames)
	add_custom_target(${targetName} ALL)
	add_custom_target(${targetName}-aliases ALL)

	add_dependencies(${targetName} ${targetName}-binaries ${targetName}-aliases)

	foreach(archPrefix ${ARCH_PREFIXES})
		foreach(toolName ${toolNames})
			set(toolFullName ${archPrefix}-nacl-${toolName})

			add_custom_target(${toolFullName}
				ALL
				COMMAND ln -sf ${toolName} ${CMAKE_INSTALL_PREFIX}/bin/${toolFullName}
				DEPENDS ${targetName}-binaries
			)

			add_dependencies(${targetName}-aliases ${toolFullName})
		endforeach()
	endforeach()
endmacro()

if (BUILD_CLANG)
	set(sourceDir ${EXTERNAL_PROJECT_SOURCES_DIR}/llvm-project)

	ExternalProject_Add(clang-binaries
		SOURCE_DIR ${sourceDir}
		GIT_REPOSITORY ${GIT_REPOSITORY_CLANG}
		GIT_TAG main
		SOURCE_SUBDIR llvm
		CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
		CMAKE_ARGS -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
		CMAKE_ARGS -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
		CMAKE_ARGS -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
		CMAKE_ARGS -DCMAKE_INSTALL_RPATH=%s/../lib
		CMAKE_ARGS -DLLVM_INSTALL_TOOLCHAIN_ONLY=ON
		CMAKE_ARGS -DLLVM_APPEND_VC_REV=ON
		CMAKE_ARGS -DLLVM_BUILD_TESTS=OFF
		CMAKE_ARGS -DLLVM_ENABLE_ASSERTIONS=ON
		CMAKE_ARGS -DLLVM_ENABLE_LIBCXX=OFF
		CMAKE_ARGS -DLLVM_ENABLE_ZLIB=OFF
		CMAKE_ARGS -DLLVM_ENABLE_ZSTD=OFF
		CMAKE_ARGS -DLLVM_ENABLE_Z3_SOLVER=OFF
		CMAKE_ARGS -DLLVM_INSTALL_UTILS=ON
		CMAKE_ARGS -DLLVM_ENABLE_LIBXML2=OFF
		CMAKE_ARGS "-DLLVM_TARGETS_TO_BUILD=X86;ARM"
		CMAKE_ARGS -DLLVM_ENABLE_PROJECTS=clang
		CMAKE_ARGS -DLLVM_BUILD_LLVM_DYLIB=OFF
		CMAKE_ARGS -DLLVM_LINK_LLVM_DYLIB=OFF
		CMAKE_ARGS -DCLANG_LINK_CLANG_DYLIB=OFF
		CMAKE_ARGS -DLLVM_CCACHE_BUILD=OFF
		# CMAKE_ARGS -DLLVM_ENABLE_TERMINFO=OFF
		CMAKE_ARGS -DLLVM_ENABLE_RUNTIMES=
		CMAKE_ARGS -DCLANG_INCLUDE_DOCS=OFF
		CMAKE_ARGS -DCLANG_INCLUDE_TESTS=OFF
		CMAKE_ARGS -DLLVM_ENABLE_LTO=${USE_LTO}
		CMAKE_ARGS -DLLVM_CCACHE_BUILD=${USE_CCACHE}
		INSTALL_COMMAND ${CMAKE_COMMAND} --install . --strip
	)
endif()

set(CLANG_TOOL_NAMES clang;clang++)

AddAliases(clang "${CLANG_TOOL_NAMES}")

if (BUILD_BINUTILS)
	set(sourceDir ${EXTERNAL_PROJECT_SOURCES_DIR}/binutils)
	set(sourceFullDir ${CMAKE_CURRENT_BINARY_DIR}/${sourceDir})

	ExternalProject_Add(binutils-binaries
		SOURCE_DIR ${sourceDir}
		GIT_REPOSITORY ${GIT_REPOSITORY_BINUTILS}
		GIT_TAG main
		CONFIGURE_COMMAND ${sourceFullDir}/configure
		CONFIGURE_COMMAND LD=lld
		CONFIGURE_COMMAND "CC=${CCACHE_CMD} ${CMAKE_C_COMPILER}"
		CONFIGURE_COMMAND "CFLAGS=${COMPILER_FLAGS} -Wno-absolute-value -Wno-unused-function -Wno-unused-const-variable -Wno-unneeded-internal-declaration -Wno-unused-private-field -Wno-format-security"
		# CONFIGURE_COMMAND "LDFLAGS=-fuse-ld=lld -static-libstdc++ -static-libgcc"
		CONFIGURE_COMMAND "LDFLAGS=-static-libstdc++ -static-libgcc"
		CONFIGURE_COMMAND ${CONFIGURE_ARG_LTO}
		CONFIGURE_COMMAND --prefix=
		CONFIGURE_COMMAND --enable-targets=x86_64-nacl,i686-nacl,arm-nacl
		# CONFIGURE_COMMAND --enable-gold --enable-plugins --disable-shared
		CONFIGURE_COMMAND --disable-gold
		CONFIGURE_COMMAND --disable-werror
		CONFIGURE_COMMAND --without-zlib
		CONFIGURE_COMMAND --disable-silent-rules
		CONFIGURE_COMMAND --enable-deterministic-archives
		BUILD_COMMAND $(MAKE) MAKEINFO=true
		INSTALL_COMMAND $(MAKE) MAKEINFO=true install-strip
		INSTALL_COMMAND DESTDIR=${CMAKE_INSTALL_PREFIX}
	)
endif()

set(BINUTILS_TOOL_NAMES addr2line;ar;as;c++filt;elfedit;gprof;nm;objcopy;objdump;ranlib;readelf;size;strings;strip;ld;ld.bfd) # ld.gold

AddAliases(binutils "${BINUTILS_TOOL_NAMES}")
