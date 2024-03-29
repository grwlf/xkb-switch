CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

PROJECT( XKB-SWITCH )
SET(MAJOR_VERSION 2)
SET(MINOR_VERSION 0)
SET(RELEASE_VERSION 1)
SET(XKBSWITCH_VERSION ${MAJOR_VERSION}.${MINOR_VERSION}.${RELEASE_VERSION})
ADD_DEFINITIONS(-DXKBSWITCH_VERSION="${XKBSWITCH_VERSION}")

# Check presence of development libraries required for build
FIND_PACKAGE(X11 REQUIRED)
if(NOT X11_FOUND)
    MESSAGE(FATAL_ERROR "Not found development files of 'libx11' required for build. (Install libx11-dev or libx11-devel package.) CMake will exit.")
elseif(NOT X11_Xkbfile_FOUND)
    MESSAGE(FATAL_ERROR "Not found development files of 'libxkbfile' required for build. (Install libxkbfile-dev or libxkbfile-devel package.) CMake will exit.")
endif()
INCLUDE_DIRECTORIES(${X11_INCLUDE_DIR})
LINK_DIRECTORIES(${X11_LIBRARY_DIR})

# Compile and link program
OPTION(BUILD_XKBSWITCH_LIB
    "Build a library compatible with vim's libcall interface" ON)
if(BUILD_XKBSWITCH_LIB)
    SET(xkblib xkbswitch)
    ADD_LIBRARY(${xkblib} SHARED src/XKbSwitchApi.cpp src/XKeyboard.cpp)
    SET_TARGET_PROPERTIES(${xkblib} PROPERTIES VERSION ${XKBSWITCH_VERSION} SOVERSION ${MAJOR_VERSION})
    TARGET_LINK_LIBRARIES(${xkblib} X11 xkbfile)
    ADD_EXECUTABLE(xkb-switch src/XKbSwitch.cpp)
    TARGET_LINK_LIBRARIES(xkb-switch ${xkblib})
else()
    ADD_EXECUTABLE(xkb-switch src/XKbSwitch.cpp src/XKeyboard.cpp)
    TARGET_LINK_LIBRARIES(xkb-switch X11 xkbfile)
endif()

# Install program
INSTALL(TARGETS xkb-switch ${xkblib}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib OPTIONAL
)

SET(MAN_COMPRESSION "gzip" CACHE STRING "Manpages compression tool")
SET(MANDIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE STRING "Manpages installation path")

# Function to compress and install man page
# Gets file name and type number
function(install_man man_filename man_type)
    # check what compression tool is available
    FIND_PROGRAM(COMPRESS_EXECUTABLE NAMES ${MAN_COMPRESSION})
    if(NOT COMPRESS_EXECUTABLE)
      SET(MAN_COMPRESSION NO)
    endif()
    # set input an output file names
    SET(raw_man man/${man_filename}.${man_type})
    # compress if there is the compression tool
    if(MAN_COMPRESSION)
      if(COMPRESS_EXECUTABLE MATCHES "gzip")
          SET(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.gz)
        else()
          SET(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.${MAN_COMPRESSION})
        endif()
        ADD_CUSTOM_COMMAND(OUTPUT ${installed_man}
            COMMAND cat ${raw_man} | ${COMPRESS_EXECUTABLE} > ${installed_man}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Compressing man file ${raw_man} to ${installed_man}"
        )
    # elsewise just copy
    else()
        SET(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type})
        MESSAGE(WARNING "There is no compression tool for man pages. Not compressed copy of man file will be used.")
        ADD_CUSTOM_COMMAND(OUTPUT ${installed_man}
            COMMAND cp ${raw_man} ${CMAKE_BINARY_DIR}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Copying man file from ${raw_man}."
        )
    endif()
    # add actions
    ADD_CUSTOM_TARGET(man_${man_filename}_${man_type} ALL DEPENDS ${installed_man})
    INSTALL(FILES ${installed_man}
        DESTINATION ${MANDIR}/man${man_type}
    )
endfunction()

# Compress and install man page
install_man(xkb-switch 1)
