# Steps
# 1. build freetype2:
#    cmake -GNinja -DCMAKE_INSTALL_PREFIX=InsDir -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_ZLIB=OFF -DWITH_BZip2=OFF -DWITH_PNG=OFF -DWITH_HarfBuzz=OFF srcdir
#    cmake --build $PWD -- install
# 2. build fribidi: https://github.com/wang-bin/fribidi
#    cmake -GNinja -DCMAKE_INSTALL_PREFIX=InsDir -DCMAKE_BUILD_TYPE=RelWithDebInfo srcdir
#    cmake --build $PWD -- install
# 3. build libass
#    cmake -GNinja -DCMAKE_INSTALL_PREFIX=InsDir -DCMAKE_BUILD_TYPE=RelWithDebInfo srcdir
#    cmake --build $PWD -- install

cmake_minimum_required(VERSION 2.8)
if(POLICY CMP0042)
    cmake_policy(SET CMP0042 NEW) # MACOSX_RPATH
endif()

PROJECT(libass)
message(STATUS "TARGET OS: ${CMAKE_SYSTEM_NAME} ${CMAKE_SYSTEM_VERSION} ${CMAKE_SYSTEM_PROCESSOR}")
if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/tools.cmake)
  set(TOOLS_CMAKE ON)
  include(tools.cmake)
endif()
INCLUDE(CheckCCompilerFlag)
INCLUDE(CheckIncludeFile)
INCLUDE(CheckTypeSize)
LIST(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")


SET(VERSION_MAJOR 0)
SET(VERSION_MINOR 13)
SET(VERSION_MICRO 7)
set(PROJECT_VERSION ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_MICRO})
set(SO_VERSION 5) #FIXME: home brew use 9.0.0

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
#list(APPEND CMAKE_FIND_ROOT_PATH ${CMAKE_INSTALL_PREFIX}) # affect yasm probe

# System architecture detection
string(TOLOWER "${CMAKE_SYSTEM_PROCESSOR}" SYSPROC)
set(X86_ALIASES x86 i386 i686 x86_64 amd64)
list(FIND X86_ALIASES "${SYSPROC}" X86MATCH)
if("${SYSPROC}" STREQUAL "" OR X86MATCH GREATER "-1")
    set(X86 1)
    if("${CMAKE_SIZEOF_VOID_P}" MATCHES 8)
        set(X64 1)
        add_definitions(-DX86_64=1)
        message(STATUS "Detected x86_64 target processor")
    else()
        message(STATUS "Detected x86 target processor")
    endif()
endif()

SET(EXTRA_LIBS "")

OPTION(CHECK_ASM "asm" ON)
OPTION(CHECK_CORETEXT "coretext" ON)
OPTION(CHECK_FONTCONFIG "fc" ON)
OPTION(CHECK_DIRECTWRITE "dwrite" ON)
OPTION(CHECK_HARFBUZZ "hb" ON)
OPTION(CHECK_ICONV "iconv" ON)
OPTION(CHECK_LIBPNG "png" "ON")
OPTION(NO_RASTERIZER "" OFF)
OPTION(ENABLE_LARGE_TILES "" OFF)
OPTION(NO_ASM "Disable use of assembly coded primitives (default: auto detect)" OFF)

SET(CONFIG_ASM 0)
IF(CHECK_CORETEXT)
  #FIND_LIBRARY(CORETEXT_LIB CoreText)
  #message(STATUS "coretext: ${CORETEXT_LIB}")
  CHECK_INCLUDE_FILE(CoreText/CoreText.h CONFIG_CORETEXT)
ENDIF()
IF(CHECK_FONTCONFIG)
  SET(CONFIG_FONTCONFIG 0)
ENDIF()
IF(CHECK_DIRECTWRITE)
  CHECK_INCLUDE_FILE(windows.h CONFIG_DIRECTWRITE)
ENDIF()
SET(CONFIG_FREETYPE 1) #required
SET(CONFIG_FRIBIDI 1) #required
IF(CHECK_HARFBUZZ)
  SET(CONFIG_HARFBUZZ 0)
ENDIF()
IF(CHECK_ICONV)
  SET(CONFIG_ICONV 0)
ENDIF()
IF(CHECK_LIBPNG)
  SET(CONFIG_LIBPNG 0)
ENDIF()
IF(NO_RASTERIZER)
  SET(CONFIG_RASTERIZER 0)
ELSE()
  SET(CONFIG_RASTERIZER 1)
ENDIF()
IF(ENABLE_LARGE_TILES)
  SET(CONFIG_LARGE_TILES 1)
ELSE()
  SET(CONFIG_LARGE_TILES 0)
ENDIF()


AUX_SOURCE_DIRECTORY(libass SRC_LIST)
IF(CONFIG_CORETEXT)
  LIST(APPEND EXTRA_LIBS "-framework CoreText -framework CoreFoundation -framework CoreGraphics")
ELSE()
  LIST(REMOVE_ITEM SRC_LIST "libass/ass_coretext.c")
ENDIF()
IF(NOT CONFIG_FONTCONFIG)
  LIST(REMOVE_ITEM SRC_LIST "libass/ass_fontconfig.c")
ENDIF()
IF(NOT CONFIG_DIRECTWRITE)
  LIST(REMOVE_ITEM SRC_LIST "libass/ass_directwrite.c")
ENDIF()

IF(NOT NO_ASM)
  IF(X86)
  enable_language(ASM_YASM)
    find_package(Yasm)
    IF(YASM_FOUND)
      SET(CONFIG_ASM 1)
      message(STATUS "Found Yasm ${YASM_VERSION_STRING} to build assembly primitives")
      SET(ASM_NAMES blend_bitmaps.asm blur.asm cpuid.asm utils.asm)
      IF(X64)
        LIST(APPEND ASM_NAMES be_blur.asm)
      ENDIF()
      IF(CONFIG_RASTERIZER)
        LIST(APPEND ASM_NAMES rasterizer.asm)
      ENDIF()
      foreach(ASM ${ASM_NAMES})
        SET(ASM_SRC "${CMAKE_CURRENT_SOURCE_DIR}/libass/x86/${ASM}")
        list(APPEND ASM_SRCS ${ASM_SRC})
        list(APPEND ASM_OBJS ${ASM}.o)
        add_custom_command(
            OUTPUT ${ASM}.o
            COMMAND ${YASM_EXECUTABLE} ARGS ${YASM_FLAGS} ${ASM_SRC} -o ${ASM}.o
            DEPENDS ${ASM_SRC})
      endforeach()
    ELSE()
      message(STATUS "No yasm found")
    ENDIF()
  ENDIF()
ENDIF()

FILE(READ libass/libass.sym DEF_CONTENT)
FILE(WRITE "${PROJECT_BINARY_DIR}/${PROJECT_NAME}.def" "EXPORTS\n${DEF_CONTENT}")
ADD_LIBRARY(ass SHARED ${SRC_LIST} ${ASM_OBJS} ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.def)
SET_TARGET_PROPERTIES(ass PROPERTIES
  VERSION ${PROJECT_VERSION}
  SOVERSION ${SO_VERSION}
  OUTPUT_NAME "ass"
  CLEAN_DIRECT_OUTPUT 1
  #LINK_SEARCH_START_STATIC 1
)

INCLUDE_DIRECTORIES(
  libass
  ${PROJECT_BINARY_DIR}
  ${CMAKE_INSTALL_PREFIX}/include
  ${CMAKE_INSTALL_PREFIX}/include/fribidi
)
CHECK_INCLUDE_FILE(stdint.h HAVE_STDINT_H)
CHECK_INCLUDE_FILE(inttypes.h HAVE_INTTYPES_H)
IF(HAVE_STDINT_H)
    #  ADD_DEFINITIONS(-DHAVE_STDINT_H=1) #redefine in freetype
ENDIF()
IF(HAVE_INTTYPES_H)
  ADD_DEFINITIONS(-DHAVE_INTTYPES_H=1)
ENDIF()
CHECK_INCLUDE_FILE(dirent.h HAVE_DIRENT_H)
IF(NOT HAVE_DIRENT_H)
  INCLUDE_DIRECTORIES(compat)
ENDIF()

#find_package(fribidi)
find_file(FRIBIDI_CMAKE fribidi.cmake HINTS ${CMAKE_INSTALL_PREFIX}/lib/cmake)
message("FRIBIDI_CMAKE: ${FRIBIDI_CMAKE}")
IF(EXISTS ${FRIBIDI_CMAKE})
  include(${FRIBIDI_CMAKE})

  set(FRIBIDI_LIB fribidi_static)
ENDIF()
LIST(APPEND EXTRA_LIBS "${FRIBIDI_LIB}")

FIND_LIBRARY(FREETYPE_LIBRARIES freetype HINTS "${CMAKE_INSTALL_PREFIX}/lib")
message("FREETYPE_LIBRARIES from CMAKE_INSTALL_PREFIX:${FREETYPE_LIBRARIES}")
if(FREETYPE_LIBRARIES)
  INCLUDE_DIRECTORIES(${CMAKE_INSTALL_PREFIX}/include/freetype2)
else()
  find_package(freetype)
  message(STATUS "freetype: ${FREETYPE_LIBRARIES}")
  INCLUDE_DIRECTORIES(${FREETYPE_INCLUDE_DIRS})
endif()
LIST(APPEND EXTRA_LIBS "${FREETYPE_LIBRARIES}")

IF(MSVC)
  CHECK_C_SOURCE_COMPILES("
    #define snprintf _snprintf
    #include <stdio.h>
    int main() {return 0;}" DEF_SNPRINTF)
  IF(DEF_SNPRINTF)
    ADD_DEFINITIONS(-Dsnprintf=_snprintf) #vc link error. vs2015 redefinition error
  ENDIF()

  #ADD_DEFINITIONS(
    #-Dinline=_inline # defined in ass_compat.h
    #-D_USE_MATH_DEFINES # M_PI. defined in ass_compat.h
  #)
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /subsystem:console")
ELSE()
  ADD_DEFINITIONS(
    -D_USE_MATH_DEFINES # gcc4.9
  )
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -lm -lz") #for android, frexp
ENDIF()
IF(WIN32)
  IF(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -static-libgcc -Wl,-Bstatic") # or LINK_SEARCH_START_STATIC property？
  ENDIF()
ENDIF()

# TODO: winrt toolchain file
IF(WINDOWS_STORE OR WINDOWS_PHONE)
message(STATUS "winrt")
  IF(MSVC14)
    SET(WINVER_HEX "0x0A00")
  ELSE()
    SET(WINVER_HEX "0x0603")
  ENDIF()
  ADD_DEFINITIONS(-D_WIN32_WINNT=${WINVER_HEX})
  IF(WINDOWS_PHONE)
    SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /opt:ref /NODEFAULTLIB:kernel32.lib /NODEFAULTLIB:ole32.lib")
  ENDIF()
  LIST(APPEND EXTRA_LIBS "dwrite")
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /APPCONTAINER")
  SET(CONF "${CONF}\n#define LoadLibraryW(X) LoadPackagedLibrary(X, 0)")
ENDIF()
IF(APPLE)
  LIST(APPEND EXTRA_LIBS "-lbz2")
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -compatibility_version 8 -current_version 8.1")
#TODO: iOS
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mmacosx-version-min=10.6")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mmacosx-version-min=10.6")
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -mmacosx-version-min=10.6")  
ENDIF()

IF(CMAKE_C_COMPILER_ABI MATCHES "ELF")
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--exclude-libs,ALL")
  TARGET_LINK_LIBRARIES(ass LINK_PRIVATE  -Wl,--version-script=${CMAKE_BINARY_DIR}/.ass.ver)
ENDIF()
file(READ libass/libass.sym LIB_SYM)
string(REGEX REPLACE "\n|\n\r" ";" LIB_SYM ${LIB_SYM}) 
file(WRITE ${CMAKE_BINARY_DIR}/.ass.ver "{global:${LIB_SYM}local:*;};")

CHECK_C_COMPILER_FLAG(-std=gnu99 HAVE_STD_GNU99)
IF(HAVE_STD_GNU99)
# gcc4.9: error: 'for' loop initial declarations are only allowed in C99 or C11 mode
# clang with c99 may not support variable length array that used in coretext code. gnu99 is required
  SET(CMAKE_C_FLAGS "-std=gnu99 ${CMAKE_C_FLAGS}")
ENDIF()

LINK_DIRECTORIES("${CMAKE_INSTALL_PREFIX}/lib")
TARGET_LINK_LIBRARIES(ass LINK_PRIVATE "${EXTRA_LIBS}")
file(GLOB HEADERS "libass/ass.h" "libass/ass_types.h")
INSTALL(FILES ${HEADERS} DESTINATION include/ass)
INSTALL(FILES ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.def DESTINATION lib)
INSTALL(TARGETS ass EXPORT ${PROJECT_NAME}
  RUNTIME DESTINATION bin # *.dll
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)
INSTALL(EXPORT ${PROJECT_NAME} DESTINATION lib/cmake)

if(NOT TOOLS_CMAKE)
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
                "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake"
                IMMEDIATE @ONLY)
  add_custom_target(uninstall
                  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake/cmake_uninstall.cmake")
endif()

SET(prefix ${CMAKE_INSTALL_PREFIX})
SET(exec_prefix "\${prefix}")
SET(libdir "\${exec_prefix}/lib")
SET(bindir "\${exec_prefix}/bin")
SET(includedir "\${prefix}/include")
SET(PACKAGE_VERSION "${PROJECT_VERSION}")
CONFIGURE_FILE(
    "${PROJECT_SOURCE_DIR}/${PROJECT_NAME}.pc.in"
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}.pc"
    @ONLY)

FOREACH(VAR CONFIG_ASM CONFIG_CORETEXT CONFIG_FONTCONFIG CONFIG_DIRECTWRITE CONFIG_FREETYPE CONFIG_FRIBIDI CONFIG_HARFBUZZ CONFIG_ICONV CONFIG_LIBPNG CONFIG_RASTERIZER CONFIG_LARGE_TILES)
  IF(${VAR})
    SET(CONF "${CONF}\n#define ${VAR} ${${VAR}}")
  ENDIF()
ENDFOREACH()
# why no #ifndef CONFIG_H?
file(WRITE "${PROJECT_BINARY_DIR}/config.h" "${CONF}")
