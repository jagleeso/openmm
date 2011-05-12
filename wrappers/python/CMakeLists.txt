find_program(PYTHON_EXECUTABLE NAMES python)

#############################################
### Copy all source files to staging area ###
#############################################

# OPENMM_PYTHON_STAGING_DIR is a staging area for python, swig, and C files in the python package we are making.
set(OPENMM_PYTHON_STAGING_DIR "${CMAKE_BINARY_DIR}/python"
    CACHE PATH "Temporary staging area for Python API wrappers")
mark_as_advanced(OPENMM_PYTHON_STAGING_DIR)

# Create package directory structure
file(MAKE_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}/simtk/openmm)
file(MAKE_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}/simtk/unit)
file(MAKE_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}/src/swig_doxygen/swig_lib/python)

##############################################################################
### Identify files that need to be copied from source area to staging area ###
##############################################################################
set(STAGING_OUTPUT_FILES "") # Will contain all required package files
file(GLOB STAGING_INPUT_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
    "${CMAKE_CURRENT_SOURCE_DIR}/MANIFEST.in"
    "${CMAKE_CURRENT_SOURCE_DIR}/README.txt"
    "${CMAKE_CURRENT_SOURCE_DIR}/*.py"
)

# file(GLOB_RECURSE temp RELATIVE "${CMAKE_SOURCE_DIR}" "${CMAKE_SOURCE_DIR}/src/*.i")
# foreach(f ${temp})
#     set(temp2 "${temp2}\n${f}")
# endforeach()

set(SUBDIRS src simtk)
foreach(SUBDIR ${SUBDIRS})
    file(GLOB_RECURSE STAGING_INPUT_FILES1 RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
        "${CMAKE_CURRENT_SOURCE_DIR}/${SUBDIR}/*README.txt"
        "${CMAKE_CURRENT_SOURCE_DIR}/${SUBDIR}/*.py"
        "${CMAKE_CURRENT_SOURCE_DIR}/${SUBDIR}/*.i"
        "${CMAKE_CURRENT_SOURCE_DIR}/${SUBDIR}/*.sh"
    )
    foreach(file ${STAGING_INPUT_FILES1})
        set(STAGING_INPUT_FILES ${STAGING_INPUT_FILES} "${file}")
    endforeach(file ${STAGING_INPUT_FILES1})
endforeach(SUBDIR ${SUBDIRS})

# message("STAGING_INPUT_FILES = " ${STAGING_INPUT_FILES})
foreach(INIT_FILE ${STAGING_INPUT_FILES})
    set(infile "${CMAKE_CURRENT_SOURCE_DIR}/${INIT_FILE}")
    set(outfile "${OPENMM_PYTHON_STAGING_DIR}/${INIT_FILE}")
    add_custom_command(
        OUTPUT "${outfile}"
        COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${infile}" "${outfile}"
        DEPENDS "${infile}"
        COMMENT "CMake-copying file ${infile} to ${outfile}")
    set(STAGING_OUTPUT_FILES ${STAGING_OUTPUT_FILES} "${outfile}")
    # message("STAGING_OUTPUT_FILE = " ${outfile})
endforeach(INIT_FILE ${STAGING_INPUT_FILES})

########################################################
### Rebuild the Swig wrappers if the API has changed ###
########################################################
# Look in ~/bin/swig first, more likely to be latest version on build machines
find_program(SWIG_EXECUTABLE swig PATHS
    "$ENV{HOME}/bin" "/Users/builder/bin" "/home/builder/bin"
    NO_DEFAULT_PATH)
find_program(SWIG_EXECUTABLE swig)
if(SWIG_EXECUTABLE)
    execute_process(COMMAND ${SWIG_EXECUTABLE} -version
        OUTPUT_VARIABLE SWIG_VERSION_output ERROR_VARIABLE SWIG_VERSION_output)
    string(REGEX REPLACE ".*SWIG Version[^0-9.]*\([0-9.]+\).*" "\\1"
        SWIG_VERSION_output "${SWIG_VERSION_output}")
    set(SWIG_VERSION ${SWIG_VERSION_output} CACHE STRING "Swig version" FORCE)
else(SWIG_EXECUTABLE)
    set(SWIG_VERSION "0.0.0" CACHE STRING "Swig version" FORCE)
endif(SWIG_EXECUTABLE)
# Enforce swig version
string(COMPARE LESS "${SWIG_VERSION}" "1.3.39" SWIG_VERSION_ERROR)
if(SWIG_VERSION_ERROR)
    message("Swig version must be 1.3.39 or greater! (You have ${SWIG_VERSION})")
endif(SWIG_VERSION_ERROR)

find_package(Doxygen REQUIRED)
mark_as_advanced(CLEAR DOXYGEN_EXECUTABLE)

find_package(Java REQUIRED)
mark_as_advanced(CLEAR JAVA_RUNTIME)

# SWIG_OPENMM_DIR is package area where swig files will be created/used
set(SWIG_OPENMM_DIR "${OPENMM_PYTHON_STAGING_DIR}/src/swig_doxygen")
file(MAKE_DIRECTORY ${SWIG_OPENMM_DIR}/swig_lib/python)

### Create OpenMM_headers.xml ###
# Step 1 - Create Doxyfile to point to OpenMM headers
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/src/swig_doxygen/doxygen/Doxyfile.in
    ${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/Doxyfile
    @ONLY
)

# Step 2 - Run doxygen in non-package area
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/index.xml" "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/combine.xslt"
    COMMAND "${DOXYGEN_EXECUTABLE}"
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/Doxyfile"
    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen"
    COMMENT "Parsing OpenMM header files with doxygen..."
)

# Step 3 - Run xslt to create combined xml file
add_custom_command(
    OUTPUT "${SWIG_OPENMM_DIR}/OpenMM_headers.xml"
    COMMAND ${JAVA_RUNTIME}
        -jar "${CMAKE_SOURCE_DIR}/wrappers/saxonb9-1-0-7j/saxon9.jar"
        -t
        -s:"${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/index.xml"
        -xsl:"${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/combine.xslt"
        -o:OpenMM_headers.xml
    DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/index.xml" "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/combine.xslt"
    WORKING_DIRECTORY "${SWIG_OPENMM_DIR}"
    COMMENT "Creating combined OpenMM API xml file..."
)
add_custom_target(CreateCombinedXmlFile DEPENDS
    "${SWIG_OPENMM_DIR}/OpenMM_headers.xml"
    "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/index.xml"
    "${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/xml/combine.xslt"
    ${CMAKE_CURRENT_BINARY_DIR}/src/swig_doxygen/doxygen/Doxyfile)

# OpenMM REFERENCE platform first

# Files which will be created by python script and used by swig (dynamically generated)
set(SWIG_INPUT_FILES_REF
    "${SWIG_OPENMM_DIR}/swig_lib/python/pythonprepend.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/pythonappend.i"
    "${SWIG_OPENMM_DIR}/OpenMM_docstring.i"
    "${SWIG_OPENMM_DIR}/OpenMM_headers.i"
    # "${SWIG_OPENMM_DIR}/OpenMM.i"
)

# Swig input files which should have been copied from source tree
# file(GLOB SWIG_INPUT_FILES2 "${SWIG_OPENMM_DIR}/swig_lib/python/*.i")
set(SWIG_INPUT_FILES2
    "${SWIG_OPENMM_DIR}/swig_lib/python/exceptions.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/extend.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/features.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/header.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/pythoncode.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/typemaps.i"
    "${SWIG_OPENMM_DIR}/swig_lib/python/pythonprepend_all.i"
)

# Create input files for swig
add_custom_command(
    OUTPUT ${SWIG_INPUT_FILES_REF}
    COMMAND ${PYTHON_EXECUTABLE} "${SWIG_OPENMM_DIR}/swigInputBuilder.py"
        -i OpenMM_headers.xml
        -c swigInputConfig.py
        -d OpenMM_docstring.i
        -o OpenMM_headers.i
        -a swig_lib/python/pythonprepend.i
        -z swig_lib/python/pythonappend.i
    WORKING_DIRECTORY "${SWIG_OPENMM_DIR}"
    DEPENDS
        "${SWIG_OPENMM_DIR}/swigInputConfig.py"
        "${SWIG_OPENMM_DIR}/swigInputBuilder.py"
        "${SWIG_OPENMM_DIR}/OpenMM_headers.xml"
    COMMENT "Creating OpenMM Python swig input files..."
)

#~ swig -python -c++ \
#~      -outdir $PYTHON_PACKAGE_DIR \
#~      -o OpenMMSwig.cxx \
#~      OpenMM.i

# Run swig
add_custom_command(
    OUTPUT "${SWIG_OPENMM_DIR}/OpenMMSwig.cxx" "${OPENMM_PYTHON_STAGING_DIR}/simtk/openmm/openmm.py"
    COMMAND ${SWIG_EXECUTABLE}
        -python -c++
        -outdir "${OPENMM_PYTHON_STAGING_DIR}/simtk/openmm"
        -o OpenMMSwig.cxx
        OpenMM.i
    WORKING_DIRECTORY "${SWIG_OPENMM_DIR}"
    DEPENDS
        "${SWIG_OPENMM_DIR}/OpenMM.i"
        ${SWIG_INPUT_FILES_REF}
        ${SWIG_INPUT_FILES2}
    COMMENT "Creating OpenMM Python module sources with swig..."
)
add_custom_target(RunSwig DEPENDS 
    "${SWIG_OPENMM_DIR}/OpenMMSwig.cxx"
    "${OPENMM_PYTHON_STAGING_DIR}/simtk/openmm/openmm.py")

set (STAGING_OUTPUT_FILES ${STAGING_OUTPUT_FILES}
    "${OPENMM_PYTHON_STAGING_DIR}/src/swig_doxygen/OpenMMSwig.cxx"
    "${OPENMM_PYTHON_STAGING_DIR}/simtk/openmm/openmm.py")

##################################################################################################
### Make a list of all folders containing include files the wrappers must be compiled against. ###
##################################################################################################

SET(WRAPPER_BASE_SUBDIRS . openmmapi olla serialization plugins/amoeba/openmmapi plugins/freeEnergy/openmmapi)
SET(WRAPPER_INCLUDE_DIRS) # start empty
FOREACH(subdir ${WRAPPER_BASE_SUBDIRS})
    # append
    SET(WRAPPER_INCLUDE_DIRS ${WRAPPER_INCLUDE_DIRS}
                         ${CMAKE_SOURCE_DIR}/${subdir}/include
                         ${CMAKE_SOURCE_DIR}/${subdir}/include/openmm
                         ${CMAKE_SOURCE_DIR}/${subdir}/include/openmm/internal)
ENDFOREACH(subdir)

###########################################################################
### Run python setup.py indirectly, so we can set environment variables ###
###########################################################################

set(PYTHON_SETUP_COMMAND build)
configure_file(
    pysetup.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/pysetupbuild.cmake"
    @ONLY
)
add_custom_command(
    OUTPUT ${OPENMM_PYTHON_STAGING_DIR}/build
    COMMAND ${CMAKE_COMMAND}
    ARGS -P "${CMAKE_CURRENT_BINARY_DIR}/pysetupbuild.cmake"
    DEPENDS 
        ${OPENMM_PYTHON_STAGING_DIR}/setup.py
        "${CMAKE_CURRENT_BINARY_DIR}/pysetupbuild.cmake"
        ${SHARED_TARGET}
        ${STAGING_OUTPUT_FILES}
    WORKING_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}
    COMMENT "Building OpenMM Python binary module..."
)
add_custom_target(BuildModule ALL DEPENDS ${STAGING_OUTPUT_FILES})

# Binary distribution
if(WIN32)
    set(PYTHON_SETUP_COMMAND bdist_wininst)
else(WIN32)
    set(PYTHON_SETUP_COMMAND bdist)
endif(WIN32)
configure_file(pysetup.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/pysetupbdist.cmake" @ONLY)
add_custom_target(PythonBdist
    COMMAND ${CMAKE_COMMAND}
        -P "${CMAKE_CURRENT_BINARY_DIR}/pysetupbdist.cmake"
    DEPENDS ${STAGING_OUTPUT_FILES} "${CMAKE_CURRENT_BINARY_DIR}/pysetupbdist.cmake"
    WORKING_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}
    COMMENT "Packaging binary distribution package (bdist)..."
)

# Source distribution
set(PYTHON_SETUP_COMMAND sdist)
configure_file(pysetup.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/pysetupsdist.cmake" @ONLY)
add_custom_target(PythonSdist
    COMMAND ${CMAKE_COMMAND}
        -P "${CMAKE_CURRENT_BINARY_DIR}/pysetupsdist.cmake"
    DEPENDS ${STAGING_OUTPUT_FILES} "${CMAKE_CURRENT_BINARY_DIR}/pysetupsdist.cmake"
    WORKING_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}
    COMMENT "Packaging source distribution package (sdist)..."
)

# Install binary module (to system location)
set(PYTHON_SETUP_COMMAND "install")
configure_file(pysetup.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/pysetupinstall.cmake" @ONLY)
add_custom_target(PythonInstall
    COMMAND ${CMAKE_COMMAND}
        -P "${CMAKE_CURRENT_BINARY_DIR}/pysetupinstall.cmake"
    DEPENDS ${STAGING_OUTPUT_FILES} ${OPENMM_PYTHON_STAGING_DIR}/build "${CMAKE_CURRENT_BINARY_DIR}/pysetupinstall.cmake"
    WORKING_DIRECTORY ${OPENMM_PYTHON_STAGING_DIR}
    COMMENT "Installing OpenMM Python binary module..."
)
