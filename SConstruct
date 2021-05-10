import os
import re
import sys
import glob
import excons
import SCons.Script # pylint: disable=import-error


major = 1
minor = 8
patch = 17

env = excons.MakeBaseEnv()
out_basedir = excons.OutputBaseDirectory()
out_incdir = excons.OutputBaseDirectory() + "/include"
out_libdir = excons.OutputBaseDirectory() + "/lib"

staticlib = (excons.GetArgument("oiio-static", 0, int) != 0)

if sys.platform == "win32":
    # sse2 is implied when building x64 binaries on windows
    simd_directives = ["avx", "avx2"]
else:
    simd_directives = ["sse2", "sse3", "ssse3", "sse4.1", "sse4.2", "avx", "avx2", "avx512f", "f16c"]
simd = excons.GetArgument("oiio-simd", "0")
if simd != "0":
    lst = simd.split(",")
    if sys.platform == "win32":
        lst = map(lambda x: x.lower(), lst)
    lst = filter(lambda x: x in simd_directives, lst)
    if sys.platform == "win32":
        lst = map(lambda x: x.upper(), lst)
    simd = ",".join(lst)

verbose = (1 if excons.GetArgument("oiio-verbose", 0, int) != 0 else 0)

python_dir = excons.GetArgument("oiio-python-dir", "", str)
python_ver = excons.GetArgument("oiio-python-ver", "2.7", str)

prjs = []
oiio_opts = {}
overrides = {}
oiio_dependecies = []

# build options
oiio_opts["LINKSTATIC"] = 1
oiio_opts["USE_fPIC"] = 1
oiio_opts["BUILDSTATIC"] = (1 if staticlib else 0)
oiio_opts["OIIO_THREAD_ALLOW_DCLP"] = 1
oiio_opts["SOVERSION"] = "%s.%s" % (major, minor)
oiio_opts["USE_CPP"] = 11
oiio_opts["USE_LIBCPLUSPLUS"] = 0
oiio_opts["USE_CCACHE"] = 0
oiio_opts["CODECOV"] = 0
oiio_opts["HIDE_SYMBOLS"] = 1
oiio_opts["USE_SIMD"] = simd
oiio_opts["OIIO_BUILD_TESTS"] = 0
oiio_opts["OIIO_BUILD_TOOLS"] = 1
oiio_opts["EMBEDPLUGINS"] = 1
oiio_opts["CMAKE_INSTALL_LIBDIR"] = "lib"
oiio_opts["VERBOSE"] = verbose

# python
oiio_opts["PYLIB_INCLUDE_SONAME"] = 0
oiio_opts["PYLIB_LIB_PREFIX"] = 0
oiio_opts["PYTHON_VERSION"] = python_ver
oiio_opts["USE_PYTHON"] = 1

if python_dir:
    if sys.platform == "win32":
        oiio_opts["PYTHON_LIBRARY"] = "%s/libs/python%s.lib" % (python_dir, python_ver.replace(".", ""))
        oiio_opts["PYTHON_EXECUTABLE"] = "%s/python.exe" % (python_dir)
        oiio_opts["PYTHON_INCLUDE_DIR"] = python_dir + "/include"
    else:
        pybin = "%s/bin/python%s" % (python_dir, python_ver)
        if not os.path.isfile(pybin):
            pybin = "%s/bin/python" % python_dir
        pylib = "%s/lib64/libpython%s.so" % (python_dir, python_ver)
        if not os.path.isfile(pylib):
            pylib = "%s/lib/libpython%s.so" % (python_dir, python_ver)
        oiio_opts["PYTHON_EXECUTABLE"] = pybin
        oiio_opts["PYTHON_LIBRARY"] = pylib
        oiio_opts["PYTHON_INCLUDE_DIR"] = "%s/include/python%s" % (python_dir, python_ver)

# boost
#   force static link
SCons.Script.ARGUMENTS["boost-static"] = "1"
rv = excons.ExternalLibRequire("boost")
if rv["require"]:
   boost_vh = rv["incdir"] + "/boost/version.hpp"
   if not os.path.isfile(boost_vh):
      excons.WarnOnce("No such a file '%s'" % boost_vh, tool="OIIO")
      sys.exit(1)

   e = re.compile(r"define\s+BOOST_VERSION\s+([^\s]+)")
   intver = 0
   strver = ""
   with open(boost_vh, "r") as f:
      for l in f.readlines():
         m = e.search(l)
         if m:
            try:
               intver = int(m.group(1))
               break
            except:
               intver = 0
               pass
   if intver == 0:
      excons.WarnOnce("Cannot figure out Boost version from '%s'" % boost_vh, tool="OIIO")
      sys.exit(1)
   strver = "%d_%d" % (intver / 100000, (intver / 100) % 1000)

   libsuffix = excons.GetArgument("boost-suffix", None)
   if libsuffix is None:
      if sys.platform == "win32":
         libsuffix = "-vc%d-mt-%s" % (int(float(excons.mscver) * 10), strver)
      else:
         libsuffix = ""
   libsuffix += (".lib" if sys.platform == "win32" else ".a")

   libs = []
   for name in ["filesystem", "regex", "system", "thread"]:
      path = rv["libdir"] + "/libboost_" + name + libsuffix
      if not os.path.isfile(path):
         excons.WarnOnce("Invalid Boost %s library '%s'" % (name, path), tool="OIIO")
         sys.exit(1)
      libs.append(path)

   oiio_opts["BOOST_CUSTOM"] = 1
   oiio_opts["BOOST_NO_AUTOLINK"] = 1
   oiio_opts["Boost_VERSION"] = intver
   oiio_opts["Boost_INCLUDE_DIRS"] = rv["incdir"]
   oiio_opts["Boost_LIBRARY_DIRS"] = rv["libdir"]
   oiio_opts["Boost_LIBRARIES"] = ";".join(libs)

   # in boost version more recent than 1.63 (at least), python version was added to the library name
   # look for those in priority
   pylib = rv["libdir"] + "/libboost_python" + python_ver.replace(".", "") + libsuffix
   if not os.path.isfile(pylib):
      pylib = rv["libdir"] + "/libboost_python" + libsuffix
   if not os.path.isfile(pylib):
      SCons.Script.ARGUMENTS["boost-python-static"] = "1"
      pyrv = excons.ExternalLibRequire("boost-python")
      if pyrv["require"] and pyrv["libdir"] != rv["libdir"]:
         pylibsuffix = excons.GetArgument("boost-python-suffix", None)
         if pylibsuffix is None:
            pylibsuffix = libsuffix
         else:
            pylibsuffix += (".lib" if sys.platform == "win32" else ".a")
         pylib = pyrv["libdir"] + "/libboost_python" + python_ver.replace(".", "") + pylibsuffix
         if not os.path.isfile(pylib):
            pylib = pyrv["libdir"] + "/libboost_python" + pylibsuffix
         if pyrv["incdir"] != rv["incdir"]:
            oiio_opts["Boost_INCLUDE_DIRS"] += ";%s" % pyrv["incdir"]
         oiio_opts["Boost_LIBRARY_DIRS"] += ";%s" % pyrv["libdir"]

   if os.path.isfile(pylib):
      oiio_opts["boost_PYTHON_FOUND"] = 1
      oiio_opts["Boost_PYTHON_LIBRARIES"] = pylib
   else:
      excons.WarnOnce("No valid Boost python found. Skipping python module.", tool="OIIO")
      oiio_opts["boost_PYTHON_FOUND"] = 0   
else:
   excons.WarnOnce("Boost is require to build OpenImageIO, please provide root directory using 'with-boost=' flag", tool="OIIO")
   sys.exit(1)

## addtional
oiio_opts["USE_FIELD3D"] = 0
oiio_opts["USE_JPEGTURBO"] = 1
oiio_opts["USE_OPENJPEG"] = 1
oiio_opts["USE_FREETYPE"] = 1
oiio_opts["USE_LIBRAW"] = 1
oiio_opts["USE_OCIO"] = 1
oiio_opts["USE_FFMPEG"] = 0
oiio_opts["USE_OPENCV"] = 0
oiio_opts["USE_PTEX"] = 0
oiio_opts["USE_GIF"] = 0
oiio_opts["USE_JASPER"] = 0
oiio_opts["USE_NUKE"] = 0
oiio_opts["USE_OPENSSL"] = 0
# Not building 'iv' so far
oiio_opts["USE_QT"] = 0
oiio_opts["USE_OPENGL"] = 0
oiio_opts["FORCE_OPENGL_1"] = 0

## extra args
oiio_opts["EXTRA_DSO_LINK_ARGS"] = ""
oiio_opts["EXTRA_CPP_ARGS"] = ""
if sys.platform != "win32":
    oiio_opts["EXTRA_CPP_ARGS"] = " -Wno-deprecated-declarations"

export_zlib = []
export_bzip2 = []
export_jpeg = []
export_openjpeg = []
export_png = []
export_tiff = []
export_raw = []
export_freetype = []
export_ocio = []
export_openexr = []


# zlib (no deps [CMake])
def ZlibName(static):
    return ("z" if sys.platform != "win32" else ("zlib" if static else "zdll"))

def ZlibDefines(static):
    return ([] if  static else ["ZLIB_DLL"])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "zlib", libnameFunc=ZlibName, definesFunc=ZlibDefines)
if not rv["require"]:
    excons.PrintOnce("OIIO: Build zlib from sources ...")
    excons.Call("zlib", targets=["zlib"], imp=["ZlibName", "ZlibPath"])
    z_static = excons.GetArgument("zlib-static", 1, int)
    z_path = ZlibPath(static=z_static) # pylint: disable=undefined-variable
    zlib_outputs = [z_path]

    oiio_opts["ZLIB_INCLUDE_DIR"] = out_incdir
    oiio_opts["ZLIB_LIBRARY"] = z_path

    overrides["with-zlib"] = os.path.dirname(os.path.dirname(z_path))
    overrides["zlib-static"] = z_static
    overrides["zlib-name"] = ZlibName(static=z_static)
    export_zlib += zlib_outputs
else:
    zlib_outputs = []
    export_zlib.append(rv.get("libpath"))
    oiio_opts["ZLIB_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["ZLIB_LIBRARY"] = rv["libpath"]

# bzip2 (no deps [SCons])
def Bzip2Libname(static):
    return ("bz2" if sys.platform != "win32" else "libbz2")

def Bzip2Defines(static):
    return ([] if (sys.platform != "win32" or static) else ["BZ_DLL"])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "bz2", libnameFunc=Bzip2Libname, definesFunc=Bzip2Defines, varPrefix="BZIP2_")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build bzip2 from sources ...")
    excons.Call("bzip2", targets=["bz2"], imp=["BZ2Name", "BZ2Path"])
    bz2_static = excons.GetArgument("bz2-static", 1, int)
    bz2_path = BZ2Path() # pylint: disable=undefined-variable
    bzip2_outputs = [bz2_path]
    
    oiio_opts["BZIP2_LIBRARY"] = bz2_path
    oiio_opts["BZIP2_INCLUDE_DIR"] = out_incdir

    overrides["with-bz2"] = os.path.dirname(os.path.dirname(bz2_path))
    overrides["bz2-static"] = bz2_static
    overrides["bz2-name"] = BZ2Name() # pylint: disable=undefined-variable
    export_bzip2 += bzip2_outputs
else:
    bzip2_outputs = []
    export_bzip2.append(rv.get("libpath"))
    oiio_opts["BZIP2_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["BZIP2_LIBRARY"] = rv["libpath"]

oiio_dependecies += bzip2_outputs

# jpeg (no deps [CMake/Automake])
#overrides["libjpeg-jpeg8"] = 1
def JpegLibname(static):
    return "jpeg"

rv = excons.cmake.ExternalLibRequire(oiio_opts, "libjpeg", libnameFunc=JpegLibname, varPrefix="JPEG_")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build libjpeg-turbo from sources ...")
    excons.Call("libjpeg-turbo", targets=["libjpeg"], overrides=overrides, imp=["LibjpegName", "LibjpegPath"])
    jpeg_static = excons.GetArgument("libjpeg-static", 1, int)
    jpeg_path = LibjpegPath(static=jpeg_static) # pylint: disable=undefined-variable
    jpeg_outputs = [jpeg_path]

    oiio_opts["JPEG_INCLUDE_DIR"] = out_incdir
    oiio_opts["JPEG_LIBRARY"] = jpeg_path

    overrides["with-libjpeg"] = os.path.dirname(os.path.dirname(jpeg_path))
    overrides["libjpeg-static"] = jpeg_static
    overrides["libjpeg-name"] = LibjpegName(jpeg_static) # pylint: disable=undefined-variable
    export_jpeg += jpeg_outputs
else:
    jpeg_outputs = []
    export_jpeg.append(rv.get("libpath"))
    oiio_opts["JPEG_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["JPEG_LIBRARY"] = rv["libpath"]

oiio_dependecies += jpeg_outputs

# openjpeg (no deps [CMake])
rv = excons.cmake.ExternalLibRequire(oiio_opts, "openjpeg")
if not rv["require"]:
    # Openjpeg submodule is referencing v2.3.0 tag
    excons.PrintOnce("OIIO: Build openjpeg from sources ...")
    excons.Call("openjpeg", targets=["openjpeg"], imp=["OpenjpegPath"])
    openjpeg_path = OpenjpegPath() # pylint: disable=undefined-variable
    openjpeg_outputs = [OpenjpegPath()] # pylint: disable=undefined-variable
    oiio_opts["OPENJPEG_HOME"] = excons.OutputBaseDirectory()
    oiio_opts["OPENJPEG_INCLUDE_DIR"] = out_incdir + "/openjpeg-2.3"
    export_openjpeg += openjpeg_outputs
else:
    oiio_opts["OPENJPEG_HOME"] = os.path.dirname(rv["incdir"])
    incdir = None
    if os.path.isdir(rv["incdir"]):
        for item in os.listdir(rv["incdir"]):
            _incdir = rv["incdir"] + "/" + item
            if item.startswith("openjpeg") and os.path.isdir(_incdir):
                incdir = _incdir
                break
    if incdir is None:
        excons.PrintOnce("OIIO: Please specify openjpeg compatibility version using openjpeg-version= flag")
        openjpeg_version = excons.GetArgument("openjpeg-version", "2.1")
        incdir = rv["incdir"] + "/openjpeg-" + openjpeg_version
    oiio_opts["OPENJPEG_INCLUDE_DIR"] = incdir
    openjpeg_outputs = []
    export_openjpeg.append(rv.get("libpath"))

oiio_dependecies += openjpeg_outputs

# png (depends on zlib [CMake])

def PngLibname(static):
    return "png"

def PngDefines(static):
    return (["PNG_USE_DLL"] if (not static and sys.platform == "win32") else [])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "libpng", libnameFunc=PngLibname, definesFunc=PngDefines, varPrefix="PNG_")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build libpng from sources ...")
    excons.cmake.AddConfigureDependencies("libpng", zlib_outputs)
    excons.Call("libpng", targets=["libpng"], overrides=overrides, imp=["LibpngName", "LibpngPath"])
    png_static = excons.GetArgument("libpng-static", 1, int)
    png_path = LibpngPath(png_static) # pylint: disable=undefined-variable
    libpng_outputs = [png_path]

    oiio_opts["PNG_INCLUDE_DIR"] = out_incdir
    oiio_opts["PNG_LIBRARY"] = png_path
    
    overrides["with-libpng"] = os.path.dirname(os.path.dirname(png_path))
    overrides["libpng-static"] = png_static
    overrides["libpng-name"] = LibpngName(png_static) # pylint: disable=undefined-variable
    export_png += libpng_outputs
else:
    libpng_outputs = []
    export_png.append(rv.get("libpath"))
    oiio_opts["PNG_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["PNG_LIBRARY"] = rv["libpath"]

oiio_dependecies += libpng_outputs

# tiff (depends on zlib, jpeg [CMake])

overrides["libtiff-use-jbig"] = 0

def TiffLibname(static):
    return "tiff"

rv = excons.cmake.ExternalLibRequire(oiio_opts, "libtiff", libnameFunc=TiffLibname, varPrefix="TIFF_")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build libtiff from sources ...")
    excons.cmake.AddConfigureDependencies("libtiff", zlib_outputs + jpeg_outputs)
    excons.Call("libtiff", targets=["libtiff"], overrides=overrides, imp=["LibtiffName", "LibtiffPath"])
    tiff_path = LibtiffPath() # pylint: disable=undefined-variable
    tiff_outputs = [tiff_path]

    oiio_opts["TIFF_INCLUDE_DIR"] = out_incdir
    oiio_opts["TIFF_LIBRARY"] = tiff_path

    overrides["with-libtiff"] = os.path.dirname(os.path.dirname(tiff_path))
    overrides["libtiff-static"] = excons.GetArgument("libtiff-static", 1, int)
    overrides["libtiff-name"] = LibtiffName() # pylint: disable=undefined-variable
    export_tiff += tiff_outputs
else:
    tiff_outputs = []
    export_tiff.append(rv.get("libpath"))
    oiio_opts["TIFF_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["TIFF_LIBRARY"] = rv["libpath"]

oiio_dependecies += tiff_outputs

# ocio (depends on tinyxml, yaml-cpp, lcms2 [SCons])

def OCIOLibname(static):
    return "OpenColorIO"

rv = excons.cmake.ExternalLibRequire(oiio_opts, "ocio", libnameFunc=OCIOLibname)
if not rv["require"]:
    if sys.platform == "win32":
        overrides["ocio-use-boost"] = 1
    excons.PrintOnce("OIIO: Build OpenColorIO from sources ...")
    excons.Call("OpenColorIO", targets=["ocio"], overrides=overrides, imp=["OCIOPath", "YamlCppPath", "TinyXmlPath", "LCMS2Name", "LCMS2Path"])
    ocio_static = excons.GetArgument("ocio-static", 1, int) != 0
    ocio_outputs = [OCIOPath(ocio_static), TinyXmlPath(), YamlCppPath(), LCMS2Path()] # pylint: disable=undefined-variable

    oiio_opts["OCIO_INCLUDE_PATH"] = out_incdir
    oiio_opts["OCIO_LIBRARIES"] = OCIOPath(ocio_static) # pylint: disable=undefined-variable
    oiio_opts["LCMS2_LIBRARY"] = LCMS2Path() # pylint: disable=undefined-variable
    oiio_opts["YAML_LIBRARY"] = YamlCppPath() # pylint: disable=undefined-variable
    oiio_opts["TINYXML_LIBRARY"] = TinyXmlPath() # pylint: disable=undefined-variable

    # setup overrides for LibRaw build
    overrides["with-lcms2-inc"] = out_incdir
    overrides["with-lcms2-lib"] = os.path.dirname(LCMS2Path()) # pylint: disable=undefined-variable
    overrides["lcms2-name"] = LCMS2Name() # pylint: disable=undefined-variable
    overrides["lcms2-static"] = excons.GetArgument("lcms2-static", 1, int)

    export_ocio += ocio_outputs
else:
    ocio_outputs = []
    export_ocio.append(rv.get("libpath"))
    oiio_opts["OCIO_INCLUDE_PATH"] = rv["incdir"]
    oiio_opts["OCIO_LIBRARIES"] = rv["libpath"]
    rv = excons.ExternalLibRequire("tinyxml")
    if rv["require"]:
        oiio_opts["TINYXML_LIBRARY"] = rv["libpath"]
    rv = excons.ExternalLibRequire("yamlcpp")
    if rv["require"]:
        oiio_opts["YAML_LIBRARY"] = rv["libpath"]
    rv = excons.ExternalLibRequire("lcms2")
    if rv["require"]:
        oiio_opts["LCMS2_LIBRARY"] = rv["libpath"]

oiio_dependecies += ocio_outputs

# libraw (depends on jpeg, lcms2 [SCons])

overrides["libraw-with-lcms2"] = 1
overrides["libraw-with-jpeg"] = 1

def RawLibname(static):
    return ("raw" if sys.platform != "win32" else "libraw")

def RawDefines(static):
    return (["LIBRAW_NODLL"] if static else [])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "libraw", libnameFunc=RawLibname, definesFunc=RawDefines, varPrefix="LibRaw_")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build libraw from sources ...")
    excons.Call("LibRaw", targets=["libraw"], overrides=overrides, imp=["LibrawPath", "LibrawName"])
    libraw_path = LibrawPath() # pylint: disable=undefined-variable
    libraw_outputs = [libraw_path]

    oiio_opts["LibRaw_INCLUDE_DIR"] = out_incdir
    oiio_opts["LibRaw_r_LIBRARIES"] = libraw_path
    export_raw += libraw_outputs
else:
    libraw_outputs = []
    oiio_opts["LibRaw_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["LibRaw_r_LIBRARIES"] = rv["libpath"]
    export_raw.append(rv.get("libpath"))

oiio_dependecies += libraw_outputs

# freetype (depends on zlib, bzip2, png [CMake])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "freetype")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build freetype from sources ...")
    excons.cmake.AddConfigureDependencies("freetype", zlib_outputs + libpng_outputs + bzip2_outputs)
    excons.Call("freetype", targets=["freetype"], overrides=overrides, imp=["FreetypeName", "FreetypePath"])
    freetype_path = FreetypePath() # pylint: disable=undefined-variable
    freetype_outputs = [freetype_path]
    
    oiio_opts["FREETYPE_INCLUDE_DIR"] = out_incdir
    oiio_opts["FREETYPE_LIBRARY"] = freetype_path
    export_freetype += freetype_outputs
else:
    freetype_outputs = []
    export_freetype.append(rv.get("libpath"))
    oiio_opts["FREETYPE_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["FREETYPE_LIBRARY"] = rv["libpath"]

oiio_dependecies += freetype_outputs

# opexnexr (depends on zlib [SCons])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "openexr")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build openexr from sources ...")
    excons.Call("openexr", targets=["ilmbase", "openexr-static", "openexr-shared"], overrides=overrides, imp=["HalfPath", "IexPath", "ImathPath", "IlmThreadPath", "IlmImfPath"])
    openexr_static = (excons.GetArgument("openexr-static", 1, int) != 0)
    openexr_half = HalfPath(openexr_static) # pylint: disable=undefined-variable
    openexr_iex = IexPath(openexr_static) # pylint: disable=undefined-variable
    openexr_imath = ImathPath(openexr_static) # pylint: disable=undefined-variable
    openexr_ilmt = IlmThreadPath(openexr_static) # pylint: disable=undefined-variable
    openexr_imf = IlmImfPath(openexr_static) # pylint: disable=undefined-variable
    openexr_outputs = [openexr_imf, openexr_imath, openexr_iex, openexr_half, openexr_ilmt]

    oiio_opts["OPENEXR_HOME"] = out_basedir
    oiio_opts["OPENEXR_INCLUDE_DIR"] = out_incdir
    oiio_opts["OPENEXR_HALF_LIBRARY"] = openexr_half
    oiio_opts["OPENEXR_IEX_LIBRARY"] = openexr_iex
    oiio_opts["OPENEXR_IMATH_LIBRARY"] = openexr_imath
    oiio_opts["OPENEXR_ILMTHREAD_LIBRARY"] = openexr_ilmt
    oiio_opts["OPENEXR_ILMIMF_LIBRARY"] = openexr_imf
    export_openexr += openexr_outputs
else:
    openexr_outputs = []

    oiio_opts["OPENEXR_HOME"] = os.path.dirname(rv["incdir"])
    oiio_opts["OPENEXR_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["OPENEXR_HALF_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "Half")
    oiio_opts["OPENEXR_IEX_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "Iex")
    oiio_opts["OPENEXR_IMATH_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "Imath")
    oiio_opts["OPENEXR_ILMTHREAD_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "IlmThread")
    oiio_opts["OPENEXR_ILMIMF_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "IlmImf")
    export_openexr.append(rv.get("libpath"))

oiio_dependecies += openexr_outputs


# oiio build

for k, v in oiio_opts.iteritems():
    if isinstance(v, basestring):
        oiio_opts[k] = v.replace("\\", "/")

def OiioName(static=False):
    libname = "OpenImageIO"
    if sys.platform == "win32" and static:
        libname = "lib" + libname
    return libname

def OiioPath(static=False):
    name = OiioName(static=static)
    if sys.platform == "win32":
        libname = name + ".lib"
    else:
        libname = "lib" + name + (".a" if static else excons.SharedLibraryLinkExt())
    return excons.OutputBaseDirectory() + "/lib/" + libname

def RequireOiio(env, static=False):
    env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
    env.Append(LIBPATH=[excons.OutputBaseDirectory() + "/lib"])
    excons.Link(env, OiioPath(static=static), static=static, force=True, silent=True)


def OiioExtraLibPaths():
    return export_png + export_jpeg + export_openjpeg + export_raw + export_tiff + export_ocio + export_freetype + export_bzip2 + export_openexr + export_zlib


prjs.append({"name": "oiio",
             "type": "cmake",
             "cmake-opts": oiio_opts,
             "cmake-cfgs": excons.CollectFiles(["src"], patterns=["CMakeLists.txt"], recursive=True) + ["CMakeLists.txt"] + oiio_dependecies,
             "cmake-srcs": excons.CollectFiles(["src"], patterns=["*.cpp"], recursive=True),
             "cmake-outputs": map(lambda x: out_incdir + "/OpenImageIO/" + os.path.basename(x), excons.glob("src/include/OpenImageIO/*.h")) +
                              [OiioPath(staticlib)]})

excons.AddHelpOptions(oiio="""OPENIMAGEIO OPTIONS
  oiio-static=0|1        : Toggle between static and shared library build [0]
  oiio-simd=0|<simd>     : Use SIMD directives [0]
                           <simd> should be '0' or a comma separated list of desired SIMD directives.
                           Known directives: %s
  oiio-verbose=0|1       : Print lots of messages while compiling [0]
  oiio-python-dir=<path> : Python prefix ['']
  oiio-python-ver=<ver>  : Python version ['2.7']""" % ", ".join(simd_directives))

excons.DeclareTargets(env, prjs)

SCons.Script.Default("oiio")

SCons.Script.Export("OiioName OiioPath RequireOiio OiioExtraLibPaths")
