# Author: Thomas Debesse
# License: CC0 1.0

cmake_minimum_required(VERSION 3.10)

include(${CMAKE_ROOT}/Modules/ExternalProject.cmake)

project(SaigoNativeClientToolchain)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
	set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/prefix" CACHE PATH "Install path prefix, prepended onto install directories." FORCE)
endif ()

get_property(CMAKE_BUILD_TYPE_HELP CACHE CMAKE_BUILD_TYPE PROPERTY HELPSTRING)
set(CMAKE_BUILD_TYPE "Release" CACHE STRING "${CMAKE_BUILD_TYPE_HELP}" FORCE)
set(COMPILER_FLAGS "-O3")

find_program(FOUND_CCACHE NAMES "ccache")

option(USE_CCACHE "Enable cached compilation." FOUND_CCACHE)
option(USE_LTO "Enable link-time optimization." ON)

if (USE_LTO)
	set(CONFIGURE_ARG_LTO --enable-lto)
endif()

if (USE_CCACHE)
	set(CCACHE_CMD "ccache")
endif()

if (CONFIGURE_HOST)
	set(CONFIGURE_ARG_HOST "--host=${CONFIGURE_HOST}")
endif()

if (USE_CCACHE)
	set(CONFIGURE_CC "${CCACHE_CMD} ${CMAKE_C_COMPILER}")
else()
	set(CONFIGURE_CC "${CMAKE_C_COMPILER}")
endif()

if (CMAKE_OSX_DEPLOYMENT_TARGET)
	set(COMPILER_FLAGS "${COMPILER_FLAGS} -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET}")
endif()

option(CLONE_SHARED_REPOSITORIES "Clone shared sources dir (and only do that)." OFF)

if (CLONE_SHARED_REPOSITORIES)
	set(DEFAULT_SHARED_REPOSITORIES ${CMAKE_CURRENT_BINARY_DIR})
else()
	set(DEFAULT_SHARED_REPOSITORIES "")
endif()

set(SHARED_REPOSITORIES_DIR ${DEFAULT_SHARED_REPOSITORIES_DIR} CACHE PATH "Shared repositories dir (doesn't use any if empty).")

set(EXTERNAL_PROJECT_BASE ExternalProjects)
set_directory_properties(PROPERTIES EP_BASE ${CMAKE_CURRENT_BINARY_DIR}/${EXTERNAL_PROJECT_BASE})

if (CLONE_SHARED_REPOSITORIES)
	set(EXTERNAL_PROJECT_SOURCES_DIR ${CMAKE_BINARY_DIR})
else()
	set(EXTERNAL_PROJECT_SOURCES_DIR ${EXTERNAL_PROJECT_BASE}/Sources)
endif()

macro(AddProject SLUG NAME DIR URL TAG)
	set(GIT_REPOSITORY_${SLUG} ${URL} CACHE STRING "${NAME} git repository location.")
	mark_as_advanced(GIT_REPOSITORY_${SLANG})

	set(REPOSITORY_DIR_${SLUG} ${DIR})
	set(REPOSITORY_TAG_${SLUG} ${TAG})

	if (SHARED_REPOSITORIES_DIR)
		set(REPOSITORY_${SLUG} ${SHARED_REPOSITORIES_DIR}/${DIR})
	else()
		set(REPOSITORY_${SLUG} ${GIT_REPOSITORY_${SLUG}})
	endif()

	set(SOURCE_DIR_${SLUG} ${EXTERNAL_PROJECT_SOURCES_DIR}/${DIR})

	if (CLONE_SHARED_REPOSITORIES)
		option(CLONE_${SLUG} "Clone the ${NAME} repository." ON)

		if (CLONE_${SLUG})
			ExternalProject_Add(${REPOSITORY_DIR_${SLUG}}-shared-repository
				SOURCE_DIR ${DIR}
				GIT_REPOSITORY ${GIT_REPOSITORY_${SLUG}}
				GIT_TAG ${TAG}
				CONFIGURE_COMMAND echo
				BUILD_COMMAND echo
				INSTALL_COMMAND echo
			)
		endif()
	else()
		option(BUILD_${SLUG} "Build the ${NAME}." ON)
	endif()
endmacro()

AddProject("CLANG"
	"NaCl Saigo clang compiler"
	"llvm-project"
	"https://chromium.googlesource.com/native_client/nacl-llvm-project-v10.git"
	"main"
)
AddProject("BINUTILS"
	"NaCl binutils"
	"binutils"
	"https://chromium.googlesource.com/native_client/nacl-binutils.git"
	"main"
)

set(REFERENCE_ARCH_NAME x86_64)
set(ALIAS_ARCH_NAMES i686;arm)

macro(RenameBinaryAliases targetName toolNames)
	add_custom_target(${targetName}-renames ALL)
	add_dependencies(${targetName} ${targetName}-renames)
	add_dependencies(${targetName} ${targetName}-renames ${targetName}-binaries)

	set(targetPath ${CMAKE_INSTALL_PREFIX}/bin/${targetName}${CMAKE_EXECUTABLE_SUFFIX})
	set(referenceName ${targetName}-reference)
	set(referencePath ${EXTERNAL_PROJECT_BASE}/tmp/${referenceName}${CMAKE_EXECUTABLE_SUFFIX})

	add_custom_target(${referenceName}
		ALL
		COMMAND cp -P ${targetPath} ${referencePath}
		DEPENDS ${targetName}-binaries
	)

	add_dependencies(${targetName} ${referenceName})

	foreach(toolName ${toolNames})
		set(aliasName ${REFERENCE_ARCH_NAME}-nacl-${toolName})
		set(aliasPath ${CMAKE_INSTALL_PREFIX}/bin/${aliasName}${CMAKE_EXECUTABLE_SUFFIX})

		add_custom_target(${aliasName}
			ALL
			COMMAND cp -P ${referencePath} ${aliasPath}
			DEPENDS ${referenceName}
		)

		add_dependencies(${targetName}-renames ${aliasName})
	endforeach()
endmacro()

macro(AddBinaryAliases targetName toolNames)
	set(targetAliases ${targetName}-aliases)
	add_custom_target(${targetName}-aliases ALL)
	add_dependencies(${targetName} ${targetName}-aliases)
	add_dependencies(${targetName}-aliases ${targetName}-binaries)

	foreach(archName ${ALIAS_ARCH_NAMES})
		foreach(toolName ${toolNames})
			set(referenceName ${REFERENCE_ARCH_NAME}-nacl-${toolName}${CMAKE_EXECUTABLE_SUFFIX})
			set(aliasName ${archName}-nacl-${toolName})
			set(aliasPath ${CMAKE_INSTALL_PREFIX}/bin/${aliasName}${CMAKE_EXECUTABLE_SUFFIX})

			add_custom_target(${aliasName}
				ALL
				COMMAND ln -sf ${referenceName} ${aliasPath}
				DEPENDS ${targetName}-binaries
			)

			add_dependencies(${targetName}-aliases ${aliasName})
		endforeach()
	endforeach()
endmacro()

macro(AddDirectoryAliases targetName toolNames)
	add_custom_target(${targetName}-directory-aliases ALL)
	add_dependencies(${targetName} ${targetName}-directory-aliases)
	add_dependencies(${targetName}-directory-aliases ${targetName}-binaries)

	set(referenceName ${REFERENCE_ARCH_NAME}-nacl)

	foreach(archName ${ALIAS_ARCH_NAMES})
		set(aliasName ${archName}-nacl)
		set(aliasPath ${CMAKE_INSTALL_PREFIX}/${aliasName})

		add_custom_target(${aliasName}-directory
			ALL
			COMMAND mkdir -p ${aliasPath}
			DEPENDS ${targetName}-binaries
		)

		add_dependencies(${targetName}-directory-aliases ${aliasName}-directory)

		add_custom_target(${aliasName}-bin-directory
			ALL
			COMMAND ln -sfT ../${referenceName}/bin ${aliasPath}/bin
			DEPENDS ${aliasName}-directory
		)

		add_dependencies(${targetName}-directory-aliases ${aliasName}-bin-directory)

		add_custom_target(${aliasName}-lib-directory
			ALL
			COMMAND mkdir -p ${aliasPath}/lib
			DEPENDS ${aliasName}-directory
		)

		add_dependencies(${targetName}-directory-aliases ${aliasName}-lib-directory)

		add_custom_target(${aliasName}-ldscripts-directory
			ALL
			COMMAND ln -sfT ../../${referenceName}/lib/ldscripts ${aliasPath}/lib/ldscripts
			DEPENDS ${aliasName}-lib-directory
		)

		add_dependencies(${targetName}-directory-aliases ${aliasName}-ldscripts-directory)
	endforeach()

	foreach(toolName ${toolNames})
		set(toolPath ${CMAKE_INSTALL_PREFIX}/${referenceName}/bin/${toolName})

		add_custom_target(${targetName}-${toolName}-delete
			ALL
			COMMAND rm -f ${toolPath}
			DEPENDS ${targetName}-binaries
		)

		add_dependencies(${targetName}-directory-aliases ${targetName}-${toolName}-delete)

		add_custom_target(${targetName}-${toolName}-alias
			ALL
			COMMAND ln -sf ../../bin/${referenceName}-${toolName} ${toolPath}
			DEPENDS ${targetName}-${toolName}-delete
		)

		add_dependencies(${targetName}-directory-aliases ${targetName}-${toolName}-alias)
	endforeach()
endmacro()

if (BUILD_CLANG)
	add_custom_target(clang ALL)

	set(INSTALL_RPATH "@executable_path/../lib")

	if (APPLE)
		set(INSTALL_RPATH "$ORIGIN/../lib")
	endif()

	set(USE_IPO ON)
	set(USE_LTO OFF)

	if (MINGW)
		set(USE_IPO OFF)
		set(USE_LTO OFF)
	elseif (CMAKE_C_COMPILER_ID STREQUAL "Clang" OR CMAKE_C_COMPILER_ID STREQUAL "AppleClang")
		set(USE_IPO OFF)
		set(USE_LTO Thin)
	endif()

	if (CMAKE_OSX_DEPLOYMENT_TARGET)
		set(CMAKE_EXTRA_ARGS "-DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}")
	endif()

	ExternalProject_Add(clang-binaries
		SOURCE_DIR "${SOURCE_DIR_CLANG}"
		GIT_REPOSITORY "${REPOSITORY_CLANG}"
		GIT_TAG "${REPOSITORY_TAG_CLANG}"
		SOURCE_SUBDIR llvm
		LIST_SEPARATOR |
		CMAKE_ARGS "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}"
		CMAKE_ARGS "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
		CMAKE_ARGS "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
		CMAKE_ARGS "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
		CMAKE_ARGS "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}"
		CMAKE_ARGS "-DCMAKE_INSTALL_RPATH=${INSTALL_RPATH}"
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
		CMAKE_ARGS "-DLLVM_TARGETS_TO_BUILD=X86|ARM"
		CMAKE_ARGS -DLLVM_ENABLE_PROJECTS=clang
		CMAKE_ARGS "-DLLVM_DEFAULT_TARGET_TRIPLE=x86_64-nacl"
		CMAKE_ARGS -DLLVM_BUILD_LLVM_DYLIB=OFF
		CMAKE_ARGS -DLLVM_LINK_LLVM_DYLIB=OFF
		CMAKE_ARGS -DCLANG_LINK_CLANG_DYLIB=OFF
		CMAKE_ARGS -DLLVM_CCACHE_BUILD=OFF
		# CMAKE_ARGS -DLLVM_ENABLE_TERMINFO=OFF
		CMAKE_ARGS -DLLVM_ENABLE_RUNTIMES=
		CMAKE_ARGS -DCLANG_INCLUDE_DOCS=OFF
		CMAKE_ARGS -DCLANG_INCLUDE_TESTS=OFF
		CMAKE_ARGS -DLLVM_CCACHE_BUILD=${USE_CCACHE}
		CMAKE_ARGS -DLLVM_USE_SYMLINKS=ON
		CMAKE_ARGS -DLLVM_USE_LTO=${USE_LTO}
		CMAKE_ARGS -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=${USE_IPO}
		CMAKE_ARGS ${CMAKE_EXTRA_ARGS}
		INSTALL_COMMAND ${CMAKE_COMMAND} --install . --strip
	)

	set(clangToolNames clang;clang++)

	RenameBinaryAliases(clang "${clangToolNames}")
	AddBinaryAliases(clang "${clangToolNames}")
endif()

if (BUILD_BINUTILS)
	add_custom_target(binutils ALL)

	if (MINGW OR CMAKE_SYSTEM_NAME STREQUAL "Linux")
		set(STATIC_LIBGCC "-static-libgcc")
	endif()

	ExternalProject_Add(binutils-binaries
		SOURCE_DIR "${SOURCE_DIR_BINUTILS}"
		GIT_REPOSITORY "${REPOSITORY_BINUTILS}"
		GIT_TAG "${REPOSITORY_TAG_BINUTILS}"
		CONFIGURE_COMMAND "${CMAKE_CURRENT_BINARY_DIR}/${SOURCE_DIR_BINUTILS}/configure"
		CONFIGURE_COMMAND "MAKEINFO=true"
		CONFIGURE_COMMAND "CC=${CONFIGURE_CC}"
		CONFIGURE_COMMAND "CFLAGS=${COMPILER_FLAGS} -Wno-absolute-value -Wno-unused-function -Wno-unused-const-variable -Wno-unneeded-internal-declaration -Wno-unused-private-field -Wno-format-security"
		CONFIGURE_COMMAND "LDFLAGS=${LINKER_FLAGS} -static-libstdc++ ${STATIC_LIBGCC}"
		CONFIGURE_COMMAND ${CONFIGURE_ARG_LTO}
		CONFIGURE_COMMAND --prefix=
		CONFIGURE_COMMAND --target=x86_64-nacl
		CONFIGURE_COMMAND ${CONFIGURE_ARG_HOST}
		CONFIGURE_COMMAND --enable-targets=x86_64-nacl,i686-nacl,arm-nacl
		# CONFIGURE_COMMAND --enable-gold --enable-plugins --disable-shared
		CONFIGURE_COMMAND --disable-gold
		CONFIGURE_COMMAND --disable-werror
		CONFIGURE_COMMAND --without-zlib
		CONFIGURE_COMMAND --disable-silent-rules
		CONFIGURE_COMMAND --enable-deterministic-archives
		BUILD_COMMAND $(MAKE) MAKEINFO=true
		INSTALL_COMMAND $(MAKE) MAKEINFO=true
		INSTALL_COMMAND install-strip
		INSTALL_COMMAND DESTDIR=${CMAKE_INSTALL_PREFIX}
	)

	set(binutilsToolNames addr2line;ar;as;c++filt;elfedit;gprof;nm;objcopy;objdump;ranlib;readelf;size;strings;strip;ld;ld.bfd) # ld.gold
	set(binutilsDirectoryToolNames ar;as;ld;ld.bfd;nm;objcopy;objdump;ranlib;strip) # ld.gold

	AddBinaryAliases(binutils "${binutilsToolNames}")
	AddDirectoryAliases(binutils "${binutilsDirectoryToolNames}")
endif()
