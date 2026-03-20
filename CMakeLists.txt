cmake_minimum_required(VERSION 3.5.0)

project(lofarbeam)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/CMake)

set(CASACORE_MAKE_REQUIRED_EXTERNALS_OPTIONAL TRUE)
find_package(Casacore REQUIRED COMPONENTS casa ms tables measures)
include_directories(${CASACORE_INCLUDE_DIR})

add_compile_options(-std=c++14 -Wall -DNDEBUG -Wl,--no-undefined)

option (BUILD_PYTHON "Build the python bindings" YES)

string(APPEND CMAKE_SHARED_LINKER_FLAGS " -Wl,--no-undefined")
include(GNUInstallDirs)
# The following stuff will set the "rpath" correctly, so that
# LD_LIBRARY_PATH doesn't have to be set.

# use, i.e. don't skip the full RPATH for the build tree
set(CMAKE_SKIP_BUILD_RPATH FALSE)
# when building, don't use the install RPATH already
# (but later on when installing)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
# add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
SET(CMAKE_INSTALL_RPATH "$ORIGIN")


add_library(stationresponse SHARED
  AntennaField.cc
  AntennaFieldHBA.cc
  AntennaFieldLBA.cc
  AntennaModelHBA.cc
  AntennaModelLBA.cc
  DualDipoleAntenna.cc
  ElementResponse.cc
  ITRFConverter.cc
  ITRFDirection.cc
  LofarMetaDataUtil.cc
  MathUtil.cc
  Station.cc
  TileAntenna.cc
  Types.cc)

add_library(elementresponse SHARED
  ElementResponse.cc)

set_target_properties(stationresponse PROPERTIES VERSION 3
  INSTALL_RPATH "$ORIGIN")
set_target_properties(elementresponse PROPERTIES VERSION 3
  INSTALL_RPATH "$ORIGIN")

# The elementresponse lib does not require casacore, so
# linking casacore only to stationresponse is enough.
target_link_libraries(stationresponse PUBLIC ${CASACORE_LIBRARIES})

install (TARGETS stationresponse elementresponse
	COMPONENT libraries
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

# FIXME add COMPONENT for headers
install (FILES
  AntennaField.h
  Constants.h
  LofarMetaDataUtil.h
  Station.h
  Types.h
  ITRFConverter.h
  ITRFDirection.h
  COMPONENT headers
DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/StationResponse)

install (FILES
  ElementResponse.h
  COMPONENT headers
DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/ElementResponse)

find_package(Doxygen)

if(DOXYGEN_FOUND)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
add_custom_target(doc
        ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Generating developer API documentation with Doxygen" VERBATIM)
elseif(DOXYGEN_FOUND)

endif(DOXYGEN_FOUND)

if (BUILD_PYTHON)
	if(CMAKE_VERSION VERSION_LESS "3.18")
		if(DEFINED ENV{CIBUILDWHEEL})
			# Build using `cibuildwheel`
			message(
				FATAL_ERROR
				"Package `cibuildwheel` requires CMake >= 3.18, but you have ${CMAKE_VERSION}."
			)
		endif()
		set(Python3_DEV_COMPONENT Development)
	else()
		# When using cibuildwheel, the "Development" component is not available.
		# Therefore, only depend on "Development.Module" with CMake >= 3.18.
		set(Python3_DEV_COMPONENT Development.Module)
	endif()

	find_package(
		Python3
		COMPONENTS Interpreter ${Python3_DEV_COMPONENT}
		REQUIRED)


  message("Compiling Python ${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR} support. If this is not the intended target use 'DPYTHON_EXECUTABLE' to control.")
  # bind against boost-python for compilation of the python API
  # depending on major release version of the python libraries
  find_package (Casacore REQUIRED COMPONENTS python3)
  set(casapylib ${CASA_PYTHON3_LIBRARY})

  set(BOOST_PYTHON_COMPONENT "python${Python3_VERSION_MAJOR}${Python3_VERSION_MINOR}")
  set(BOOST_PYTHON_LIBRARY "Boost::${BOOST_PYTHON_COMPONENT}")
  find_package(Boost REQUIRED COMPONENTS ${BOOST_PYTHON_COMPONENT})

  add_library(_stationresponse MODULE pystationresponse.cc)
  set_target_properties(_stationresponse PROPERTIES PREFIX "")

  target_link_libraries(_stationresponse PRIVATE stationresponse ${casapylib} Python3::Module ${BOOST_PYTHON_LIBRARY})
  set_target_properties(_stationresponse PROPERTIES INSTALL_RPATH
                                            "$ORIGIN/../../${CMAKE_INSTALL_LIBDIR}")
  if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    target_link_options(_stationresponse PRIVATE "LINKER:--as-needed")
  endif()

  # Install pyeverybeam in site-packages directory
  install(
    TARGETS _stationresponse
    LIBRARY DESTINATION lofar/stationresponse)
  install(
    FILES __init__.py
    DESTINATION lofar/stationresponse)

endif(BUILD_PYTHON)

