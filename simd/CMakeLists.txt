macro(simd_fail message)
  if(REQUIRE_SIMD)
    message(FATAL_ERROR "${message}.")
  else()
    message(WARNING "${message}.  Performance will suffer.")
    set(WITH_SIMD 0 PARENT_SCOPE)
  endif()
endmacro()


###############################################################################
# x86[-64] (NASM)
###############################################################################

if(CPU_TYPE STREQUAL "x86_64" OR CPU_TYPE STREQUAL "i386")

set(CMAKE_ASM_NASM_FLAGS_DEBUG_INIT "-g")
set(CMAKE_ASM_NASM_FLAGS_RELWITHDEBINFO_INIT "-g")

# Allow the location of the NASM executable to be specified using the ASM_NASM
# environment variable.  This should happen automatically, but unfortunately
# enable_language(ASM_NASM) doesn't parse the ASM_NASM environment variable
# until after CMAKE_ASM_NASM_COMPILER has been populated with the results of
# searching for NASM or YASM in the PATH.
if(NOT DEFINED CMAKE_ASM_NASM_COMPILER AND DEFINED ENV{ASM_NASM})
  set(CMAKE_ASM_NASM_COMPILER $ENV{ASM_NASM})
endif()

if(CPU_TYPE STREQUAL "x86_64")
  if(CYGWIN)
    set(CMAKE_ASM_NASM_OBJECT_FORMAT win64)
  endif()
  if(CMAKE_C_COMPILER_ABI MATCHES "ELF X32")
    set(CMAKE_ASM_NASM_OBJECT_FORMAT elfx32)
  endif()
elseif(CPU_TYPE STREQUAL "i386")
  if(BORLAND)
    set(CMAKE_ASM_NASM_OBJECT_FORMAT obj)
  elseif(CYGWIN)
    set(CMAKE_ASM_NASM_OBJECT_FORMAT win32)
  endif()
endif()

if(NOT REQUIRE_SIMD)
  include(CheckLanguage)
  check_language(ASM_NASM)
  if(NOT CMAKE_ASM_NASM_COMPILER)
    simd_fail("SIMD extensions disabled: could not find NASM compiler")
    return()
  endif()
endif()
enable_language(ASM_NASM)
message(STATUS "CMAKE_ASM_NASM_COMPILER = ${CMAKE_ASM_NASM_COMPILER}")

if(CMAKE_ASM_NASM_OBJECT_FORMAT MATCHES "^macho")
  set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DMACHO")
elseif(CMAKE_ASM_NASM_OBJECT_FORMAT MATCHES "^elf")
  set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DELF")
  set(CMAKE_ASM_NASM_DEBUG_FORMAT "dwarf2")
endif()
if(CPU_TYPE STREQUAL "x86_64")
  if(WIN32 OR CYGWIN)
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DWIN64")
  endif()
  set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -D__x86_64__")
elseif(CPU_TYPE STREQUAL "i386")
  if(BORLAND)
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DOBJ32")
  elseif(WIN32 OR CYGWIN)
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DWIN32")
  endif()
endif()

message(STATUS "CMAKE_ASM_NASM_OBJECT_FORMAT = ${CMAKE_ASM_NASM_OBJECT_FORMAT}")

if(NOT CMAKE_ASM_NASM_OBJECT_FORMAT)
  simd_fail("SIMD extensions disabled: could not determine NASM object format")
  return()
endif()

get_filename_component(CMAKE_ASM_NASM_COMPILER_TYPE
  "${CMAKE_ASM_NASM_COMPILER}" NAME_WE)
if(CMAKE_ASM_NASM_COMPILER_TYPE MATCHES "yasm")
  foreach(var CMAKE_ASM_NASM_FLAGS_DEBUG CMAKE_ASM_NASM_FLAGS_RELWITHDEBINFO)
    if(${var} STREQUAL "-g")
      if(CMAKE_ASM_NASM_DEBUG_FORMAT)
        set_property(CACHE ${var} PROPERTY VALUE "-g ${CMAKE_ASM_NASM_DEBUG_FORMAT}")
      else()
        set_property(CACHE ${var} PROPERTY VALUE "")
      endif()
    endif()
  endforeach()
endif()

if(NOT WIN32 AND (CMAKE_POSITION_INDEPENDENT_CODE OR ENABLE_SHARED))
  set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -DPIC")
endif()

string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)
set(EFFECTIVE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} ${CMAKE_ASM_NASM_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "CMAKE_ASM_NASM_FLAGS = ${EFFECTIVE_ASM_NASM_FLAGS}")

set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -I\"${CMAKE_CURRENT_SOURCE_DIR}/nasm/\" -I\"${CMAKE_CURRENT_SOURCE_DIR}/${CPU_TYPE}/\"")

set(GREP grep)
if(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
  set(GREP ggrep)
endif()
add_custom_target(jsimdcfg COMMAND
  ${CMAKE_C_COMPILER} -E -I${CMAKE_BINARY_DIR} -I${CMAKE_CURRENT_BINARY_DIR}
    -I${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/nasm/jsimdcfg.inc.h |
  ${GREP} -E '^[\;%]|^\ %' | sed 's%_cpp_protection_%%' |
  sed 's@% define@%define@g' >${CMAKE_CURRENT_SOURCE_DIR}/nasm/jsimdcfg.inc)

if(CPU_TYPE STREQUAL "x86_64")
  set(SIMD_SOURCES x86_64/jsimdcpu.asm x86_64/jfdctflt-sse.asm
    x86_64/jccolor-sse2.asm x86_64/jcgray-sse2.asm x86_64/jchuff-sse2.asm
    x86_64/jcphuff-sse2.asm x86_64/jcsample-sse2.asm x86_64/jdcolor-sse2.asm
    x86_64/jdmerge-sse2.asm x86_64/jdsample-sse2.asm x86_64/jfdctfst-sse2.asm
    x86_64/jfdctint-sse2.asm x86_64/jidctflt-sse2.asm x86_64/jidctfst-sse2.asm
    x86_64/jidctint-sse2.asm x86_64/jidctred-sse2.asm x86_64/jquantf-sse2.asm
    x86_64/jquanti-sse2.asm
    x86_64/jccolor-avx2.asm x86_64/jcgray-avx2.asm x86_64/jcsample-avx2.asm
    x86_64/jdcolor-avx2.asm x86_64/jdmerge-avx2.asm x86_64/jdsample-avx2.asm
    x86_64/jfdctint-avx2.asm x86_64/jidctint-avx2.asm x86_64/jquanti-avx2.asm)
else()
  set(SIMD_SOURCES i386/jsimdcpu.asm i386/jfdctflt-3dn.asm
    i386/jidctflt-3dn.asm i386/jquant-3dn.asm
    i386/jccolor-mmx.asm i386/jcgray-mmx.asm i386/jcsample-mmx.asm
    i386/jdcolor-mmx.asm i386/jdmerge-mmx.asm i386/jdsample-mmx.asm
    i386/jfdctfst-mmx.asm i386/jfdctint-mmx.asm i386/jidctfst-mmx.asm
    i386/jidctint-mmx.asm i386/jidctred-mmx.asm i386/jquant-mmx.asm
    i386/jfdctflt-sse.asm i386/jidctflt-sse.asm i386/jquant-sse.asm
    i386/jccolor-sse2.asm i386/jcgray-sse2.asm i386/jchuff-sse2.asm
    i386/jcphuff-sse2.asm i386/jcsample-sse2.asm i386/jdcolor-sse2.asm
    i386/jdmerge-sse2.asm i386/jdsample-sse2.asm i386/jfdctfst-sse2.asm
    i386/jfdctint-sse2.asm i386/jidctflt-sse2.asm i386/jidctfst-sse2.asm
    i386/jidctint-sse2.asm i386/jidctred-sse2.asm i386/jquantf-sse2.asm
    i386/jquanti-sse2.asm
    i386/jccolor-avx2.asm i386/jcgray-avx2.asm i386/jcsample-avx2.asm
    i386/jdcolor-avx2.asm i386/jdmerge-avx2.asm i386/jdsample-avx2.asm
    i386/jfdctint-avx2.asm i386/jidctint-avx2.asm i386/jquanti-avx2.asm)
endif()

if(MSVC_IDE)
  set(OBJDIR "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CFG_INTDIR}")
  string(REGEX REPLACE " " ";" CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS}")
elseif(XCODE)
  set(OBJDIR "${CMAKE_CURRENT_BINARY_DIR}")
  string(REGEX REPLACE " " ";" CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS}")
endif()

file(GLOB INC_FILES nasm/*.inc)

foreach(file ${SIMD_SOURCES})
  set(OBJECT_DEPENDS "")
  if(${file} MATCHES jccolor)
    string(REGEX REPLACE "jccolor" "jccolext" DEPFILE ${file})
    set(OBJECT_DEPENDS ${OBJECT_DEPENDS}
      ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE})
  endif()
  if(${file} MATCHES jcgray)
    string(REGEX REPLACE "jcgray" "jcgryext" DEPFILE ${file})
    set(OBJECT_DEPENDS ${OBJECT_DEPENDS}
      ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE})
  endif()
  if(${file} MATCHES jdcolor)
    string(REGEX REPLACE "jdcolor" "jdcolext" DEPFILE ${file})
    set(OBJECT_DEPENDS ${OBJECT_DEPENDS}
      ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE})
  endif()
  if(${file} MATCHES jdmerge)
    string(REGEX REPLACE "jdmerge" "jdmrgext" DEPFILE ${file})
    set(OBJECT_DEPENDS ${OBJECT_DEPENDS}
      ${CMAKE_CURRENT_SOURCE_DIR}/${DEPFILE})
  endif()
  set(OBJECT_DEPENDS ${OBJECT_DEPENDS} ${INC_FILES})
  if(MSVC_IDE OR XCODE)
    # The CMake Visual Studio generators do not work properly with the ASM_NASM
    # language, so we have to go rogue here and use a custom command like we
    # did in prior versions of libjpeg-turbo.  (This is why we can't have nice
    # things.)
    string(REGEX REPLACE "${CPU_TYPE}/" "" filename ${file})
    set(SIMD_OBJ ${OBJDIR}/${filename}${CMAKE_C_OUTPUT_EXTENSION})
    add_custom_command(OUTPUT ${SIMD_OBJ} DEPENDS ${file} ${OBJECT_DEPENDS}
      COMMAND ${CMAKE_ASM_NASM_COMPILER} -f${CMAKE_ASM_NASM_OBJECT_FORMAT}
        ${CMAKE_ASM_NASM_FLAGS} ${CMAKE_CURRENT_SOURCE_DIR}/${file}
        -o${SIMD_OBJ})
    set(SIMD_OBJS ${SIMD_OBJS} ${SIMD_OBJ})
  else()
    set_source_files_properties(${file} PROPERTIES OBJECT_DEPENDS
      "${OBJECT_DEPENDS}")
  endif()
endforeach()

if(MSVC_IDE OR XCODE)
set(SIMD_OBJS ${SIMD_OBJS} PARENT_SCOPE)
  add_library(simd OBJECT ${CPU_TYPE}/jsimd.c)
  add_custom_target(simd-objs DEPENDS ${SIMD_OBJS})
  add_dependencies(simd simd-objs)
else()
  add_library(simd OBJECT ${SIMD_SOURCES} ${CPU_TYPE}/jsimd.c)
endif()
if(NOT WIN32 AND (CMAKE_POSITION_INDEPENDENT_CODE OR ENABLE_SHARED))
  set_target_properties(simd PROPERTIES POSITION_INDEPENDENT_CODE 1)
endif()


###############################################################################
# Arm (Intrinsics or GAS)
###############################################################################

elseif(CPU_TYPE STREQUAL "arm64" OR CPU_TYPE STREQUAL "arm")

include(CheckSymbolExists)
if(BITS EQUAL 32)
  set(CMAKE_REQUIRED_FLAGS -mfpu=neon)
endif()
check_symbol_exists(vld1_s16_x3 arm_neon.h HAVE_VLD1_S16_X3)
check_symbol_exists(vld1_u16_x2 arm_neon.h HAVE_VLD1_U16_X2)
check_symbol_exists(vld1q_u8_x4 arm_neon.h HAVE_VLD1Q_U8_X4)
if(BITS EQUAL 32)
  unset(CMAKE_REQUIRED_FLAGS)
endif()
configure_file(arm/neon-compat.h.in arm/neon-compat.h @ONLY)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/arm)

# GCC (as of this writing) and some older versions of Clang do not have a full
# or optimal set of Neon intrinsics, so for performance reasons, when using
# those compilers, we default to using the older GAS implementation of the Neon
# SIMD extensions for certain algorithms.  The presence or absence of the three
# intrinsics we tested above is a reasonable proxy for this.  We always default
# to using the full Neon intrinsics implementation when building for macOS or
# iOS, to avoid the need for gas-preprocessor.
if((HAVE_VLD1_S16_X3 AND HAVE_VLD1_U16_X2 AND HAVE_VLD1Q_U8_X4) OR APPLE)
  set(DEFAULT_NEON_INTRINSICS 1)
else()
  set(DEFAULT_NEON_INTRINSICS 0)
endif()
option(NEON_INTRINSICS
  "Because GCC (as of this writing) and some older versions of Clang do not have a full or optimal set of Neon intrinsics, for performance reasons, the default when building libjpeg-turbo with those compilers is to continue using the older GAS implementation of the Neon SIMD extensions for certain algorithms.  Setting this option forces the full Neon intrinsics implementation to be used with all compilers.  Unsetting this option forces the hybrid GAS/intrinsics implementation to be used with all compilers."
  ${DEFAULT_NEON_INTRINSICS})
boolean_number(NEON_INTRINSICS PARENT_SCOPE)
if(NEON_INTRINSICS)
  add_definitions(-DNEON_INTRINSICS)
  message(STATUS "Use full Neon SIMD intrinsics implementation (NEON_INTRINSICS = ${NEON_INTRINSICS})")
else()
  message(STATUS "Use partial Neon SIMD intrinsics implementation (NEON_INTRINSICS = ${NEON_INTRINSICS})")
endif()

set(SIMD_SOURCES arm/jcgray-neon.c arm/jcphuff-neon.c arm/jcsample-neon.c
  arm/jdmerge-neon.c arm/jdsample-neon.c arm/jfdctfst-neon.c
  arm/jidctred-neon.c arm/jquanti-neon.c)
if(NEON_INTRINSICS)
  set(SIMD_SOURCES ${SIMD_SOURCES} arm/jccolor-neon.c arm/jidctint-neon.c)
endif()
if(NEON_INTRINSICS OR BITS EQUAL 64)
  set(SIMD_SOURCES ${SIMD_SOURCES} arm/jidctfst-neon.c)
endif()
if(NEON_INTRINSICS OR BITS EQUAL 32)
  set(SIMD_SOURCES ${SIMD_SOURCES} arm/aarch${BITS}/jchuff-neon.c
    arm/jdcolor-neon.c arm/jfdctint-neon.c)
endif()
if(BITS EQUAL 32)
  set_source_files_properties(${SIMD_SOURCES} COMPILE_FLAGS -mfpu=neon)
endif()
if(NOT NEON_INTRINSICS)
  enable_language(ASM)

  set(CMAKE_ASM_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_ASM_FLAGS}")

  string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)
  set(EFFECTIVE_ASM_FLAGS "${CMAKE_ASM_FLAGS} ${CMAKE_ASM_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
  message(STATUS "CMAKE_ASM_FLAGS = ${EFFECTIVE_ASM_FLAGS}")

  set(SIMD_SOURCES ${SIMD_SOURCES} arm/aarch${BITS}/jsimd_neon.S)
endif()

add_library(simd OBJECT ${SIMD_SOURCES} arm/aarch${BITS}/jsimd.c)

if(CMAKE_POSITION_INDEPENDENT_CODE OR ENABLE_SHARED)
  set_target_properties(simd PROPERTIES POSITION_INDEPENDENT_CODE 1)
endif()


###############################################################################
# MIPS (GAS)
###############################################################################

elseif(CPU_TYPE STREQUAL "mips" OR CPU_TYPE STREQUAL "mipsel")

enable_language(ASM)

string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)
set(EFFECTIVE_ASM_FLAGS "${CMAKE_ASM_FLAGS} ${CMAKE_ASM_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "CMAKE_ASM_FLAGS = ${EFFECTIVE_ASM_FLAGS}")

set(CMAKE_REQUIRED_FLAGS -mdspr2)

check_c_source_compiles("
  #if !(defined(__mips__) && __mips_isa_rev >= 2)
  #error MIPS DSPr2 is currently only available on MIPS32r2 platforms.
  #endif
  int main(void) {
    int c = 0, a = 0, b = 0;
    __asm__ __volatile__ (
      \"precr.qb.ph %[c], %[a], %[b]\"
      : [c] \"=r\" (c)
      : [a] \"r\" (a), [b] \"r\" (b)
    );
    return c;
  }" HAVE_DSPR2)

unset(CMAKE_REQUIRED_FLAGS)

if(NOT HAVE_DSPR2)
  simd_fail("SIMD extensions not available for this CPU")
  return()
endif()

add_library(simd OBJECT mips/jsimd_dspr2.S mips/jsimd.c)

if(CMAKE_POSITION_INDEPENDENT_CODE OR ENABLE_SHARED)
  set_target_properties(simd PROPERTIES POSITION_INDEPENDENT_CODE 1)
endif()

###############################################################################
# MIPS64 (Intrinsics)
###############################################################################

elseif(CPU_TYPE STREQUAL "loongson" OR CPU_TYPE MATCHES "^mips64")

set(CMAKE_REQUIRED_FLAGS -Wa,-mloongson-mmi,-mloongson-ext)

check_c_source_compiles("
  int main(void) {
    int c = 0, a = 0, b = 0;
    asm (
      \"paddb %0, %1, %2\"
      : \"=f\" (c)
      : \"f\" (a), \"f\" (b)
    );
    return c;
  }" HAVE_MMI)

unset(CMAKE_REQUIRED_FLAGS)

if(NOT HAVE_MMI)
  simd_fail("SIMD extensions not available for this CPU")
  return()
endif()

set(SIMD_SOURCES mips64/jccolor-mmi.c mips64/jcgray-mmi.c mips64/jcsample-mmi.c
  mips64/jdcolor-mmi.c mips64/jdmerge-mmi.c mips64/jdsample-mmi.c
  mips64/jfdctfst-mmi.c mips64/jfdctint-mmi.c mips64/jidctfst-mmi.c
  mips64/jidctint-mmi.c mips64/jquanti-mmi.c)

if(CMAKE_COMPILER_IS_GNUCC)
  foreach(file ${SIMD_SOURCES})
    set_property(SOURCE ${file} APPEND_STRING PROPERTY COMPILE_FLAGS
      " -fno-strict-aliasing")
  endforeach()
endif()
foreach(file ${SIMD_SOURCES})
  set_property(SOURCE ${file} APPEND_STRING PROPERTY COMPILE_FLAGS
    " -Wa,-mloongson-mmi,-mloongson-ext")
endforeach()

add_library(simd OBJECT ${SIMD_SOURCES} mips64/jsimd.c)

if(CMAKE_POSITION_INDEPENDENT_CODE OR ENABLE_SHARED)
  set_target_properties(simd PROPERTIES POSITION_INDEPENDENT_CODE 1)
endif()

###############################################################################
# PowerPC (Intrinsics)
###############################################################################

elseif(CPU_TYPE STREQUAL "powerpc")

set(CMAKE_REQUIRED_FLAGS -maltivec)

check_c_source_compiles("
  #include <altivec.h>
  int main(void) {
    __vector int vi = { 0, 0, 0, 0 };
    int i[4];
    vec_st(vi, 0, i);
    return i[0];
  }" HAVE_ALTIVEC)

unset(CMAKE_REQUIRED_FLAGS)

if(NOT HAVE_ALTIVEC)
  simd_fail("SIMD extensions not available for this CPU (PowerPC SPE)")
  return()
endif()

set(SIMD_SOURCES powerpc/jccolor-altivec.c powerpc/jcgray-altivec.c
  powerpc/jcsample-altivec.c powerpc/jdcolor-altivec.c
  powerpc/jdmerge-altivec.c powerpc/jdsample-altivec.c
  powerpc/jfdctfst-altivec.c powerpc/jfdctint-altivec.c
  powerpc/jidctfst-altivec.c powerpc/jidctint-altivec.c
  powerpc/jquanti-altivec.c)

set_source_files_properties(${SIMD_SOURCES} PROPERTIES
  COMPILE_FLAGS -maltivec)

add_library(simd OBJECT ${SIMD_SOURCES} powerpc/jsimd.c)

if(CMAKE_POSITION_INDEPENDENT_CODE OR ENABLE_SHARED)
  set_target_properties(simd PROPERTIES POSITION_INDEPENDENT_CODE 1)
endif()


###############################################################################
# None
###############################################################################

else()

simd_fail("SIMD extensions not available for this CPU (${CMAKE_SYSTEM_PROCESSOR})")

endif() # CPU_TYPE
