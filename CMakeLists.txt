cmake_minimum_required(VERSION 3.10)

project(midifile)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(CheckIncludeFiles)
include(GNUInstallDirs)

include_directories(include)

check_include_files(unistd.h HAVE_UNISTD_H)
check_include_files(humdrum.h HAVE_HUMDRUM_H)
check_include_files(sys/io.h HAVE_SYS_IO_H)

option(BUILD_MIDILIBRARY_ONLY "Build only the midifile library" OFF)

##############################
##
## Operating-system specific settings:
##

if(APPLE)
  set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9" CACHE STRING "Minimum OS X deployment version")
endif()

if(MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)

  option(STATIC_CRT "Use static CRT libraries" OFF)

  # Rewrite command line flags to use /MT if necessary
  if(STATIC_CRT)
    foreach(flag_var
      CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
      CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
      if(${flag_var} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
      endif(${flag_var} MATCHES "/MD")
    endforeach(flag_var)
  endif()
endif()


##############################
##
## Library:
##

set(SRCS
  src/Options.cpp
  src/Binasc.cpp
  src/MidiEvent.cpp
  src/MidiEventList.cpp
  src/MidiFile.cpp
  src/MidiMessage.cpp
)

set(HDRS
  include/Binasc.h
  include/MidiEvent.h
  include/MidiEventList.h
  include/MidiFile.h
  include/MidiMessage.h
  include/Options.h
)

add_library(midifile STATIC ${SRCS} ${HDRS})

# Modern CMake: Ensure headers are found by other targets linking to midifile
target_include_directories(midifile PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

##############################
##
## Programs:
##

if(NOT BUILD_MIDILIBRARY_ONLY)

  set(TOOL_LIST
    80off asciimidi base642midi binasc chaninfo createmidifile
    createmidifile2 deltatimes drumtab durations extractlyrics
    maxtick mid2hex mid2mat mid2mtb mid2svg midi2base64
    midi2binasc midi2chords midi2melody midi2notes midi2skini
    midi2text midicat mididiss midimean midimixup midirange
    midireg miditime midiuniq mts-type2 mts-type9 perfid
    readstatus redexpress removenote retick shutak smfdur
    stretch sysextest temper text2midi textmidi toascii
    tobin tobinary todec tohex type0 vlv
  )

  foreach(tool ${TOOL_LIST})
    add_executable(${tool} tools/${tool}.cpp)
    target_link_libraries(${tool} PRIVATE midifile)
  endforeach()

endif()

if(HAVE_UNISTD_H AND HAVE_SYS_IO_H)
  add_executable(midi2beep tools/midi2beep.cpp)
  target_link_libraries(midi2beep PRIVATE midifile)
endif()

# The following programs require headers from humextra repository.
# <https://github.com/humdrum-tools/humextra>
if(HAVE_HUMDRUM_H)
  set(HUM_TOOLS henonfile mid2hum midiexcerpt peep2midi)
  foreach(tool ${HUM_TOOLS})
    add_executable(${tool} tools/${tool}.cpp)
    target_link_libraries(${tool} PRIVATE midifile)
  endforeach()
endif()

install(TARGETS midifile
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(FILES ${HDRS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/midifile)