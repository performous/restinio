cmake_minimum_required(VERSION 3.5)

SET(CMAKE_CXX_STANDARD 14)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)

# Determine if RESTinio is built as a subproject (using add_subdirectory)
# or if it is the master project.
SET(RESTINIO_MASTER_PROJECT OFF)
IF (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
	SET(RESTINIO_MASTER_PROJECT ON)
ENDIF ()

option(RESTINIO_PERFORMOUS_HACK "Hacks for building as a Performous dependency." ON)
option(RESTINIO_USE_BOOST_REGEX "Include Boost.Regex support." ${PERFORMOUS_USE_BOOST_REGEX})

option(RESTINIO_INSTALL "Generate the install target." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_TEST "Build the tests." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_SAMPLE "Build samples." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_INSTALL_SAMPLES "Build install samples." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_BENCH "Build the test target." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_INSTALL_BENCHES "Build install samples." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_FIND_DEPS "Use `find_package()` for including RESTinio dependencies." OFF)
option(RESTINIO_FMT_HEADER_ONLY "Include fmt library as header-only." ON)

option(RESTINIO_ALLOW_SOBJECTIZER "Allow add_subdirectory(so_5) in the main RESTinio CMakeLists.txt" ${RESTINIO_MASTER_PROJECT})

# Since v.0.6.5 a user can force usage of already installed http-parser
# library.
option(RESTINIO_USE_EXTERNAL_HTTP_PARSER "Use already installed http-parser library" OFF)

# Since v.0.6.7 a user can force usage of already installed nonstd
# libraries.
option(RESTINIO_USE_EXTERNAL_EXPECTED_LITE
	"Use already installed expected-lite library" OFF)
option(RESTINIO_USE_EXTERNAL_OPTIONAL_LITE
	"Use already installed optional-lite library" OFF)
option(RESTINIO_USE_EXTERNAL_STRING_VIEW_LITE
	"Use already installed string-view-lite library" OFF)
option(RESTINIO_USE_EXTERNAL_VARIANT_LITE
	"Use already installed variant-lite library" OFF)

SET(RESTINIO_USE_BOOST_ASIO "none" CACHE STRING "Use boost version of ASIO")
SET(RESTINIO_USE_BOOST_ASIO_VALUES "none;static;shared")

set_property(CACHE RESTINIO_USE_BOOST_ASIO PROPERTY STRINGS ${RESTINIO_USE_BOOST_ASIO_VALUES})

IF ((RESTINIO_TEST OR RESTINIO_SAMPLE OR RESTINIO_BENCH)
		AND (NOT RESTINIO_ALLOW_SOBJECTIZER))
	message(FATAL_ERROR "RESTINIO_ALLOW_SOBJECTIZER can't be OFF if RESTINIO_TEST or RESTINIO_SAMPLE or RESTINIO_BENCH is used")
ENDIF ()

IF (RESTINIO_MASTER_PROJECT OR RESTINIO_PERFORMOUS_HACK)
	# RESTinio is a main project (not subproject)
	# Do regular stuff here.

	list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")
	include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/link_threads_if_necessary.cmake")

	# ------------------------------------------------------------------------------
	# RESTinio dependencies:
	# ASIO or Boost::ASIO
	if (RESTINIO_PERFORMOUS_HACK)
		set (RESTINIO_USE_BOOST_ASIO "shared" CACHE STRING "Use boost version of ASIO" FORCE)
		set (BOOST_COMPONENTS)
		if (RESTINIO_USE_BOOST_REGEX)
			list (APPEND BOOST_COMPONENTS regex)
			# Try to find boost for boost regex.
		endif ()
	
		if (RESTINIO_USE_BOOST_ASIO MATCHES "none")
			set (RESTINIO_STAND_ALONE_ASIO_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/asio/include)
			set (RESTINIO_STAND_ALONE_ASIO_DEFINES -DASIO_STANDALONE -DASIO_HAS_STD_CHRONO -DASIO_DISABLE_STD_STRING_VIEW)
		else ()
			if (RESTINIO_USE_BOOST_ASIO MATCHES "shared")
				set (Boost_USE_STATIC_LIBS OFF)
			else ()
				set (Boost_USE_STATIC_LIBS ON)
			endif ()
			list (APPEND BOOST_COMPONENTS system)
		endif ()
		list (LENGTH BOOST_COMPONENTS BOOST_COMPONENTS_NUM)
		if (BOOST_COMPONENTS_NUM GREATER_EQUAL 1)
			set (Boost_USE_MULTITHREADED ON)
			find_package (Boost 1.66.0 REQUIRED COMPONENTS ${BOOST_COMPONENTS})
		endif ()
	
		find_package (http_parser 2.8.1 MODULE)
		if (HTTP_PARSER_FOUND)
			add_library (http_parser SHARED IMPORTED GLOBAL)
			set_target_properties (http_parser 
			  PROPERTIES
				IMPORTED_LOCATION ${HTTP_PARSER_LIBRARIES}
				INTERFACE_INCLUDE_DIRECTORIES ${HTTP_PARSER_INCLUDE_DIRS}
			)
		else ()
			message (STATUS "Could not find http_parser; will use embedded version.")
			add_subdirectory (3rdparty-deps/http-parser)
		endif ()
		add_library (unofficial::http_parser::http_parser ALIAS http_parser)
		find_package (fmt 6.1.0 CONFIG QUIET)
		if (NOT fmt_FOUND)
			message (STATUS "libfmt not found or installed version not suitable; using embedded version.")
			add_subdirectory (3rdparty-deps/fmt)
		endif ()
	endif (RESTINIO_PERFORMOUS_HACK)
	
IF (RESTINIO_MASTER_PROJECT)
	IF ( RESTINIO_USE_BOOST_ASIO MATCHES "none" )
		SET( RESTINIO_STAND_ALONE_ASIO_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/asio/include )
		SET( RESTINIO_STAND_ALONE_ASIO_DEFINES -DASIO_STANDALONE -DASIO_HAS_STD_CHRONO -DASIO_DISABLE_STD_STRING_VIEW)

		# Try to find boost for boost regex.
		SET(Boost_USE_MULTITHREADED ON)
		find_package(Boost COMPONENTS regex)
		if ( Boost_FOUND )
			message(STATUS "Found Boost for regex.")
		endif ()
	ELSE ()
		SET(Boost_USE_MULTITHREADED ON)
		IF ( RESTINIO_USE_BOOST_ASIO MATCHES "shared" )
			SET(Boost_USE_STATIC_LIBS OFF)
		ELSE ()
			SET(Boost_USE_STATIC_LIBS ON)
		ENDIF ()

		find_package(Boost 1.66.0 REQUIRED COMPONENTS system regex)
	ENDIF ()

	IF (RESTINIO_FIND_DEPS)
		# Require necessary packages.

		# Catch2
		IF (RESTINIO_TEST)
			find_package(Catch2 CONFIG REQUIRED)
		ENDIF ()

		# HTTP parser
		IF (RESTINIO_USE_EXTERNAL_HTTP_PARSER)
			find_package(http-parser REQUIRED)
		ELSE ()
			find_package(unofficial-http-parser CONFIG REQUIRED)
		ENDIF ()

		# fmtlib
		find_package(fmt REQUIRED)
	ELSE ()
		# Add necessary dependecies.

		include_directories("${CMAKE_SOURCE_DIR}/clara")

		# HTTP parser
		IF (RESTINIO_USE_EXTERNAL_HTTP_PARSER)
			find_package(http-parser REQUIRED)
		ELSE ()
			add_subdirectory(nodejs/http_parser)
		ENDIF ()

		# fmtlib
		add_subdirectory(fmt)
	ENDIF ()

	# If RESTINIO_USE_EXTERNAL_*_LITE is defined we should
	# search that libs regardless of RESTINIO_FIND_DEPS option.
	IF (RESTINIO_USE_EXTERNAL_EXPECTED_LITE)
		find_package(expected-lite REQUIRED)
	ENDIF()
	IF (RESTINIO_USE_EXTERNAL_OPTIONAL_LITE)
		find_package(optional-lite REQUIRED)
	ENDIF()
	IF (RESTINIO_USE_EXTERNAL_STRING_VIEW_LITE)
		find_package(string-view-lite REQUIRED)
	ENDIF()
	IF (RESTINIO_USE_EXTERNAL_VARIANT_LITE)
		find_package(variant-lite REQUIRED)
	ENDIF()

	# ------------------------------------------------------------------------------

	# ------------------------------------------------------------------------------
	# Tests, samples and benchmark dependencies:
	# OpenSSL
	find_package(OpenSSL)
	IF ( OPENSSL_FOUND )
		message(STATUS "OpenSSL include dir: ${OPENSSL_INCLUDE_DIR}")
		message(STATUS "OpenSSL libraries: ${OPENSSL_LIBRARIES}")
	ENDIF ( OPENSSL_FOUND )

	# PCRE
	find_package(PCRE)
	IF (PCRE_FOUND)
		message( STATUS "PCRE_LIBRARIES='" ${PCRE_LIBRARIES} "'" )
		message( STATUS "PCRE_INCLUDE_DIRS='" ${PCRE_INCLUDE_DIRS} "'" )
	ENDIF ()

	# PCRE2
	find_package(PCRE2)
	IF (PCRE2_FOUND)
		message( STATUS "PCRE2_LIBRARIES='" ${PCRE2_LIBRARIES} "'" )
		message( STATUS "PCRE2_INCLUDE_DIRS='" ${PCRE2_INCLUDE_DIRS} "'" )
	ENDIF ()

	# PCRE2
	find_package(PCRE2)
	IF (PCRE2_FOUND)
		message( STATUS "PCRE2_LIBRARIES='" ${PCRE2_LIBRARIES} "'" )
		message( STATUS "PCRE2_INCLUDE_DIRS='" ${PCRE2_INCLUDE_DIRS} "'" )
	ENDIF ()

	# ------------------------------------------------------------------------------
	# SObjectizer
	IF (RESTINIO_ALLOW_SOBJECTIZER)
		SET(SOBJECTIZER_BUILD_STATIC ON)
		SET(SOBJECTIZER_LIBS sobjectizer::StaticLib)
		add_subdirectory(so_5)
	ENDIF ()

	# ------------------------------------------------------------------------------
	# Zlib
	find_package(ZLIB)

	IF (NOT ZLIB_FOUND)
		add_subdirectory(restinio/third_party/zlib)
		message( STATUS "USE OWN ZLIB SOURCES")
		SET(ZLIB_LIBRARIES zlibstatic)
		SET(ZLIB_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/restinio/third_party/zlib)
	ENDIF ()

	message( STATUS "ZLIB_LIBRARIES='" ${ZLIB_LIBRARIES} "'" )
	message( STATUS "ZLIB_INCLUDE_DIRS='" ${ZLIB_INCLUDE_DIRS} "'" )
ENDIF (RESTINIO_MASTER_PROJECT)

ENDIF (RESTINIO_MASTER_PROJECT OR RESTINIO_PERFORMOUS_HACK)

# RESTinio itself.
add_subdirectory(restinio)

# ------------------------------------------------------------------------------
# Tests
IF (RESTINIO_TEST)
	enable_testing()
	add_subdirectory(test)

	IF (WIN32)
		configure_file(${CMAKE_SOURCE_DIR}/cmake/run_tests.bat ${CMAKE_BINARY_DIR} NEWLINE_STYLE WIN32)
	ENDIF ()
ENDIF ()

# ------------------------------------------------------------------------------
# Samples
IF (RESTINIO_SAMPLE)
	add_subdirectory(sample)
ENDIF ()

# ------------------------------------------------------------------------------
# Benches
IF (RESTINIO_BENCH)
	add_subdirectory(benches)
ENDIF ()
