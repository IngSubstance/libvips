project('vips', 'c', 'cpp',
    version: '8.13.0',
    meson_version: '>=0.56',
    default_options: [
        # this is what glib uses (one of our required deps), so we use it too
        'c_std=gnu99',
        # turn off asserts etc. in release mode
        'b_ndebug=if-release'
    ]
)

version_parts = meson.project_version().split('.')
version_major = version_parts[0]
version_minor = version_parts[1]
version_patch = version_parts[2]

# rules:
# sources changed: increment revision
# binary interface changed: increment current, reset revision to 0
#   binary interface changes backwards compatible?: increment age
#   binary interface changes not backwards compatible?: reset age to 0
library_current = 57
library_age = 15
library_revision = 0
library_version = '@0@.@1@.@2@'.format(library_current - library_age, library_age, library_revision)

gnome = import('gnome')
pymod = import('python')
pkg = import('pkgconfig')
i18n = import('i18n')

add_project_arguments('-Drestrict=__restrict', language: 'cpp')
add_project_link_arguments(
    '-no-undefined',
    language: 'c',
)

# if we're optimising (eg. release mode) we turn off cast checks and
# g_asserts
if get_option('optimization') in ['2', '3', 's']
    add_project_arguments('-DG_DISABLE_CAST_CHECKS', language : ['cpp', 'c'])
    add_project_arguments('-DG_DISABLE_CHECKS', language : ['cpp', 'c'])
    add_project_arguments('-DG_DISABLE_ASSERT', language : ['cpp', 'c'])
endif

# in debug mode we automatically enable leak checks and fatal warnings
# also true for 'debugoptimized'
if get_option('debug')
    add_project_arguments('-DDEBUG_FATAL', language : ['cpp', 'c'])
    add_project_arguments('-DDEBUG_LEAK', language : ['cpp', 'c'])
endif

host_os = host_machine.system()
cc = meson.get_compiler('c')
cpp = meson.get_compiler('cpp')

glib_dep = dependency('glib-2.0', version: '>=2.40')
gio_dep = dependency('gio-2.0')
gobject_dep = dependency('gobject-2.0')
gmodule_dep = dependency('gmodule-no-export-2.0', required: get_option('modules'))
expat_dep = dependency('expat')
thread_dep = dependency('threads')
m_dep = cc.find_library('m', required: false)

libvips_deps = [
    glib_dep,
    gio_dep,
    gobject_dep,
    gmodule_dep,
    expat_dep,
    thread_dep,
    m_dep,
]

prefix_dir = get_option('prefix')
lib_dir = prefix_dir / get_option('libdir')
if gmodule_dep.found() and gmodule_dep.get_variable(pkgconfig: 'gmodule_supported') == 'true'
    # Disable modules by default when building static libraries
    modules_enabled = get_option('modules').enabled() or get_option('default_library') == 'shared'
elif get_option('modules').enabled()
    error('GModule is not supported on your system, please reconfigure with -Dmodules=disabled')
else
    modules_enabled = false
endif

module_dir = lib_dir / 'vips-modules-@0@.@1@'.format(version_major, version_minor)

cfg_var = configuration_data()
cfg_var.set_quoted('G_LOG_DOMAIN', 'VIPS')

if modules_enabled
    cfg_var.set('ENABLE_MODULES', '1')
endif

# Detect and set symbol visibility
if get_option('default_library') == 'shared' and (host_os == 'windows' or host_os == 'cygwin')
    cfg_var.set('DLL_EXPORT', true)
    if cc.get_id() == 'msvc' or cc.get_id() == 'clang-cl'
        cfg_var.set('_VIPS_PUBLIC', '__declspec(dllexport)')
    elif cc.has_function_attribute('visibility:hidden')
        cfg_var.set('_VIPS_PUBLIC', '__attribute__((visibility("default"))) __declspec(dllexport)')
    endif
elif cc.has_function_attribute('visibility:hidden')
    cfg_var.set('_VIPS_PUBLIC', '__attribute__((visibility("default")))')
endif

# we also need to be able to mix vector and scalar arithmetic
vector_arithmetic_check = '''
typedef float v4f __attribute__((vector_size(4 * sizeof(float)),aligned(16)));
int main(void) {
    v4f f = {1, 2, 3, 4}; f *= 12.0; 
    v4f g = {5, 6, 7, 8}; f = g > 0 ? g : -1 * g;
}
'''

# gcc 7.2 seems to work, but then gets confused by signed constants in templates
signed_constants_check = '''
typedef float v4f __attribute__((vector_size(4 * sizeof(float)),aligned(16)));
template <typename T>
static void
h( v4f B )
{
    v4f f;
    f = -1 * B;
}
'''

if cpp.compiles(vector_arithmetic_check, name: 'Has vector arithmetic', dependencies: m_dep)
    if cpp.compiles(signed_constants_check, name: 'Has signed constants in vector templates', dependencies: m_dep)
        cfg_var.set('HAVE_VECTOR_ARITH', '1')
    endif
endif

func_names = [ 'vsnprintf', '_aligned_malloc', 'posix_memalign', 'memalign', 'cbrt', 'hypot', 'atan2', 'asinh' ]
foreach func_name : func_names
    if cc.has_function(func_name, dependencies: m_dep)
        cfg_var.set('HAVE_' + func_name.to_upper(), '1')
    endif
endforeach

if cc.has_function('pthread_setattr_default_np', args: '-D_GNU_SOURCE', prefix: '#include <pthread.h>', dependencies: thread_dep)
    cfg_var.set('HAVE_PTHREAD_DEFAULT_NP', '1')
endif

# needed by rsvg and others
zlib_dep = dependency('zlib', version: '>=0.4', required: get_option('zlib'))
if zlib_dep.found()
    libvips_deps += zlib_dep
    cfg_var.set('HAVE_ZLIB', '1')
endif

gsf_dep = dependency('libgsf-1', version: '>=1.14.26', required: get_option('gsf'))
if gsf_dep.found()
    libvips_deps += gsf_dep
    cfg_var.set('HAVE_GSF', '1')
    if gsf_dep.version().version_compare('>=1.14.31')
        cfg_var.set('HAVE_GSF_ZIP64', '1')
        cfg_var.set('HAVE_GSF_DEFLATE_LEVEL', '1')
    endif
endif

fftw_dep = dependency('fftw3', required: get_option('fftw'))
if fftw_dep.found()
    libvips_deps += fftw_dep
    cfg_var.set('HAVE_FFTW', '1')
endif

# TODO: simplify this when requiring meson>=0.60.0
magick_dep = dependency(get_option('magick-package'), required: false)
if not magick_dep.found()
    # very old versions called it "ImageMagick"
    magick_dep = dependency('ImageMagick', required: get_option('magick'))
endif
if not get_option('magick').disabled() and magick_dep.found()
    if modules_enabled and not get_option('magick-module').disabled()
        cfg_var.set('MAGICK_MODULE', '1')
    else
        libvips_deps += magick_dep
    endif
    if magick_dep.version().version_compare('>=7.0')
        cfg_var.set('HAVE_MAGICK7', '1')
    else
        # come here for imagemagick6, and graphicsmagick1.x, which also uses 
        # the im6 API
        cfg_var.set('HAVE_MAGICK6', '1')
        if cc.has_member('struct _ImageInfo', 'number_scenes', prefix: '#include <magick/api.h>', dependencies: magick_dep)
            cfg_var.set('HAVE_NUMBER_SCENES', '1')
        endif
        func_names = [ 'InheritException', 'AcquireExceptionInfo', 'SetImageProperty', 'SetImageExtent', 'AcquireImage', 'GetVirtualPixels', 'ResetImageProfileIterator', 'ResetImageAttributeIterator', 'ResetImagePropertyIterator', 'MagickCoreGenesis', 'SetImageOption', 'BlobToStringInfo', 'OptimizePlusImageLayers', 'OptimizeImageTransparency' ]
        foreach func_name : func_names
            if cc.has_function(func_name, prefix: '#include <magick/api.h>', dependencies: magick_dep)
                cfg_var.set('HAVE_' + func_name.to_upper(), '1')
            endif
        endforeach
        if cc.compiles('#include <magick/api.h>\nColorspaceType colorspace = CMYColorspace;', name: 'Has CMYColorspace', dependencies: magick_dep)
            cfg_var.set('HAVE_CMYCOLORSPACE', '1')
        endif
        if cc.compiles('#include <magick/api.h>\nColorspaceType colorspace = HCLpColorspace;', name: 'Has HCLpColorspace', dependencies: magick_dep)
            cfg_var.set('HAVE_HCLPCOLORSPACE', '1')
        endif
        # GetImageMagick() takes two args under GM, three under IM
        if cc.compiles('#include <magick/api.h>\nint main() {(void)GetImageMagick(NULL, 0, NULL);}', name: 'GetImageMagick takes three arguments', dependencies: magick_dep)
            cfg_var.set('HAVE_GETIMAGEMAGICK3', '1')
        endif
    endif

    if 'load' in get_option('magick-features')
        cfg_var.set('ENABLE_MAGICKLOAD', '1')
    endif
    if 'save' in get_option('magick-features')
        cfg_var.set('ENABLE_MAGICKSAVE', '1')
        if cc.has_function('ImportImagePixels', prefix: '#include <magick/api.h>', dependencies: magick_dep)
            cfg_var.set('HAVE_IMPORTIMAGEPIXELS', '1')
        endif
        if cc.has_function('ImagesToBlob', prefix: '#include <magick/api.h>', dependencies: magick_dep)
            cfg_var.set('HAVE_IMAGESTOBLOB', '1')
        endif
    endif
endif

cfitsio_dep = dependency('cfitsio', required: get_option('cfitsio'))
if cfitsio_dep.found()
    libvips_deps += cfitsio_dep
    cfg_var.set('HAVE_CFITSIO', '1')
endif

# quant package we use
quantisation_package = disabler()

imagequant_dep = dependency('imagequant', required: get_option('imagequant'))
if imagequant_dep.found()
    libvips_deps += imagequant_dep
    cfg_var.set('HAVE_IMAGEQUANT', '1')
    quantisation_package = imagequant_dep
endif

# only if libimagequant not found
quantizr_dep = disabler()
if not quantisation_package.found()
    quantizr_dep = dependency('quantizr', required: get_option('quantizr'))
    if quantizr_dep.found()
        libvips_deps += quantizr_dep
        cfg_var.set('HAVE_QUANTIZR', '1')
        quantisation_package = quantizr_dep
    endif
endif

cgif_dep = disabler()
if quantisation_package.found()
    cgif_dep = dependency('cgif', version: '>=0.2.0', required: get_option('cgif'))
    if cgif_dep.found()
        libvips_deps += cgif_dep
        cfg_var.set('HAVE_CGIF', '1')
        if cc.compiles('#include <cgif.h>\nint i = CGIF_ATTR_NO_LOOP;', name: 'Has CGIF_ATTR_NO_LOOP', dependencies: cgif_dep)
            cfg_var.set('HAVE_CGIF_ATTR_NO_LOOP', '1')
        endif
    endif
endif

libexif_dep = dependency('libexif', version: '>=0.6', required: get_option('exif'))
if libexif_dep.found()
    libvips_deps += libexif_dep
    cfg_var.set('HAVE_EXIF', '1')
    # some libexif packages need include <libexif/poop.h>, some just <poop.h>
    # how annoying
    if cc.has_header('exif-data.h', dependencies: libexif_dep)
        # libexif includes don't need libexif prefix
        cfg_var.set('UNTAGGED_EXIF', '1')
    endif
endif


libjpeg_dep = dependency('libjpeg', required: get_option('jpeg'))
if libjpeg_dep.found()
    libvips_deps += libjpeg_dep
    cfg_var.set('HAVE_JPEG', '1')
    # features like trellis quant are exposed as extension parameters ...
    # mozjpeg 3.2 and later have #define JPEG_C_PARAM_SUPPORTED, but we must 
    # work with earlier versions
    if cc.has_function('jpeg_c_bool_param_supported', prefix: '#include <stdio.h>\n#include <jpeglib.h>', dependencies: libjpeg_dep)
        cfg_var.set('HAVE_JPEG_EXT_PARAMS', '1')
    endif
endif

# Look for libspng first
# - it's sometimes called "spng.pc", sometimes "libspng.pc", we must search for
#   both
# - we need 0.7+ for PNG write support
# TODO: simplify this when requiring meson>=0.60.0
spng_dep = dependency('spng', version: '>=0.7', required: false)
if not spng_dep.found()
    spng_dep = dependency('libspng', version: '>=0.7', required: get_option('spng'))
endif
if not get_option('spng').disabled() and spng_dep.found()
    libvips_deps += spng_dep
    cfg_var.set('HAVE_SPNG', '1')
endif

# only if libspng not found
png_dep = disabler()
if not spng_dep.found()
    png_dep = dependency('libpng', version: '>=1.2.9', required: get_option('png'))
    if png_dep.found()
        libvips_deps += png_dep
        cfg_var.set('HAVE_PNG', '1')
        if cc.has_function('png_set_chunk_malloc_max', prefix: '#include <png.h>', dependencies: png_dep)
            cfg_var.set('HAVE_PNG_SET_CHUNK_MALLOC_MAX', '1')
        endif
    endif
endif


# libwebp ... target 0.6+ to reduce complication
# webp has the stuff for handling metadata in two separate libraries -- we
# insist on having both of them
libwebp_dep = dependency('libwebp', version: '>=0.6', required: get_option('webp'))
if libwebp_dep.found()
    libvips_deps += libwebp_dep
    libvips_deps += dependency('libwebpmux', version: '>=0.6')
    libvips_deps += dependency('libwebpdemux', version: '>=0.6')
    cfg_var.set('HAVE_LIBWEBP', '1')
endif

pangocairo_dep = dependency('pangocairo', required: get_option('pangocairo'))
if pangocairo_dep.found()
    libvips_deps += pangocairo_dep
    cfg_var.set('HAVE_PANGOCAIRO', '1')
endif

fontconfig_dep = dependency('fontconfig', required: get_option('fontconfig'))
if fontconfig_dep.found() and pangocairo_dep.found()
    libvips_deps += fontconfig_dep
    cfg_var.set('HAVE_FONTCONFIG', '1')
endif

libtiff_dep = dependency('libtiff-4', required: get_option('tiff'))
if libtiff_dep.found()
    libvips_deps += libtiff_dep
    cfg_var.set('HAVE_TIFF', '1')
    # ZSTD and WEBP in TIFF added in libtiff 4.0.10
    if cc.get_define('COMPRESSION_WEBP', prefix: '#include <tiff.h>', dependencies: libtiff_dep) != ''
        cfg_var.set('HAVE_TIFF_COMPRESSION_WEBP', '1')
    endif
endif

# 2.40.3 so we get the UNLIMITED open flag
librsvg_dep = dependency('librsvg-2.0', version: '>=2.40.3', required: get_option('rsvg'))
cairo_dep = dependency('cairo', version: '>=1.2', required: get_option('rsvg'))
if librsvg_dep.found() and cairo_dep.found()
    libvips_deps += librsvg_dep
    libvips_deps += cairo_dep
    cfg_var.set('HAVE_RSVG', '1')
    # 2.46 for rsvg_handle_render_document
    if librsvg_dep.version().version_compare('>=2.46')
        cfg_var.set('HAVE_RSVG_HANDLE_RENDER_DOCUMENT', '1')
    endif
endif

openslide_dep = dependency('openslide', version: '>=3.3.0', required: get_option('openslide'))
if openslide_dep.found()
    if modules_enabled and not get_option('openslide-module').disabled()
        cfg_var.set('OPENSLIDE_MODULE', '1')
    else
        libvips_deps += openslide_dep
    endif
    cfg_var.set('HAVE_OPENSLIDE', '1')
    if openslide_dep.version().version_compare('>=3.4.0')
        cfg_var.set('HAVE_OPENSLIDE_3_4', '1')
    endif
endif

matio_dep = dependency('matio', required: get_option('matio'))
if matio_dep.found()
    libvips_deps += matio_dep
    cfg_var.set('HAVE_MATIO', '1')
endif

# lcms ... refuse to use lcms1
lcms_dep = dependency('lcms2', required: get_option('lcms'))
if lcms_dep.found()
    libvips_deps += lcms_dep
    cfg_var.set('HAVE_LCMS', '1')
    cfg_var.set('HAVE_LCMS2', '1')
endif

# require 1.2.2 since 1.2.1 has a broken ImfCloseTiledInputFile()
openexr_dep = dependency('OpenEXR', version: '>=0.1.2.2', required: get_option('openexr'))
if openexr_dep.found()
    libvips_deps += openexr_dep
    cfg_var.set('HAVE_OPENEXR', '1')
endif

# 2.4 is the first one to have working threading and tiling
libopenjp2_dep = dependency('libopenjp2', version: '>=2.4', required: get_option('openjpeg'))
if libopenjp2_dep.found()
    libvips_deps += libopenjp2_dep
    cfg_var.set('HAVE_LIBOPENJP2', '1')
endif

# we use loadpw etc.
orc_dep = dependency('orc-0.4', version: '>=0.4.11', required: get_option('orc'))
if orc_dep.found()
    libvips_deps += orc_dep
    cfg_var.set('HAVE_ORC', '1')
    # orc 0.4.30+ works with cf-protection, but 0.4.30 has a bug with multiple
    # definitions of OrcTargetPowerPCFlags, so insist on 0.4.31
    if orc_dep.version().version_compare('>=0.4.31')
        cfg_var.set('HAVE_ORC_CF_PROTECTION', '1')
    endif
    if cc.has_function('orc_program_get_error', prefix: '#include <orc/orc.h>', dependencies: orc_dep)
        cfg_var.set('HAVE_ORC_PROGRAM_GET_ERROR', '1')
    endif
endif

# pick 4200 as the starting version number ... no reason, really, it'd
# probably work with much older versions
pdfium_dep = dependency('pdfium', version: '>=4200', required: get_option('pdfium'))
if pdfium_dep.found()
    libvips_deps += pdfium_dep
    cfg_var.set('HAVE_PDFIUM', '1')
endif


libheif_dep = dependency('libheif', version: '>=0.4.11', required: get_option('heif'))
if libheif_dep.found()
    if modules_enabled and not get_option('heif-module').disabled()
        cfg_var.set('HEIF_MODULE', '1')
    else
        libvips_deps += libheif_dep
    endif
    if libheif_dep.get_variable(pkgconfig: 'builtin_h265_decoder', internal: 'builtin_h265_decoder', default_value: 'no') != 'no' or libheif_dep.get_variable(pkgconfig: 'builtin_avif_decoder', internal: 'builtin_avif_decoder', default_value: 'no') != 'no'
        cfg_var.set('HAVE_HEIF_DECODER', '1')
    endif
    if libheif_dep.get_variable(pkgconfig: 'builtin_h265_encoder', internal: 'builtin_h265_encoder', default_value: 'no') != 'no' or libheif_dep.get_variable(pkgconfig: 'builtin_avif_encoder', internal: 'builtin_avif_encoder', default_value: 'no') != 'no'
        cfg_var.set('HAVE_HEIF_ENCODER', '1')
    endif
    if cc.has_function('heif_image_handle_get_raw_color_profile', prefix: '#include <libheif/heif.h>', dependencies: libheif_dep)
        cfg_var.set('HAVE_HEIF_COLOR_PROFILE', '1')
    endif
    if cc.has_member('struct heif_decoding_options', 'convert_hdr_to_8bit', prefix: '#include <libheif/heif.h>', dependencies: libheif_dep)
        cfg_var.set('HAVE_HEIF_DECODING_OPTIONS_CONVERT_HDR_TO_8BIT', '1')
    endif
    if cc.has_function('heif_context_set_maximum_image_size_limit', prefix: '#include <libheif/heif.h>', dependencies: libheif_dep)
        cfg_var.set('HAVE_HEIF_SET_MAX_IMAGE_SIZE_LIMIT', '1')
    endif
    # heif_main_brand added in 1.4.0, but heif_avif appeared in 1.7 ... just check
    # the libheif version number since testing for enums is annoying
    if libheif_dep.version().version_compare('>=1.7.0')
        cfg_var.set('HAVE_HEIF_AVIF', '1')
    endif
endif

libjxl_dep = dependency('libjxl', version: '>=0.5', required: get_option('jpeg-xl'))
libjxl_threads_dep = dependency('libjxl_threads', version: '>=0.5', required: get_option('jpeg-xl'))
if libjxl_dep.found() and libjxl_threads_dep.found()
    if modules_enabled and not get_option('jpeg-xl-module').disabled()
        cfg_var.set('LIBJXL_MODULE', '1')
    else
        libvips_deps += libjxl_dep
        libvips_deps += libjxl_threads_dep
    endif
    cfg_var.set('HAVE_LIBJXL', '1')
    if cc.has_function('JxlEncoderInitBasicInfo', prefix: '#include <jxl/encode.h>', dependencies: libjxl_dep)
        cfg_var.set('HAVE_LIBJXL_JXLENCODERINITBASICINFO', '1')
    endif
endif

libpoppler_dep = dependency('poppler-glib', version: '>=0.16.0', required: get_option('poppler'))
if not cairo_dep.found()
    cairo_dep = dependency('cairo', version: '>=1.2', required: get_option('poppler'))
endif
if libpoppler_dep.found() and cairo_dep.found() and pdfium_dep.found()
    message('PDFium has been found, ignoring Poppler support')
elif libpoppler_dep.found() and cairo_dep.found()
    if modules_enabled and not get_option('poppler-module').disabled()
        cfg_var.set('POPPLER_MODULE', '1')
    else
        libvips_deps += libpoppler_dep
        libvips_deps += cairo_dep
    endif
    cfg_var.set('HAVE_POPPLER', '1')
endif

# niftiio.pc is not always present, so fall back to CMake's find_package() 
# functionality
# cmake find fails on ubuntu due to /usr/lib/x86_64-linux-gnu madness, so it's
# simplest to create a libnifti.pc in this case
# TODO: simplify this when requiring meson>=0.60.0
libnifti_dep = dependency('niftiio', required: get_option('nifti'))
if not libnifti_dep.found()
    libnifti_dep = dependency('NIFTI',
      method: 'cmake',
      modules: ['NIFTI::niftiio'],
      required: get_option('nifti'))
endif
if not get_option('nifti').disabled() and libnifti_dep.found()
    libvips_deps += libnifti_dep
    cfg_var.set('HAVE_NIFTI', '1')
endif

if cc.has_header('sys/file.h')
    cfg_var.set('HAVE_SYS_FILE_H', '1')
endif
if cc.has_header('sys/param.h')
    cfg_var.set('HAVE_SYS_PARAM_H', '1')
endif
if cc.has_header('sys/mman.h')
    cfg_var.set('HAVE_SYS_MMAN_H', '1')
endif
if cc.has_header('unistd.h')
    cfg_var.set('HAVE_UNISTD_H', '1')
endif
if cc.has_header('io.h')
    cfg_var.set('HAVE_IO_H', '1')
endif
if cc.has_header('direct.h')
    cfg_var.set('HAVE_DIRECT_H', '1')
endif
if cc.has_header('windows.h')
    cfg_var.set('HAVE_WINDOWS_H', '1')
endif
if get_option('deprecated')
    cfg_var.set('ENABLE_DEPRECATED', '1')
endif
if get_option('nsgif')
    cfg_var.set('HAVE_NSGIF', '1')
endif
if get_option('ppm')
    cfg_var.set('HAVE_PPM', '1')
endif
if get_option('analyze')
    cfg_var.set('HAVE_ANALYZE', '1')
endif
if get_option('radiance')
    cfg_var.set('HAVE_RADIANCE', '1')
endif

gettext_domain = 'vips@0@.@1@'.format(version_major, version_minor)
cfg_var.set_quoted('GETTEXT_PACKAGE', gettext_domain)
cfg_var.set_quoted('VIPS_PREFIX', prefix_dir)
cfg_var.set_quoted('VIPS_LIBDIR', lib_dir)
if cc.has_function('ngettext')
    cfg_var.set('ENABLE_NLS', 1)
    have_bind_textdomain_codeset = cc.has_function('bind_textdomain_codeset')
else
    libintl_dep = cc.find_library('intl', required: false)
    if libintl_dep.found()
        libvips_deps += libintl_dep
        cfg_var.set('ENABLE_NLS', 1)
        have_bind_textdomain_codeset = cc.has_function('bind_textdomain_codeset', prefix: '#include <libintl.h>', dependencies: libintl_dep)
    else
        have_bind_textdomain_codeset = false
    endif
endif
cfg_var.set('HAVE_BIND_TEXTDOMAIN_CODESET', have_bind_textdomain_codeset)

if host_os == 'darwin'
    profile_dir = '/Library/ColorSync/Profiles'
elif host_os == 'windows'
    # need double escapes since this will get pasted into a #define in a C
    # header ... the C:\Windows is usually overwritten with the result of
    # GetWindowsDirectoryW()
    profile_dir = 'C:\\\\Windows\\\\System32\\\\spool\\\\drivers\\\\color'
else
    profile_dir = get_option('prefix') / get_option('datadir') / 'color' / 'icc'
endif
cfg_var.set_quoted('VIPS_ICC_DIR', profile_dir)

config_file = configure_file(
    configuration: cfg_var,
    output: 'config.h'
)

config_dep = declare_dependency(
    sources: config_file,
    include_directories: include_directories('.'),
    compile_args: '-DHAVE_CONFIG_H=1',
)

libvips_deps += config_dep

subdir('libvips')
if get_option('gtk_doc')
    subdir('doc')
endif
subdir('cplusplus')
subdir('man')
subdir('po')
subdir('tools')
subdir('test')
subdir('fuzz')
