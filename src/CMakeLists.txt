find_package (Threads)
find_package (Bfd)

include(TargetArch)
target_architecture(CMAKE_TARGET_ARCHITECTURES)
message(STATUS "Target architectures: ${CMAKE_TARGET_ARCHITECTURES}")

find_package (ZLIB REQUIRED)
find_package (CURL)

if (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
SET (DL_LIBRARY "")
else (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
find_library (DL_LIBRARY NAMES dl)
if (DL_LIBRARY)
    message (STATUS "Found DL: ${DL_LIBRARY}")
else (DL_LIBRARY)
    message (FATAL_ERROR "Could NOT find DL")
endif (DL_LIBRARY)
endif (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)

if (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
find_library (INTL_LIBRARY NAMES intl)
if (INTL_LIBRARY)
    message (STATUS "Found INTL: ${INTL_LIBRARY}")
else (INTL_LIBRARY)
    message (FATAL_ERROR "Could NOT find INTL")
endif (INTL_LIBRARY)
else (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
SET (INTL_LIBRARY "")
endif (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)

find_library (M_LIBRARY NAMES m)
if (M_LIBRARY)
    message (STATUS "Found M: ${M_LIBRARY}")
else (M_LIBRARY)
    message (FATAL_ERROR "Could NOT find M")
endif (M_LIBRARY)

set (KCOV kcov)


# See http://eatmyrandom.blogspot.se/2010/06/automate-version-numbering-using-git.html
if (EXISTS "${CMAKE_SOURCE_DIR}/.git")
    find_program (GIT_EXECUTABLE NAMES git)
    if (GIT_EXECUTABLE)
        message (STATUS "Found Git: ${GIT_EXECUTABLE}")
    else (GIT_EXECUTABLE)
        message (FATAL_ERROR "Could NOT find Git")
    endif (GIT_EXECUTABLE)

    execute_process (COMMAND "${GIT_EXECUTABLE}"
            "--git-dir=${CMAKE_SOURCE_DIR}/.git"
            describe
            --abbrev=4
            HEAD
        OUTPUT_VARIABLE POD_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
else (EXISTS "${CMAKE_SOURCE_DIR}/.git")
    execute_process (COMMAND head
            -n 1
            "${CMAKE_SOURCE_DIR}/ChangeLog"
        COMMAND cut -d \( -f 2
        COMMAND cut -d \) -f 1
        OUTPUT_VARIABLE POD_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
endif (EXISTS "${CMAKE_SOURCE_DIR}/.git")

add_custom_command(
    OUTPUT version.c
    COMMAND "${CMAKE_COMMAND}"
           -E echo
           "const char *kcov_version = \\\"${POD_VERSION}\\\"\;"
        > version.c
)

set (SOLIB kcov_sowrapper)

set (${SOLIB}_SRCS
	solib-parser/phdr_data.c
	solib-parser/lib.c
	)

set (DISASSEMBLER_SRCS
	parsers/dummy-disassembler.cc
)

set (HAS_LIBBFD "0")

if (CMAKE_TARGET_ARCHITECTURES STREQUAL "i386" OR CMAKE_TARGET_ARCHITECTURES STREQUAL "x86_64")
	if (LIBBFD_FOUND)
		set (HAS_LIBBFD "1")
		set (DISASSEMBLER_SRCS
			parsers/bfd-disassembler.cc
		)
		set (DISASSEMBLER_LIBRARIES
			${LIBBFD_OPCODES_LIBRARY}
			${LIBBFD_BFD_LIBRARY}
			${LIBBFD_IBERTY_LIBRARY}
		)
	endif (LIBBFD_FOUND)
endif (CMAKE_TARGET_ARCHITECTURES STREQUAL "i386" OR CMAKE_TARGET_ARCHITECTURES STREQUAL "x86_64")

set (coveralls_SRCS writers/dummy-coveralls-writer.cc)

if (CURL_FOUND)
	set (coveralls_SRCS writers/coveralls-writer.cc)
endif (CURL_FOUND)

if (KCOV_STATIC_BUILD STREQUAL "1")
	message(STATUS "Building a static binary (no coveralls support)")

	set (CURL_LIBRARIES "")
	# Coveralls doesn't work in a static setting
	set (coveralls_SRCS writers/dummy-coveralls-writer.cc)
	set (CMAKE_EXE_LINKER_FLAGS "-static")
endif (KCOV_STATIC_BUILD STREQUAL "1")

set (ELF_SRCS
	dummy-solib-handler.cc
)
set (MACHO_SRCS
)
set (SOLIB_generated )

# Linux-specific sources
if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
	find_package (LibElf)
	find_package (Elfutils)
	set (ELF_SRCS
		engines/clang-coverage-engine.cc
		engines/ptrace.cc
		engines/ptrace_linux.cc
		engines/kernel-engine.cc
		parsers/elf.cc
		parsers/elf-parser.cc
		parsers/dwarf.cc
		solib-handler.cc
		solib-parser/phdr_data.c
	)
	set (SOLIB_generated library.cc)
	add_library (${SOLIB} SHARED ${${SOLIB}_SRCS})
	set_target_properties(${SOLIB} PROPERTIES SUFFIX ".so")
	target_link_libraries(${SOLIB} dl)
elseif (CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
	find_package (LibElf)
	find_package (Elfutils)
	set (ELF_SRCS
		engines/clang-coverage-engine.cc
		engines/ptrace.cc
		engines/ptrace_freebsd.cc
		parsers/elf.cc
		parsers/elf-parser.cc
		parsers/dwarf.cc
		solib-handler.cc
		solib-parser/phdr_data.c
	)
	set (SOLIB_generated library.cc)
	add_library (${SOLIB} SHARED ${${SOLIB}_SRCS})
	set_target_properties(${SOLIB} PROPERTIES SUFFIX ".so")
else (CMAKE_SYSTEM_NAME STREQUAL "Linux")
	find_library(LLDB_LIBRARY
      NAMES
	    LLDB
	  PATHS
        "/usr/lib"
        "/usr/lib64"
        "/usr/local/lib"
        "/usr/local/lib64"
        "/opt/local/lib"
        "/opt/usr/lib64"
		"/Library/Developer/CommandLineTools/Library/PrivateFrameworks"
		"/Applications/Xcode.app/Contents/SharedFrameworks"
		ENV LIBRARY_PATH
		ENV LD_LIBRARY_PATH
	)
	mark_as_advanced(LLDB_LIBRARY)

	# Not really true, but anyway
	set(LIBELF_FOUND 1)

	set (MACHO_SRCS
		engines/lldb-engine.cc
	)
endif (CMAKE_SYSTEM_NAME STREQUAL "Linux")

set (${KCOV}_SRCS
    capabilities.cc
    collector.cc
    configuration.cc
    engine-factory.cc
    engines/bash-engine.cc
    engines/system-mode-engine.cc
    engines/system-mode-file-format.cc
    engines/python-engine.cc
    filter.cc
    gcov.cc
    main.cc
    merge-file-parser.cc
    output-handler.cc
    ${DISASSEMBLER_SRCS}
    parser-manager.cc
    reporter.cc
    source-file-cache.cc
    utils.cc
    writers/cobertura-writer.cc
    writers/json-writer.cc
    ${coveralls_SRCS}
    writers/html-writer.cc
    writers/sonarqube-xml-writer.cc
    writers/writer-base.cc
    ${ELF_SRCS}
    ${MACHO_SRCS}
    include/capabilities.hh
    include/gcov.hh
    include/reporter.hh
    include/collector.hh
    include/generated-data-base.hh
    include/solib-handler.hh
    include/configuration.hh
    include/lineid.hh
    include/swap-endian.hh
    include/engine.hh
    include/manager.hh
    include/utils.hh
    include/file-parser.hh
    include/output-handler.hh
    include/writer.hh
    include/filter.hh
    include/phdr_data.h
    system-mode/file-data.cc
    )

set (KCOV_SYSTEM_MODE_SRCS
    configuration.cc
	dummy-solib-handler.cc
    engine-factory.cc
    engines/system-mode-file-format.cc
    engines/ptrace.cc
    engines/ptrace_linux.cc
    filter.cc
    gcov.cc
    include/capabilities.hh
    include/collector.hh
    include/configuration.hh
    include/engine.hh
    include/file-parser.hh
    include/filter.hh
    include/gcov.hh
    include/generated-data-base.hh
    include/lineid.hh
    include/manager.hh
    include/output-handler.hh
    include/phdr_data.h
    include/reporter.hh
    include/solib-handler.hh
    include/swap-endian.hh
    include/utils.hh
    include/writer.hh
    main-system-daemon.cc
    parser-manager.cc
	system-mode/file-data.cc
	system-mode/registration.cc
    utils.cc
)

set (KCOV_LIBRARY_PREFIX "/tmp")

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -g -Wall -D_GLIBCXX_USE_NANOSLEEP -DKCOV_LIBRARY_PREFIX=${KCOV_LIBRARY_PREFIX} -DKCOV_HAS_LIBBFD=${HAS_LIBBFD}")

include_directories(
	include/
	../external/lldb/include/
	${ZLIB_INCLUDE_DIR}
	)

if (LIBDW_FOUND)
    include_directories(${LIBDW_INCLUDE_DIRS})
endif (LIBDW_FOUND)

if (LIBELF_FOUND)
    include_directories(${LIBELF_INCLUDE_DIRS})
endif (LIBELF_FOUND)

if (CURL_FOUND)
    include_directories(${CURL_INCLUDE_DIRS})
endif (CURL_FOUND)

add_library (bash_execve_redirector SHARED engines/bash-execve-redirector.c)
set_target_properties(bash_execve_redirector PROPERTIES SUFFIX ".so")
target_link_libraries(bash_execve_redirector ${DL_LIBRARY})

add_library (kcov_system_lib SHARED engines/system-mode-binary-lib.cc utils.cc system-mode/registration.cc)
set_target_properties(kcov_system_lib PROPERTIES SUFFIX ".so")
target_link_libraries(kcov_system_lib
    "${DL_LIBRARY}"
)

add_library (kcov_system_lib_32 SHARED engines/system-mode-binary-lib.cc utils.cc system-mode/registration.cc)
set_target_properties(kcov_system_lib_32 PROPERTIES SUFFIX ".so" COMPILE_FLAGS "-m32" LINK_FLAGS "-m32")
target_link_libraries(kcov_system_lib_32
    "${DL_LIBRARY}"
)


add_custom_command(
    OUTPUT library.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            lib${SOLIB}.so
            __library
        > library.cc
    DEPENDS
        ${SOLIB}
        "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

add_custom_command(
    OUTPUT bash-redirector-library.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            $<TARGET_FILE:bash_execve_redirector>
            bash_redirector_library
        > bash-redirector-library.cc
    DEPENDS
        bash_execve_redirector
        "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

add_custom_command(
    OUTPUT kcov-system-library.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            $<TARGET_FILE:kcov_system_lib>
            kcov_system_library
        > kcov-system-library.cc
    DEPENDS
        kcov_system_lib
       "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

add_custom_command(
    OUTPUT kcov-system-library-32.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            $<TARGET_FILE:kcov_system_lib_32>
            kcov_system_library_32
        > kcov-system-library-32.cc
    DEPENDS
        kcov_system_lib_32
       "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

add_custom_command(
    OUTPUT python-helper.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            "${CMAKE_CURRENT_SOURCE_DIR}/engines/python-helper.py"
            python_helper
        > python-helper.cc
    DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/engines/python-helper.py"
        "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

add_custom_command(
    OUTPUT bash-helper.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            "${CMAKE_CURRENT_SOURCE_DIR}/engines/bash-helper.sh"
            bash_helper
            "${CMAKE_CURRENT_SOURCE_DIR}/engines/bash-helper-debug-trap.sh"
            bash_helper_debug_trap
        > bash-helper.cc
    DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/engines/bash-helper.sh"
        "${CMAKE_CURRENT_SOURCE_DIR}/engines/bash-helper-debug-trap.sh"
        "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

add_custom_command(
    OUTPUT html-data-files.cc
    COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/bcov.css" css_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/amber.png" icon_amber
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/glass.png" icon_glass
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/source-file.html" source_file_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/index.html" index_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/handlebars.js" handlebars_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/kcov.js" kcov_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/jquery.min.js" jquery_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/jquery.tablesorter.min.js" tablesorter_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/jquery.tablesorter.widgets.min.js" tablesorter_widgets_text
            "${CMAKE_CURRENT_SOURCE_DIR}/../data/tablesorter-theme.css" tablesorter_theme_text
        > html-data-files.cc
    DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/bcov.css"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/amber.png"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/glass.png"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/source-file.html"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/index.html"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/handlebars.js"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/kcov.js"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/jquery.min.js"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/jquery.tablesorter.min.js"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/js/jquery.tablesorter.widgets.min.js"
        "${CMAKE_CURRENT_SOURCE_DIR}/../data/tablesorter-theme.css"
        "${CMAKE_CURRENT_SOURCE_DIR}/bin-to-c-source.py"
)

# Reference: http://www.cmake.org/Wiki/CMake_RPATH_handling
if (SPECIFY_RPATH)
	set (CMAKE_SKIP_BUILD_RPATH  FALSE)
	set (CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
	set (CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
	# the RPATH to be used when installing, but only if it's not a system directory
	list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" IS_SYSTEM_DIR)
	if (IS_SYSTEM_DIR STREQUAL "-1")
	    set (CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
	endif (IS_SYSTEM_DIR STREQUAL "-1")
endif (SPECIFY_RPATH)

if (LIBELF_FOUND)
	add_executable (${KCOV} ${${KCOV}_SRCS} ${SOLIB_generated} bash-redirector-library.cc python-helper.cc bash-helper.cc kcov-system-library.cc kcov-system-library-32.cc html-data-files.cc version.c)

	target_link_libraries(${KCOV}
		stdc++
		${CMAKE_THREAD_LIBS_INIT}
		${CURL_LIBRARIES}
		"${DL_LIBRARY}"
		"${INTL_LIBRARY}"
		${DISASSEMBLER_LIBRARIES}
		${LIBDW_LIBRARIES}
		${LIBELF_LIBRARIES}
		${LLDB_LIBRARY}
		"${M_LIBRARY}"
		${ZLIB_LIBRARIES}
	)
	install (TARGETS ${KCOV} DESTINATION "${KCOV_INSTALL_BINDIR}")
endif (LIBELF_FOUND)

if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_executable (kcov-system-daemon ${KCOV_SYSTEM_MODE_SRCS} version.c)

    target_link_libraries(kcov-system-daemon
        stdc++
        ${CMAKE_THREAD_LIBS_INIT}
        "${DL_LIBRARY}"
        "${M_LIBRARY}"
        ${ZLIB_LIBRARIES}
	)
	install (TARGETS kcov-system-daemon DESTINATION "${KCOV_INSTALL_BINDIR}")
endif (CMAKE_SYSTEM_NAME STREQUAL "Linux")
