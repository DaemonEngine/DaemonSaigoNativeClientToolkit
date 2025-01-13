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
option(USE_SYMLINKS "Use symlinks instead of copying binaries" ${CMAKE_HOST_UNIX})

if (USE_LTO)
	set(CONFIGURE_ARG_LTO --enable-lto)
endif()

if (USE_CCACHE)
	set(CCACHE_CMD "ccache")
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

	set(targetPath ${CMAKE_INSTALL_PREFIX}/bin/${targetName})
	set(referenceName ${targetName}-reference)
	set(referencePath ${EXTERNAL_PROJECT_BASE}/tmp/${referenceName})

	add_custom_target(${referenceName}
		ALL
		COMMAND cp -P ${targetPath} ${referencePath}
		DEPENDS ${targetName}-binaries
	)

	add_dependencies(${targetName} ${referenceName})

	foreach(toolName ${toolNames})
		set(aliasName ${REFERENCE_ARCH_NAME}-nacl-${toolName})
		set(aliasPath ${CMAKE_INSTALL_PREFIX}/bin/${aliasName})

		add_custom_target(${aliasName}
			ALL
			COMMAND cp -P ${referencePath} ${aliasPath}
			DEPENDS ${referenceName}
		)

		add_dependencies(${targetName}-renames ${aliasName})

		set(deleteName ${toolName}-delete)
		set(deleteFullPath ${CMAKE_INSTALL_PREFIX}/bin/${toolName})

		add_custom_target(${deleteName}
			ALL
			COMMAND rm -f ${deleteFullPath}
			DEPENDS ${toolFullName}
		)

		add_dependencies(${targetName}-renames ${deleteName})
	endforeach()
endmacro()

macro(AddBinaryAliases targetName toolNames)
	set(targetAliases ${targetName}-aliases)
	add_custom_target(${targetName}-aliases ALL)
	add_dependencies(${targetName} ${targetName}-aliases)
	add_dependencies(${targetName}-aliases ${targetName}-binaries)

	foreach(archName ${ALIAS_ARCH_NAMES})
		foreach(toolName ${toolNames})
			set(referenceName ${REFERENCE_ARCH_NAME}-nacl-${toolName})
			set(aliasName ${archName}-nacl-${toolName})
			set(aliasPath ${CMAKE_INSTALL_PREFIX}/bin/${aliasName})

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

	ExternalProject_Add(clang-binaries
		SOURCE_DIR ${SOURCE_DIR_CLANG}
		GIT_REPOSITORY ${REPOSITORY_CLANG}
		GIT_TAG ${REPOSITORY_TAG_CLANG}
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
		CMAKE_ARGS -DLLVM_DEFAULT_TARGET_TRIPLE=x86_64-nacl
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

	set(clangToolNames clang;clang++)

	RenameBinaryAliases(clang "${clangToolNames}")
	AddBinaryAliases(clang "${clangToolNames}")
endif()

if (BUILD_BINUTILS)
	add_custom_target(binutils ALL)

	ExternalProject_Add(binutils-binaries
		SOURCE_DIR ${SOURCE_DIR_BINUTILS}
		GIT_REPOSITORY ${REPOSITORY_BINUTILS}
		GIT_TAG ${REPOSITORY_TAG_BINUTILS}
		CONFIGURE_COMMAND ${CMAKE_CURRENT_BINARY_DIR}/${SOURCE_DIR_BINUTILS}/configure
		CONFIGURE_COMMAND LD=lld
		CONFIGURE_COMMAND "CC=${CCACHE_CMD} ${CMAKE_C_COMPILER}"
		CONFIGURE_COMMAND "CFLAGS=${COMPILER_FLAGS} -Wno-absolute-value -Wno-unused-function -Wno-unused-const-variable -Wno-unneeded-internal-declaration -Wno-unused-private-field -Wno-format-security"
		# CONFIGURE_COMMAND "LDFLAGS=-fuse-ld=lld -static-libstdc++ -static-libgcc"
		CONFIGURE_COMMAND "LDFLAGS=-static-libstdc++ -static-libgcc"
		CONFIGURE_COMMAND ${CONFIGURE_ARG_LTO}
		CONFIGURE_COMMAND --prefix=
		CONFIGURE_COMMAND --target=x86_64-nacl
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

	set(binutilsToolNames addr2line;ar;as;c++filt;elfedit;gprof;nm;objcopy;objdump;ranlib;readelf;size;strings;strip;ld;ld.bfd) # ld.gold
	set(binutilsDirectoryToolNames ar;as;ld;ld.bfd;nm;objcopy;objdump;ranlib;strip) # ld.gold

	AddBinaryAliases(binutils "${binutilsToolNames}")
	AddDirectoryAliases(binutils "${binutilsDirectoryToolNames}")
endif()
