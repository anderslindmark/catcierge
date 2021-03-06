#
# Catcierge CMake project
# -----------------------
#
# Copyright (c) Joakim Soderberg 2013-2014
#
#  This file is part of Catcierge.
#
#    Catcierge is free software: you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation, either version 2 of the License, or
#    (at your option) any later version.
#
#    Foobar is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License
#    along with Foobar.  If not, see <http://www.gnu.org/licenses/>.
#
cmake_minimum_required(VERSION 2.6) 
project(catcierge)

set(CATCIERGE_VERSION_MAJOR 0)
set(CATCIERGE_VERSION_MINOR 4)
set(CATCIERGE_VERSION_PATCH 0)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

set(CATCIERGE_VERSION_STR "${CATCIERGE_VERSION_MAJOR}.${CATCIERGE_VERSION_MINOR}.${CATCIERGE_VERSION_PATCH}")

if (UNIX)
	option(CATCIERGE_WITH_MEMCHECK "Run tests with memory check using valgrind" ON)
	option(WITH_RFID "Build support for the serial port communication with RFID readers" ON)
	option(RPI "Build stuff for raspberry parts. Otherwise only the catcierge stuff is built." ON)
	option(CATCIERGE_COVERAGE "(GCC Only! Requires gcov/lcov to be installed). Include target for doing coverage analysis for the test suite. Note that -DCMAKE_BUILD_TYPE=Debug must be set" OFF)
	option(CATCIERGE_COVERALLS "Turn on generating coverage data for http://coveralls.io/. This only works when run on Travis-CI." OFF)
	option(CATCIERGE_COVERALLS_UPLOAD "If CATCIERGE_COVERALLS is set, upload the generated JSON." ON)
	set(RPI_USERLAND "${PROJECT_SOURCE_DIR}/rpi_userland" CACHE STRING "Path to rpi userland tools https://github.com/raspberrypi/userland.git")
endif()

option(WITH_TEST_PROGRAMS "Turns on compilation of tester programs" ON)
option(WITH_UNIT_TESTS "Turn on compilation of unit test" ON)
option(WITH_ZMQ "Compile ZMQ support" OFF)
option(CATCIERGE_GUI_TESTS "Include GUI tests" OFF)

# Turn on coverage if we're running coveralls!
if (CATCIERGE_COVERALLS)
	if (NOT "$ENV{TRAVIS}" STREQUAL "true")
		message(WARNING "CATCIERGE_COVERALLS turned on but not running on Travis-CI. Upload will fail! Set CATCIERGE_COVERALLS_UPLOAD=OFF to turn off this warning")
	endif()

	set(CATCIERGE_COVERAGE ON)
endif()

if (CATCIERGE_GUI_TESTS)
	add_definitions(-DCATCIERGE_GUI_TESTS)
endif()

# Set some nicer output dirs.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)

find_package(Git)

if (GIT_FOUND)
	# TODO: Move this to separate CMake script.
	macro (git_log_format FORMAT_CHARS VAR_NAME)
		execute_process(
			COMMAND ${GIT_EXECUTABLE} log -1 --pretty=format:%${FORMAT_CHARS}
			WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
			OUTPUT_VARIABLE ${VAR_NAME}
			OUTPUT_STRIP_TRAILING_WHITESPACE
		)
	endmacro()

	git_log_format(h GIT_HASH_SHORT)
	git_log_format(H GIT_HASH)
	git_log_format(an GIT_AUTHOR_EMAIL)
	git_log_format(ae GIT_AUTHOR_EMAIL)
	git_log_format(cn GIT_COMMITTER_NAME)
	git_log_format(ce GIT_COMMITTER_EMAIL)
	git_log_format(B GIT_COMMIT_MESSAGE)

	# Branch.
	execute_process(
		COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
		WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
		OUTPUT_VARIABLE GIT_BRANCH
		OUTPUT_STRIP_TRAILING_WHITESPACE
	)

	# Check the git status is something has changed
	# compared to the current git commit hash.
	execute_process(
		COMMAND ${GIT_EXECUTABLE} status -s -uno --porcelain
		WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
		OUTPUT_VARIABLE GIT_STATUS_OUTPUT
		OUTPUT_STRIP_TRAILING_WHITESPACE
	)
	string(LENGTH "${GIT_STATUS_OUTPUT}" GIT_STATUS)
	if (${GIT_STATUS} GREATER 0)
		set(GIT_STATUS 1)
	else()
		set(GIT_STATUS 0)
	endif()
endif()

# Coverage only works with GCC for a debug build.
if (CATCIERGE_COVERAGE)
   include(CodeCoverage)
   include(Coveralls)

   # This adds coverage arguments to gcc/clang.
   coveralls_turn_on_coverage()
endif()

if (WIN32)
	set(WITH_RFID OFF)
	message("\n!!! Windows currently doesn't support RFID readers !!!\n")

	# Windows couldn't possibly be a raspberry pi...
	set(RPI OFF)
endif()

if (MSVC)
	# Turn off deprecation warnings
	add_definitions(-D_CRT_SECURE_NO_DEPRECATE -D_CRT_NONSTDC_NO_DEPRECATE -D_CRT_SECURE_NO_WARNINGS)
endif()

if (RPI)
	add_definitions(-DRPI)
endif()

#
# System introspection.
#
include(CheckIncludeFiles)
check_include_files(unistd.h CATCIERGE_HAVE_UNISTD_H)
check_include_files(fcntl.h CATCIERGE_HAVE_FCNTL_H)
check_include_files(sys/types.h CATCIERGE_HAVE_SYS_TYPES_H)
check_include_files(sys/stat.h CATCIERGE_HAVE_SYS_STAT_H)
check_include_files(pwd.h CATCIERGE_HAVE_PWD_H)
check_include_files(grp.h CATCIERGE_HAVE_GRP_H)
check_include_files(pty.h CATCIERGE_HAVE_PTY_H)
check_include_files(util.h CATCIERGE_HAVE_UTIL_H)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/catcierge_config.h.in ${CMAKE_CURRENT_BINARY_DIR}/catcierge_config.h)
include_directories(
	${PROJECT_SOURCE_DIR}
	${CMAKE_CURRENT_BINARY_DIR}
	${PROJECT_SOURCE_DIR}/src/sha1)

if (NOT RPI AND NOT OpenCV_DIR)
message("-----------------------------------------------------------------")
message("If CMake cannot find your OpenCV install automatically, you can set:")
message("")
message("-DOpenCV_DIR=<path>")
message("")
message("to point to the <path> where OpenCVConfig.cmake can be found.")
message("This file can be found in the build tree of OpenCV")
message("-----------------------------------------------------------------")
endif()

# Lazy solution for linking against OpenCV on Raspbian.
# Simply hard code the paths to the libs...
#
# This can be overridden by specifying your own OpenCV build
# using -DOpenCV_DIR=<path>
# Where <path> is the directory containing OpenCVConfig.cmake
# 
if (RPI AND NOT OpenCV_DIR)
	# NOTE! The order below is important!
	set(OpenCV_LIBS
		/usr/lib/libopencv_highgui.a
		/usr/lib/libopencv_objdetect.a
		/usr/lib/libopencv_imgproc.a
		/usr/lib/libopencv_core.a
		/usr/lib/libopencv_calib3d.a
		/usr/lib/libopencv_features2d.a
		/usr/lib/libopencv_ml.a
		/usr/lib/libopencv_contrib.a
		/usr/lib/libopencv_flann.a
		#/usr/lib/libopencv_gpu.a
		/usr/lib/libopencv_legacy.a
		/usr/lib/libopencv_video.a
		)

	set(OPENEXR_LIBRARIES
		/usr/lib/libIlmImf.a
		/usr/lib/libIlmImf.so
		/usr/lib/libHalf.so.6
		)

	set(JASPER_LIBRARIES
		/usr/lib/arm-linux-gnueabihf/libjasper.a
		/usr/lib/arm-linux-gnueabihf/libjasper.so
		/usr/lib/arm-linux-gnueabihf/libjasper.so.1
		)

	find_package(ZLIB)
	find_package(JPEG)
	find_package(TIFF)
	find_package(GTK2)
	find_package(PNG)
	find_package(Threads)

	set(LIBS 
		${JASPER_LIBRARIES}
		${OPENEXR_LIBRARIES}
		${GTK2_LIBRARIES}
		${TIFF_LIBRARIES}
		${JPEG_LIBRARIES}
		${ZLIB_LIBRARIES}
		${PNG_LIBRARIES}
		${OpenCV_LIBS}
		stdc++ 
		rt
		${CMAKE_THREAD_LIBS_INIT}
		)
else()
	if (WIN32)
		# We link statically on windows so we don't have to copy DLLs around.
		option(LINK_STATIC "Link eveything static such as OpenCV. Will produce a single exe (big since it contains OpenCV and more). Default on" ON)

		if (LINK_STATIC)
			set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
			set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
			set(OpenCV_STATIC ON)
		endif()
	endif()

	find_package(OpenCV REQUIRED)
	set(LIBS ${OpenCV_LIBS})
endif()

# Raspicam lib.
if (RPI)

	if (NOT EXISTS "${RPI_USERLAND}")
		message(FATAL_ERROR 
			"**************************************************************\n"
			"Missing Raspberry Pi userland sources: ${RPI_USERLAND}.\n"
			"Either point to a separate build using -DRPI_USERLAND=<path>\n"
			"Or run this to get and build it:\n"
			"git submodule update --init\n"
			"./build_userland.sh\n"
			"**************************************************************\n")
	endif()

	include_directories(
		${RPI_USERLAND}/host_applications/linux/libs/bcm_host/include/
		${RPI_USERLAND}/host_applications/linux/apps/raspicam/
		${RPI_USERLAND}/interface/vcos/pthreads/
		${RPI_USERLAND}/interface/vmcs_host/linux/
		${RPI_USERLAND}/interface/mmal/
		${RPI_USERLAND}/interface/khronos/include/
		${RPI_USERLAND}
		${PROJECT_SOURCE_DIR}/src/raspicam_cv
		)

	file(GLOB RASPICAM_SRC ${RPI_USERLAND}/host_applications/linux/apps/raspicam/*.c)
	list(APPEND RASPICAM_SRC ${PROJECT_SOURCE_DIR}/src/raspicam_cv/RaspiCamCV.c)

	add_library(raspicamcv ${RASPICAM_SRC})

	set(RASPICAM_LIBS
		${RPI_USERLAND}/build/lib/libmmal_core.so
		${RPI_USERLAND}/build/lib/libmmal.so
		${RPI_USERLAND}/build/lib/libmmal_util.so
		${RPI_USERLAND}/build/lib/libvcos.so
		${RPI_USERLAND}/build/lib/libbcm_host.so
		m
		)

	target_link_libraries(raspicamcv ${RASPICAM_LIBS})

	# Raspicam test app.
	if (WITH_TEST_PROGRAMS)
		set(RASPICAM_TEST_SRC ${PROJECT_SOURCE_DIR}/src/raspicam_cv/RaspiCamTest.c)
		add_executable(raspicam_test ${RASPICAM_TEST_SRC})
		target_link_libraries(raspicam_test raspicamcv ${LIBS})
	endif()
endif()

if (WITH_ZMQ)
	add_definitions(-DWITH_ZMQ)
	find_package(CZMQ REQUIRED)
	message("CZMQ Library: ${CZMQ_LIBRARIES}")
	message("CZMQ Include: ${CZMQ_INCLUDE_DIRS}")
	include_directories("${CZMQ_INCLUDE_DIRS}")
	list(APPEND LIBS "${CZMQ_LIBRARIES}")
endif()

# Catcierge lib.
set(LIB_SRC
	${PROJECT_SOURCE_DIR}/src/catcierge_strftime.c
	${PROJECT_SOURCE_DIR}/src/catcierge_matcher.c
	${PROJECT_SOURCE_DIR}/src/catcierge_template_matcher.c
	${PROJECT_SOURCE_DIR}/src/catcierge_haar_matcher.c
	${PROJECT_SOURCE_DIR}/src/catcierge_haar_wrapper.cpp
	${PROJECT_SOURCE_DIR}/src/catcierge_util.c
	${PROJECT_SOURCE_DIR}/src/catcierge_log.c
	${PROJECT_SOURCE_DIR}/src/alini/alini.c
	${PROJECT_SOURCE_DIR}/src/sha1/sha1.c
	${PROJECT_SOURCE_DIR}/src/catcierge_args.c
	${PROJECT_SOURCE_DIR}/src/catcierge_timer.c
	${PROJECT_SOURCE_DIR}/src/catcierge_fsm.c
	${PROJECT_SOURCE_DIR}/src/catcierge_output.c)

if (WIN32)
	list(APPEND LIB_SRC ${PROJECT_SOURCE_DIR}/src/win32/gettimeofday.c)
endif()

if (RPI)
	list(APPEND LIB_SRC ${PROJECT_SOURCE_DIR}/src/catcierge_gpio.c)
endif()

if (WITH_RFID)
	add_definitions(-DWITH_RFID)
	list(APPEND LIB_SRC ${PROJECT_SOURCE_DIR}/src/catcierge_rfid.c)
endif()

add_library(catcierge ${LIB_SRC})
target_link_libraries(catcierge ${LIBS})

set(CATCIERGE_PROGRAMS catcierge_grabber2)

if (WITH_TEST_PROGRAMS)
	list(APPEND CATCIERGE_PROGRAMS
		catcierge_tester
		catcierge_fsm_tester)

	# We use some of the unit test helpers in fsm tester.
	list(APPEND catcierge_fsm_tester_SRC
		${PROJECT_SOURCE_DIR}/test/catcierge_test_common.c
		${PROJECT_SOURCE_DIR}/test/catcierge_test_helpers.c)

	if (WITH_RFID)
		list(APPEND CATCIERGE_PROGRAMS catcierge_rfid_tester)
	endif()
endif()

foreach (PROGRAM_NAME ${CATCIERGE_PROGRAMS})
	list(APPEND ${PROGRAM_NAME}_SRC "${PROJECT_SOURCE_DIR}/src/${PROGRAM_NAME}.c")
	message("${PROGRAM_NAME}_SRC = ${${PROGRAM_NAME}_SRC}")

	add_executable(${PROGRAM_NAME} ${${PROGRAM_NAME}_SRC})
	target_link_libraries(${PROGRAM_NAME} catcierge)

	if (RPI)
		target_link_libraries(${PROGRAM_NAME} raspicamcv)
	endif()
endforeach()

################################## Tests ##################################
if (WITH_UNIT_TESTS)
	set(REGRESS_SRC ${LIB_SRC})

	ENABLE_TESTING()
	include_directories(${PROJECT_SOURCE_DIR}/src/)
	add_subdirectory(test)
endif()

message("------------------------------------------------------------------")
message(" Settings:")
message("------------------------------------------------------------------")
message("                                 Version: v${CATCIERGE_VERSION_STR}")
message("          Built for Raspberry pi (-DRPI): ${RPI}")
message("  Raspberry pi userland (-DRPI_USERLAND): ${RPI_USERLAND}")
message("              RFID support (-DWITH_RFID): ${WITH_RFID}")
message("          Run valgrind memcheck on tests ")
message("             (-DCATCIERGE_WITH_MEMCHECK): ${CATCIERGE_WITH_MEMCHECK}")
message("           Compile with coverage support")
message("                  (-DCATCIERGE_COVERAGE): ${CATCIERGE_COVERAGE}")
message("      Generate http://coveralls.io/ json:")
message("                  (-DCATCIERGE_COVERALLS) ${CATCIERGE_COVERALLS}")
message("                 Upload json to coverlls:")
message("           (-DCATCIERGE_COVERALLS_UPLOAD) ${CATCIERGE_COVERALLS_UPLOAD}")
message("   Compile with ZMQ support (-DWITH_ZMQ): ${WITH_ZMQ}")
message("-----------------------------------------------------------------")

if (GIT_STATUS)
	message(WARNING "\nWarning build made from uncommited changes!\n'${GIT_STATUS_OUTPUT}'")
endif()

if (RPI)
message(" If compiling on another platform than raspberry pi, make sure ")
message(" you add -DRPI=OFF to the command line !!!")
message("-----------------------------------------------------------------")
endif()

