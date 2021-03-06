SET(DOC_DIR ${CMAKE_SOURCE_DIR}/../doc)

#
# api documentation
#
find_package(Doxygen)

# build doc always
add_custom_target(doc ALL)

if(DOXYGEN_EXECUTABLE)
  OPTION(DBUS_ENABLE_DOXYGEN_DOCS "build DOXYGEN documentation (requires Doxygen)" ON)
endif(DOXYGEN_EXECUTABLE)

if (DBUS_ENABLE_DOXYGEN_DOCS)
    set (top_srcdir ${CMAKE_SOURCE_DIR}/..)
    configure_file(${CMAKE_SOURCE_DIR}/../Doxyfile.in ${CMAKE_BINARY_DIR}/Doxyfile )
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/api/html/index.html
        COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_BINARY_DIR}/Doxyfile
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
    add_custom_target(apidoc
        COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_BINARY_DIR}/Doxyfile
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
    add_dependencies(doc apidoc)
    add_custom_target(devhelp2
        DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/api/html/index.html ${CMAKE_CURRENT_BINARY_DIR}/dbus.devhelp2
    )
    add_dependencies(doc devhelp2)
endif (DBUS_ENABLE_DOXYGEN_DOCS)

find_program(XSLTPROC_EXECUTABLE xsltproc)
if(XSLTPROC_EXECUTABLE)
    message(STATUS "${XSLTPROC_EXECUTABLE} found")
    set(DBUS_HAVE_XSLTPROC 1)
endif()

if(DBUS_HAVE_XSLTPROC)
    add_custom_command(
        OUTPUT dbus.devhelp2
        COMMAND ${XSLTPROC_EXECUTABLE} -o dbus.devhelp2 ${DOC_DIR}/doxygen_to_devhelp.xsl api/xml/index.xml
        DEPENDS ${DOC_DIR}/doxygen_to_devhelp.xsl api/html/index.html
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
endif()

#
# docbook based documentation
# 
find_package(DocBookXSL)

if(DOCBOOKXSL_DIR AND XSLTPROC_EXECUTABLE)
    option(DBUS_ENABLE_XML_DOCS "build XML documentation" ON)
    set(DBUS_XML_DOCS_ENABLED 1)
    message(STATUS "xsltproc docbook generator found")
    add_custom_target(xmldoc)
    add_dependencies(doc xmldoc)
endif()

if(DBUS_ENABLE_XML_DOCS)

macro(DOCBOOK _sources _format)
    get_filename_component(_infile ${_sources} ABSOLUTE)
    get_filename_component(_name ${_infile} NAME)
    if(${_format} STREQUAL "man")
        string(REPLACE ".xml" "" _outname ${_name})
        set(STYLESHEET "${DOCBOOKXSL_DIR}/manpages/docbook.xsl")
    else()
        string(REPLACE ".xml" ".html" _outname ${_name})
        set(STYLESHEET "${DOCBOOKXSL_DIR}/html/docbook.xsl")
    endif()
    set(_outfile ${CMAKE_CURRENT_BINARY_DIR}/${_outname})

    set(_outfile ${CMAKE_CURRENT_BINARY_DIR}/${_outname})
    if(EXISTS ${_sources})
        if(XSLTPROC_EXECUTABLE)
            add_custom_target(${_outname}
                ${XSLTPROC_EXECUTABLE} --output ${_outfile} --nonet --xinclude  --param passivetex.extensions '1' ${STYLESHEET} ${_infile}
                DEPENDS ${_infile}
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            )
        endif ()
        add_dependencies(xmldoc ${_outname})
        if (${_format} STREQUAL "man")
            install(FILES ${_outfile} DESTINATION share/man/man1)
        else()
            install(FILES ${_outfile} DESTINATION share/doc/dbus)
        endif()
    else()
        message(STATUS "skipping xml doc generating for ${_infile}, file not found")
    endif()
endmacro(DOCBOOK)

### copy tests to builddir so that generated tests and static tests 
### are all in one place.
### todo how to add more filetypes 
MACRO (COPYDIR _src _type)
    FOREACH(FILE_TYPE ${_type})
    	FOREACH(DIR ${_src})
    		FILE(GLOB FILES "${CMAKE_SOURCE_DIR}/../${DIR}/${FILE_TYPE}" )
    		FILE(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/${DIR})
    		FOREACH(FILE ${FILES})
    			GET_FILENAME_COMPONENT(FILENAME ${FILE} NAME)
    			SET (TARGET ${CMAKE_BINARY_DIR}/${DIR}/${FILENAME})
    			configure_file(${FILE} ${TARGET} COPYONLY)
    			IF (CONFIG_VERBOSE)
    				MESSAGE("FROM: ${FILE}\nTO: ${TARGET}\n")
    			ENDIF (CONFIG_VERBOSE)
    		ENDFOREACH(FILE)
    	ENDFOREACH(DIR)
    ENDFOREACH(FILE_TYPE)
ENDMACRO (COPYDIR)

COPYDIR(doc *.png)
COPYDIR(doc *.svg)

docbook(${DOC_DIR}/dbus-test-plan.xml html)
docbook(${DOC_DIR}/dbus-tutorial.xml html)
docbook(${DOC_DIR}/dbus-specification.xml html)
docbook(${DOC_DIR}/dbus-faq.xml html)

configure_file(${DOC_DIR}/dbus-cleanup-sockets.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-cleanup-sockets.1.xml)
configure_file(${DOC_DIR}/dbus-daemon.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-daemon.1.xml)
configure_file(${DOC_DIR}/dbus-launch.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-launch.1.xml)
configure_file(${DOC_DIR}/dbus-monitor.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-monitor.1.xml)
configure_file(${DOC_DIR}/dbus-send.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-send.1.xml)
configure_file(${DOC_DIR}/dbus-test-tool.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-test-tool.1.xml)
configure_file(${DOC_DIR}/dbus-update-activation-environment.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-update-activation-environment.1.xml)
configure_file(${DOC_DIR}/dbus-uuidgen.1.xml.in ${CMAKE_BINARY_DIR}/doc/dbus-uuidgen.1.xml)
if(UNIX)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-daemon.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-monitor.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-send.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-test-tool.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-launch.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-uuidgen.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-cleanup-sockets.1.xml man)
    docbook(${CMAKE_BINARY_DIR}/doc/dbus-update-activation-environment.1.xml man)
endif()
docbook(${CMAKE_BINARY_DIR}/doc/dbus-daemon.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-monitor.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-send.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-test-tool.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-launch.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-uuidgen.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-cleanup-sockets.1.xml html)
docbook(${CMAKE_BINARY_DIR}/doc/dbus-update-activation-environment.1.xml html)

#
# handle html index file
#
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/index.html.cmake ${CMAKE_CURRENT_BINARY_DIR}/index.html )
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/index.html DESTINATION share/doc/dbus)

set (EXTRA_DIST
    ${DOC_DIR}/busconfig.dtd
    ${DOC_DIR}/introspect.dtd
    ${DOC_DIR}/introspect.xsl
)

install(FILES ${EXTRA_DIST} DESTINATION share/doc/dbus)

endif()
