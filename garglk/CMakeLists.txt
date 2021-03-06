set(SRCS babeldata.c cgblorb.c cgdate.c cgfref.c cggestal.c cgmisc.c cgstream.c
    cgstyle.c cgunicod.c config.c draw.c event.c fontdata.c gi_blorb.c
    gi_dispa.c imgload.c imgscale.c winblank.c window.c wingfx.c wingrid.c
    winmask.c winpair.c wintext.c)

option(WITH_BUNDLED_FONTS "Embed default fonts into Gargoyle." ON)
option(WITH_TTS "Enable text-to-speech support." OFF)

set(GARGLKINI "/etc/garglk.ini" CACHE STRING "Full path for garglk.ini.")
set(GARGLKPRE "" CACHE STRING "Binary prefix.")

if(UNIX)
  set(GUILIB "GTK" CACHE STRING "Select GUILIB. Options: GTK, EFL.")

  find_package(PkgConfig REQUIRED)

  pkg_check_modules(FONTCONFIG REQUIRED fontconfig)
  find_package(Freetype REQUIRED)
  find_package(JPEG REQUIRED)
  find_package(PNG REQUIRED)
  list(APPEND GARGLK_INCLUDES ${FONTCONFIG_INCLUDE_DIRS} ${FREETYPE_INCLUDE_DIRS} ${JPEG_INCLUDE_DIR} ${PNG_INCLUDE_DIRS})
  list(APPEND GARGLK_LIBS ${FONTCONFIG_LIBRARIES} ${FREETYPE_LIBRARIES} ${JPEG_LIBRARIES} ${PNG_LIBRARIES})
elseif(MINGW)
  # On Windows, the only GUILIB currently supported is MinGW, so don't
  # pretend that there are choices available.
  set(GUILIB "MINGW32")
else()
  message(FATAL_ERROR "Unsupported platform (${CMAKE_SYSTEM_NAME}).")
endif()

set(SOUND_SOURCE sndnull.c)
if(WITH_SDL)
  if(UNIX)
    find_package(SDL REQUIRED)
    find_package(SDL_mixer REQUIRED)
    find_package(SDL_sound REQUIRED)
    list(APPEND GARGLK_INCLUDES ${SDL_INCLUDE_DIR} ${SDL_MIXER_INCLUDE_DIRS} ${SDL_SOUND_INCLUDE_DIRS})
    # NOTE: This should be ${SDL_SOUND_LIBRARIES} but that gets clobbered in FindSDL_sound
    list(APPEND GARGLK_LIBS ${SDL_LIBRARY} ${SDL_MIXER_LIBRARIES} ${SDL_SOUND_LIBRARY})
    set(SOUND_SOURCE sndsdl.c)
  elseif(MINGW)
    list(APPEND GARGLK_INCLUDES ../support/sdl ../support/sdl_sound)
    list(APPEND GARGLK_LIBS sdl_sound)
    list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/sdl/SDL.dll)
    list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/sdl/SDL_mixer.dll)
    list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/sdl/smpeg.dll)
    list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/sdl/libvorbisfile-3.dll)
    set(SOUND_SOURCE sndsdl.c)
  endif()
endif()
list(APPEND SRCS ${SOUND_SOURCE})

if(WITH_BUNDLED_FONTS)
  list(APPEND GARGLK_DEFINITIONS PRIVATE BUNDLED_FONTS)
endif()

if(WITH_BABEL)
  list(APPEND GARGLK_LIBS babel)
  list(APPEND GARGLK_INCLUDES PRIVATE ../support/babel)
  list(APPEND GARGLK_DEFINITIONS PRIVATE BABEL_HANDLER)
endif()

if(WITH_TTS)
  if(UNIX)
    pkg_check_modules(SPEECH_DISPATCHER REQUIRED speech-dispatcher)
    list(APPEND GARGLK_INCLUDES ${SPEECH_DISPATCHER_INCLUDE_DIRS})
    list(APPEND GARGLK_LIBS ${SPEECH_DISPATCHER_LIBRARIES})
    list(APPEND SRCS ttsspeechd.c)
  elseif(MINGW)
    list(APPEND SRCS ttswin.c)
  else()
    message(FATAL_ERROR "TTS requested but no implementation is available on this platform (${CMAKE_SYSTEM_NAME}).")
  endif()
else()
  list(APPEND SRCS ttsnull.c)
endif()

if("${GUILIB}" STREQUAL "GTK")
  find_package(GTK2 REQUIRED)
  list(APPEND GARGLK_INCLUDES ${GTK2_INCLUDE_DIRS})
  list(APPEND GARGLK_LIBS ${GTK2_LIBRARIES})
  list(APPEND SRCS sysgtk.c fontfc.c gtk_utils.c)

  set(GARGOYLE_SOURCES launchgtk.c)
  set(GARGOYLE_INCLUDES ${GTK2_INCLUDE_DIRS})
elseif("${GUILIB}" STREQUAL "EFL")
  find_package(Elementary REQUIRED)
  find_package(Evas REQUIRED)
  find_package(Ecore REQUIRED)

  find_package(PkgConfig REQUIRED)
  pkg_check_modules(ECORE_EXTRAS REQUIRED ecore-x ecore-evas)

  list(APPEND GARGLK_INCLUDES ${ELEMENTARY_INCLUDE_DIRS} ${EVAS_INCLUDE_DIRS} ${ECORE_INCLUDE_DIRS} ${ECORE_EXTRAS_INCLUDE_DIRS})
  list(APPEND GARGLK_LIBS ${ELEMENTARY_LIBRARIES} ${EVAS_LIBRARIES} ${ECORE_LIBRARIES} ${ECORE_EXTRAS_LIBRARIES})
  list(APPEND GARGLK_DEFINITIONS PRIVATE EFL_4BPP)
  list(APPEND SRCS sysefl.c fontfc.c)

  set(GARGOYLE_SOURCES launchefl.c)
  set(GARGOYLE_INCLUDES ${ELEMENTARY_INCLUDE_DIRS} ${EVAS_INCLUDE_DIRS} ${ECORE_INCLUDE_DIRS} ${ECORE_EXTRAS_DIRS})
elseif("${GUILIB}" STREQUAL "MINGW32")
  list(APPEND GARGLK_INCLUDES ../support/freetype ../support/libpng ../support/zlib ../support/libjpeg)
  list(APPEND GARGLK_LIBS winmm)

  # These libraries can be found with a combination of
  # include_directories and adding "png14-14", etc. to ${GARGLK_LIBS};
  # however, once winmm is also added, these can no longer be found, as
  # link_directories seems to be ignored. Linking the DLLs in directly
  # is a workaround.
  #link_directories(../support/libpng ../support/freetype ../support/libjpeg)
  list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/freetype/freetype6.dll)
  list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/libpng/libpng14-14.dll)
  list(APPEND GARGLK_LIBS ${CMAKE_CURRENT_SOURCE_DIR}/../support/libjpeg/libjpeg-8.dll)
  list(APPEND SRCS syswin.c fontwin.c icons.rc)

  set(GARGOYLE_SOURCES launchwin.c)
else()
  message(FATAL_ERROR "Invalid GUILIB: ${GUILIB}.")
endif()

# Shared fails on MinGW, and since the DLL isn't really useful installed
# the way it is on Unix, don't worry about it.
if(UNIX)
  add_library(garglk SHARED ${SRCS})
else()
  add_library(garglk STATIC ${SRCS})
endif()
set_property(TARGET garglk PROPERTY SOVERSION 1)
set_property(TARGET garglk PROPERTY C_STANDARD 99)
target_link_libraries(garglk ${GARGLK_LIBS} m)
target_include_directories(garglk PRIVATE ${GARGLK_INCLUDES})
target_compile_definitions(garglk ${GARGLK_DEFINITIONS})
target_compile_definitions(garglk PRIVATE "GARGLKINI=\"${GARGLKINI}\"")

add_executable(gargoyle launcher.c ${GARGOYLE_SOURCES})
set_property(TARGET gargoyle PROPERTY C_STANDARD 99)
target_include_directories(gargoyle PRIVATE ${GARGOYLE_INCLUDES})
target_link_libraries(gargoyle garglk)
target_compile_definitions(gargoyle PRIVATE "GARGLKINI=\"${GARGLKINI}\"" "GARGLKPRE=\"${GARGLKPRE}\"")

add_library(garglkmain STATIC main.c)

install(TARGETS gargoyle DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(TARGETS garglk garglkmain DESTINATION "${CMAKE_INSTALL_LIBDIR}")
install(FILES gi_blorb.h glk.h glkstart.h DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/garglk")
configure_file("garglk.pc.in" "${CMAKE_CURRENT_BINARY_DIR}/garglk.pc" @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/garglk.pc" DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
