OPTIONAL_COMPILE_DIR =
OPTIONAL_DIST_DIR =
OPTIONAL_LIB =

if ENABLE_DEPRECATED
OPTIONAL_COMPILE_DIR += deprecated
OPTIONAL_LIB += deprecated/libdeprecated.la
else
OPTIONAL_DIST_DIR += deprecated
endif

if ENABLE_NSGIF
OPTIONAL_LIB += foreign/libnsgif/libnsgif.la
endif

SUBDIRS = \
	include \
	foreign \
	arithmetic \
	resample \
	colour \
	conversion \
	convolution \
	$(OPTIONAL_COMPILE_DIR) \
	freqfilt \
	histogram \
	draw \
	iofuncs \
	morphology \
	module \
	mosaicing \
	create

lib_LTLIBRARIES = libvips.la

# empty means default to C linking
libvips_la_SOURCES = 

# make sure we link the top-level with c++
nodist_EXTRA_libvips_la_SOURCES = resample/dummy2.cc

# DLLs need dependant libs there too ... put @VIPS_LIBS@ at the end
libvips_la_LIBADD = \
	resample/libresample.la \
	arithmetic/libarithmetic.la \
	colour/libcolour.la \
	conversion/libconversion.la \
	convolution/libconvolution.la \
	$(OPTIONAL_LIB) \
	foreign/libforeign.la \
	freqfilt/libfreqfilt.la \
	histogram/libhistogram.la \
	draw/libdraw.la \
	iofuncs/libiofuncs.la \
	morphology/libmorphology.la \
	mosaicing/libmosaicing.la \
	create/libcreate.la \
	@VIPS_LIBS@

libvips_la_LDFLAGS = \
	-no-undefined \
	-version-info @LIBRARY_CURRENT@:@LIBRARY_REVISION@:@LIBRARY_AGE@ 

EXTRA_DIST = \
	$(OPTIONAL_DIST_DIR)

CLEANFILES = 

# Modules

module_LTLIBRARIES = 

# All modules within the $VIPSHOME/lib/vips-modules-MAJOR.MINOR
# directory are automatically loaded on vips_init.

moduledir = @VIPS_LIBDIR@/vips-modules-@VIPS_MAJOR_VERSION@.@VIPS_MINOR_VERSION@

if MAGICK_MODULE
module_LTLIBRARIES += vips-magick.la
endif # MAGICK_MODULE

if LIBJXL_MODULE
module_LTLIBRARIES += vips-jxl.la
endif # LIBJXL_MODULE

if HEIF_MODULE
module_LTLIBRARIES += vips-heif.la
endif # HEIF_MODULE

if POPPLER_MODULE
module_LTLIBRARIES += vips-poppler.la
endif # POPPLER_MODULE

if OPENSLIDE_MODULE
module_LTLIBRARIES += vips-openslide.la
endif # OPENSLIDE_MODULE

MODULE_CPPFLAGS = \
	-I${top_srcdir}/libvips/include \
	$(REQUIRED_CFLAGS)

MODULE_LDFLAGS = \
	-no-undefined \
	-shared \
	-module \
	-avoid-version

MODULE_LIBADD = \
	libvips.la \
	$(REQUIRED_LIBS)

# Note that only the GObject part should be included in a
# dynamically loadable module. The C definitions are always
# included in the main library.

vips_magick_la_SOURCES = \
	module/magick.c \
	foreign/magick.c \
	foreign/magick.h \
	foreign/magick2vips.c \
	foreign/magick6load.c \
	foreign/magick7load.c \
	foreign/vips2magick.c
vips_magick_la_CPPFLAGS = $(MODULE_CPPFLAGS) $(MAGICK_CFLAGS)
vips_magick_la_LDFLAGS = $(MODULE_LDFLAGS)
vips_magick_la_LIBADD = $(MODULE_LIBADD) $(MAGICK_LIBS)

vips_jxl_la_SOURCES = module/jxl.c foreign/jxlload.c foreign/jxlsave.c
vips_jxl_la_CPPFLAGS = $(MODULE_CPPFLAGS) $(LIBJXL_CFLAGS)
vips_jxl_la_LDFLAGS = $(MODULE_LDFLAGS)
vips_jxl_la_LIBADD = $(MODULE_LIBADD) $(LIBJXL_LIBS)

vips_heif_la_SOURCES = module/heif.c foreign/heifload.c foreign/heifsave.c
vips_heif_la_CPPFLAGS = $(MODULE_CPPFLAGS) $(HEIF_CFLAGS)
vips_heif_la_LDFLAGS = $(MODULE_LDFLAGS)
vips_heif_la_LIBADD = $(MODULE_LIBADD) $(HEIF_LIBS)

vips_poppler_la_SOURCES = module/poppler.c foreign/popplerload.c
vips_poppler_la_CPPFLAGS = $(MODULE_CPPFLAGS) $(POPPLER_CFLAGS)
vips_poppler_la_LDFLAGS = $(MODULE_LDFLAGS)
vips_poppler_la_LIBADD = $(MODULE_LIBADD) $(POPPLER_LIBS)

vips_openslide_la_SOURCES = module/openslide.c foreign/openslideload.c
vips_openslide_la_CPPFLAGS = $(MODULE_CPPFLAGS) $(OPENSLIDE_CFLAGS)
vips_openslide_la_LDFLAGS = $(MODULE_LDFLAGS)
vips_openslide_la_LIBADD = $(MODULE_LIBADD) $(OPENSLIDE_LIBS)

# Introspection

-include $(INTROSPECTION_MAKEFILE)
INTROSPECTION_GIRS =
INTROSPECTION_SCANNER_ARGS = --add-include-path=$(srcdir)
INTROSPECTION_COMPILER_ARGS = --includedir=$(srcdir)

if HAVE_INTROSPECTION

AM_CPPFLAGS = \
	      -I${top_srcdir}/libvips/include \
	      @VIPS_CFLAGS@ \
	      @VIPS_INCLUDES@ \
	      @INTROSPECTION_CFLAGS@ 
AM_LDFLAGS = \
	     @INTROSPECTION_LIBS@ \
	     @LDFLAGS@ 
LDADD = @INTROSPECTION_LIBS@ @VIPS_CFLAGS@ libvips.la @VIPS_LIBS@ 

noinst_PROGRAMS = \
	introspect$(EXEEXT)
introspect_SOURCES = \
	introspect.c

# we can't get the _SOURCES lists from the subdirs directly, we get passed it
# by configure instead
introspection_sources = @vips_introspection_sources@

# we make the vips8 API
Vips-8.0.gir: introspect$(EXEEXT)
Vips_8_0_gir_INCLUDES = GObject-2.0
Vips_8_0_gir_CFLAGS = $(INCLUDES) -I${top_srcdir}/libvips/include
Vips_8_0_gir_LIBS = libvips.la
Vips_8_0_gir_FILES = $(introspection_sources)
INTROSPECTION_GIRS += Vips-8.0.gir

# don't use
#   --warn-all --verbose
# too annoying
Vips_8_0_gir_SCANNERFLAGS = \
	--program=./introspect$(EXEEXT) \
	--identifier-prefix=Vips \
	--identifier-prefix=vips \
	--symbol-prefix=vips \
	--c-include=vips/vips.h

girdir = $(datadir)/gir-1.0
gir_DATA = $(INTROSPECTION_GIRS)

typelibdir = $(libdir)/girepository-1.0
typelib_DATA = $(INTROSPECTION_GIRS:.gir=.typelib)

CLEANFILES += $(gir_DATA) $(typelib_DATA)

endif
