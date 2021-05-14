import os
import re
import sys
import glob
import subprocess
import excons
import SCons.Script # pylint: disable=import-error


major = 2
minor = 1
patch = 20


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
oiio_dependencies = []

# build options
oiio_opts["BUILD_DOCS"] = 0
oiio_opts["LINKSTATIC"] = 1
oiio_opts["BUILD_SHARED_LIBS"] = (0 if staticlib else 1)
oiio_opts["OIIO_THREAD_ALLOW_DCLP"] = 1
oiio_opts["SOVERSION"] = "%s.%s" % (major, minor)
oiio_opts["USE_LIBCPLUSPLUS"] = 0
oiio_opts["USE_CCACHE"] = 0
oiio_opts["CODECOV"] = 0
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
oiio_opts["USE_PYTHON"] = excons.GetArgument("oiio-python", 1, int)
oiio_opts["PYBIND11_INCLUDE_DIR"] = os.path.abspath("pybind11/include")
oiio_opts["ROBINMAP_INCLUDE_DIR"] = os.path.abspath("robin-map")


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
boost_outputs = []
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
   oiio_opts["Boost_USE_STATIC_LIBS"] = 1
   oiio_opts["Boost_VERSION"] = intver
   oiio_opts["Boost_INCLUDE_DIRS"] = rv["incdir"]
   oiio_opts["Boost_LIBRARY_DIRS"] = rv["libdir"]
   oiio_opts["Boost_LIBRARIES"] = ";".join(libs)
   boost_outputs = libs

else:
   excons.WarnOnce("Boost is require to build OpenImageIO, please provide root directory using 'with-boost=' flag", tool="OIIO")
   sys.exit(1)

## additional
oiio_opts["USE_FIELD3D"] = 0
oiio_opts["USE_JPEGTURBO"] = 1
oiio_opts["USE_OPENJPEG"] = 1
oiio_opts["USE_FREETYPE"] = 1
oiio_opts["USE_LIBRAW"] = 1
oiio_opts["USE_OPENCOLORIO"] = 1
oiio_opts["USE_FFMPEG"] = 0
oiio_opts["USE_OPENCV"] = 0
oiio_opts["USE_PTEX"] = 0
oiio_opts["USE_GIF"] = 0
oiio_opts["USE_NUKE"] = 0
oiio_opts["USE_WEBP"] = 0
# Not building 'iv' so far
oiio_opts["USE_QT"] = 0
oiio_opts["USE_OPENGL"] = 0

## extra args
oiio_opts["EXTRA_DSO_LINK_ARGS"] = ""
oiio_opts["EXTRA_CPP_ARGS"] = ""
# if sys.platform != "win32":
#     oiio_opts["EXTRA_CPP_ARGS"] = " -Wno-deprecated-declarations"

export_zlib = []
export_bzip2 = []
export_jbig = []
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
    zlib_outputs = [z_path, "{}/zlib.h".format(out_incdir)]
    export_zlib = [z_path]
    oiio_opts["ZLIB_INCLUDE_DIR"] = out_incdir
    oiio_opts["ZLIB_LIBRARY"] = z_path
    oiio_opts["ZLIB_LIBRARY_RELEASE"] = z_path

    overrides["with-zlib"] = os.path.dirname(os.path.dirname(z_path))
    overrides["zlib-static"] = z_static
    overrides["zlib-name"] = ZlibName(static=z_static)
else:
    zlib_outputs = []
    export_zlib = [rv.get("libpath")]
    oiio_opts["ZLIB_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["ZLIB_LIBRARY"] = rv["libpath"]
    oiio_opts["ZLIB_LIBRARY_RELEASE"] = rv["libpath"]

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
    bzip2_outputs = [bz2_path, "{}/bzlib.h".format(out_incdir)]
    export_bzip2 = [bz2_path]
    oiio_opts["BZIP2_LIBRARY"] = bz2_path
    oiio_opts["BZIP2_LIBRARY_RELEASE"] = bz2_path
    oiio_opts["BZIP2_INCLUDE_DIR"] = out_incdir

    overrides["with-bz2"] = os.path.dirname(os.path.dirname(bz2_path))
    overrides["bz2-static"] = bz2_static
    overrides["bz2-name"] = BZ2Name() # pylint: disable=undefined-variable
else:
    bzip2_outputs = []
    bz2_path = rv["libpath"]
    export_bzip2 = [bz2_path]
    oiio_opts["BZIP2_LIBRARY"] = rv["libpath"]
    oiio_opts["BZIP2_LIBRARY_RELEASE"] = rv["libpath"]
    oiio_opts["BZIP2_INCLUDE_DIR"] = rv["incdir"]

oiio_dependencies += bzip2_outputs

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
    jpeg_outputs = [jpeg_path, "{}/jpeglib.h".format(out_incdir)]
    export_jpeg = [jpeg_path]
    oiio_opts["JPEG_INCLUDE_DIR"] = out_incdir
    oiio_opts["JPEG_LIBRARY"] = jpeg_path

    overrides["with-libjpeg"] = os.path.dirname(os.path.dirname(jpeg_path))
    overrides["libjpeg-static"] = jpeg_static
    overrides["libjpeg-name"] = LibjpegName(jpeg_static) # pylint: disable=undefined-variable
else:
    jpeg_outputs = []
    export_jpeg = [rv.get("libpath")]
    oiio_opts["JPEG_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["JPEG_LIBRARY"] = rv["libpath"]

oiio_dependencies += jpeg_outputs

# openjpeg (no deps [CMake])
def OpenjpegLibname(static):
    return "openjp2"

rv = excons.cmake.ExternalLibRequire(oiio_opts, "openjpeg", libnameFunc=OpenjpegLibname)
if not rv["require"]:
    # Openjpeg submodule is referencing v2.4.0 tag
    excons.PrintOnce("OIIO: Build openjpeg from sources ...")
    excons.Call("openjpeg", targets=["openjpeg"], imp=["OpenjpegPath"])
    openjpeg_path = OpenjpegPath() # pylint: disable=undefined-variable
    openjpeg_outputs = [openjpeg_path] # pylint: disable=undefined-variable
    export_openjpeg = [openjpeg_path] # pylint: disable=undefined-variable
    oiio_opts["OpenJpeg_ROOT"] = os.path.dirname(os.path.dirname(openjpeg_path))
else:
    openjpeg_outputs = []
    openjpeg_path = rv["libpath"]
    export_openjpeg = [openjpeg_path]
    oiio_opts["OpenJpeg_ROOT"] = os.path.dirname(rv["libdir"])

oiio_dependencies += openjpeg_outputs

# jbig (no deps [SCons])
def JbigLibname(static):
   return "jbig"

rv = excons.cmake.ExternalLibRequire(oiio_opts, name="jbig", libnameFunc=JbigLibname)
if rv["require"] is None:
   excons.PrintOnce("OIIO: Build jbig from sources ...")
   excons.Call("jbigkit", targets=["jbig"], imp=["RequireJbig", "JbigPath", "JbigName"])
   jbig_path = JbigPath() # pylint: disable=undefined-variable
   jbig_outputs = [JbigPath(), # pylint: disable=undefined-variable
                   "{}/jbig.h".format(out_incdir),
                   "{}/jbig_ar.h".format(out_incdir)]
   export_jbig = [JbigPath()] # pylint: disable=undefined-variable

   overrides["libtiff-use-jbig"] = 1
   overrides["with-jbig"] = os.path.dirname(os.path.dirname(JbigPath())) # pylint: disable=undefined-variable
   overrides["jbig-name"] = JbigName() # pylint: disable=undefined-variable
   overrides["jbig-static"] = 1
else:
   jbig_path = rv.get("libpath")
   jbig_incdir = rv.get("incdir")
   jbig_libdir = rv.get("libdir")

   if os.path.isfile(jbig_path) and os.path.isdir(jbig_incdir) and os.path.isdir(jbig_libdir):
      jbig_outputs = []
      export_jbig = [jbig_path]

      overrides["libtiff-use-jbig"] = 1
      overrides["with-jbig-inc"] = jbig_incdir
      overrides["with-jbig-lib"] = jbig_libdir
      overrides["jbig-name"] = rv.get("libname")
      overrides["jbig-static"] = rv.get("static")
   else:
      overrides["libtiff-use-jbig"] = 0

oiio_dependencies += jbig_outputs

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
    libpng_outputs = [png_path, "{}/png.h".format(out_incdir)]
    export_png = [png_path]
    oiio_opts["PNG_INCLUDE_DIR"] = out_incdir
    oiio_opts["PNG_LIBRARY"] = png_path
    oiio_opts["PNG_LIBRARY_RELEASE"] = png_path
    
    overrides["with-libpng"] = os.path.dirname(os.path.dirname(png_path))
    overrides["libpng-static"] = png_static
    overrides["libpng-name"] = LibpngName(png_static) # pylint: disable=undefined-variable
else:
    libpng_outputs = []
    export_png = [rv.get("libpath")]
    oiio_opts["PNG_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["PNG_LIBRARY"] = rv["libpath"]
    oiio_opts["PNG_LIBRARY_RELEASE"] = rv["libpath"]

oiio_dependencies += libpng_outputs

# tiff (depends on zlib, jpeg, jbig [CMake])
def TiffLibname(static):
    return "tiff"

rv = excons.cmake.ExternalLibRequire(oiio_opts, "libtiff", libnameFunc=TiffLibname, varPrefix="TIFF_")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build libtiff from sources ...")
    excons.cmake.AddConfigureDependencies("libtiff", zlib_outputs + jpeg_outputs + jbig_outputs)
    excons.Call("libtiff", targets=["libtiff"], overrides=overrides, imp=["LibtiffName", "LibtiffPath"])
    tiff_path = LibtiffPath() # pylint: disable=undefined-variable
    tiff_outputs = [tiff_path, "{}/tiff.h".format(out_incdir)]
    export_tiff = [tiff_path]
    oiio_opts["TIFF_INCLUDE_DIR"] = out_incdir
    oiio_opts["TIFF_LIBRARY"] = tiff_path
    oiio_opts["TIFF_LIBRARY_RELEASE"] = tiff_path
    if overrides["libtiff-use-jbig"] != 0:
        # seems TIFF_LIBRARIES get overwritten internal back to tiff_path only
        oiio_opts["TIFF_LIBRARIES"] = "%s;%s" % (tiff_path, jbig_path)

    overrides["with-libtiff"] = os.path.dirname(os.path.dirname(tiff_path))
    overrides["libtiff-static"] = excons.GetArgument("libtiff-static", 1, int)
    overrides["libtiff-name"] = LibtiffName() # pylint: disable=undefined-variable
else:
    tiff_outputs = []
    export_tiff = [rv.get("libpath")]
    oiio_opts["TIFF_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["TIFF_LIBRARY"] = rv["libpath"]
    oiio_opts["TIFF_LIBRARY_RELEASE"] = rv["libpath"]
    if overrides["libtiff-use-jbig"] != 0:
        # seems TIFF_LIBRARIES get overwritten internal back to tiff_path only
        oiio_opts["TIFF_LIBRARIES"] = "%s;%s" % (rv["libpath"], jbig_path)

oiio_dependencies += tiff_outputs

# ocio (depends on tinyxml, yaml-cpp, lcms2 [SCons])

def OCIOLibname(static):
    return "OpenColorIO"

rv = excons.cmake.ExternalLibRequire(oiio_opts, "ocio", libnameFunc=OCIOLibname)
if not rv["require"]:
    if sys.platform == "win32":
        overrides["ocio-use-boost"] = 1
    excons.PrintOnce("OIIO: Build OpenColorIO from sources ...")
    excons.Call("OpenColorIO", targets=["ocio-libs"], overrides=overrides, imp=["OCIOPath", "YamlCppPath", "TinyXmlPath", "LCMS2Name", "LCMS2Path"])
    ocio_static = excons.GetArgument("ocio-static", 1, int) != 0
    export_ocio = [OCIOPath(ocio_static), TinyXmlPath(), YamlCppPath(), LCMS2Path()] # pylint: disable=undefined-variable
    ocio_outputs = export_ocio + ["{}/OpenColorIO/OpenColorIO.h".format(out_incdir)] # pylint: disable=undefined-variable
    oiio_opts["OPENCOLORIO_INCLUDE_PATH"] = out_incdir
    oiio_opts["OPENCOLORIO_LIBRARY"] = ocio_outputs[0]
    oiio_opts["OPENCOLORIO_LIBRARIES"] = ocio_outputs[0]
    oiio_opts["LCMS2_LIBRARY"] = LCMS2Path() # pylint: disable=undefined-variable
    if overrides["libtiff-use-jbig"] != 0:
        oiio_opts["LCMS2_LIBRARIES"] = "%s;%s" % (oiio_opts["LCMS2_LIBRARY"], jbig_path)
    oiio_opts["YAML_LIBRARY"] = YamlCppPath() # pylint: disable=undefined-variable
    oiio_opts["TINYXML_LIBRARY"] = TinyXmlPath() # pylint: disable=undefined-variable

    overrides["with-lcms2-inc"] = out_incdir
    overrides["with-lcms2-lib"] = os.path.dirname(LCMS2Path()) # pylint: disable=undefined-variable
    overrides["lcms2-name"] = LCMS2Name() # pylint: disable=undefined-variable
    overrides["lcms2-static"] = excons.GetArgument("lcms2-static", 1, int)
else:
    ocio_outputs = []
    export_ocio = [rv.get("libpath")]

    oiio_opts["OPENCOLORIO_INCLUDE_PATH"] = rv["incdir"]
    oiio_opts["OPENCOLORIO_LIBRARY"] = rv["libpath"]
    oiio_opts["OPENCOLORIO_LIBRARIES"] = rv["libpath"]

    rv = excons.ExternalLibRequire("tinyxml")
    if rv["require"]:
        oiio_opts["TINYXML_LIBRARY"] = rv["libpath"]
        export_ocio.append(rv["libpath"])

    rv = excons.ExternalLibRequire("yamlcpp")
    if rv["require"]:
        oiio_opts["YAML_LIBRARY"] = rv["libpath"]
        export_ocio.append(rv["libpath"])

    rv = excons.ExternalLibRequire("lcms2")
    if rv["require"]:
        oiio_opts["LCMS2_LIBRARY"] = rv["libpath"]
        if overrides["libtiff-use-jbig"] != 0:
            oiio_opts["LCMS2_LIBRARIES"] = "%s;%s" % (rv["libpath"], jbig_path)
        export_ocio.append(rv["libpath"])

oiio_dependencies += ocio_outputs

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
    libraw_outputs = [libraw_path, "{}/libraw/libraw.h".format(out_incdir)]
    export_raw = [libraw_path]
    oiio_opts["LibRaw_INCLUDE_DIR"] = out_incdir
    oiio_opts["LibRaw_r_LIBRARIES"] = libraw_path
else:
    libraw_outputs = []
    oiio_opts["LibRaw_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["LibRaw_r_LIBRARIES"] = rv["libpath"]
    export_raw = [rv.get("libpath")]

oiio_dependencies += libraw_outputs

# freetype (depends on zlib, bzip2, png [CMake])

rv = excons.cmake.ExternalLibRequire(oiio_opts, "freetype")
if not rv["require"]:
    excons.PrintOnce("OIIO: Build freetype from sources ...")
    excons.cmake.AddConfigureDependencies("freetype", zlib_outputs + libpng_outputs + bzip2_outputs)
    excons.Call("freetype", targets=["freetype"], overrides=overrides, imp=["FreetypeName", "FreetypePath"])
    freetype_path = FreetypePath() # pylint: disable=undefined-variable
    freetype_outputs = [freetype_path, "{}/freetype2/freetype/freetype.h".format(out_incdir)]
    export_freetype = [freetype_path]
    oiio_opts["FREETYPE_INCLUDE_DIR"] = out_incdir
    oiio_opts["FREETYPE_LIBRARY"] = freetype_path
    oiio_opts["FREETYPE_LIBRARIES"] = "%s;%s" % (freetype_path, bz2_path)
else:
    freetype_outputs = []
    export_freetype = [rv.get("libpath")]
    oiio_opts["FREETYPE_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["FREETYPE_LIBRARY"] = rv["libpath"]
    oiio_opts["FREETYPE_LIBRARIES"] = "%s;%s" % (freetype_path, bz2_path)

oiio_dependencies += freetype_outputs

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

    oiio_opts["OPENEXR_INCLUDE_DIR"] = out_incdir
    oiio_opts["OPENEXR_HALF_LIBRARY"] = openexr_half
    oiio_opts["OPENEXR_IEX_LIBRARY"] = openexr_iex
    oiio_opts["OPENEXR_IMATH_LIBRARY"] = openexr_imath
    oiio_opts["OPENEXR_ILMTHREAD_LIBRARY"] = openexr_ilmt
    oiio_opts["OPENEXR_ILMIMF_LIBRARY"] = openexr_imf
    export_openexr = [out_basedir, out_incdir, openexr_half, openexr_iex, openexr_imath, openexr_ilmt, openexr_imf]
else:
    openexr_outputs = []
    export_openexr = []

    oiio_opts["OPENEXR_INCLUDE_DIR"] = rv["incdir"]
    oiio_opts["OPENEXR_HALF_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "Half")
    oiio_opts["OPENEXR_IEX_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "Iex")
    oiio_opts["OPENEXR_IMATH_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "Imath")
    oiio_opts["OPENEXR_ILMTHREAD_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "IlmThread")
    oiio_opts["OPENEXR_ILMIMF_LIBRARY"] = rv["libdir"] + "/" + os.path.basename(rv["libpath"]).replace("openexr", "IlmImf")

    export_openexr.append(oiio_opts["OPENEXR_HALF_LIBRARY"])
    export_openexr.append(oiio_opts["OPENEXR_IEX_LIBRARY"])
    export_openexr.append(oiio_opts["OPENEXR_IMATH_LIBRARY"])
    export_openexr.append(oiio_opts["OPENEXR_ILMTHREAD_LIBRARY"])
    export_openexr.append(oiio_opts["OPENEXR_ILMIMF_LIBRARY"])

oiio_dependencies += openexr_outputs

if not os.path.isfile("pybind11/include/pybind11/pybind11.h"):
    cmd = "git submodule update --init pybind11"
    p = subprocess.Popen(cmd, shell=True)
    p.communicate()

if not os.path.isfile("robin-map/tsl/robin_map.h"):
    cmd = "git submodule update --init robin-map"
    p = subprocess.Popen(cmd, shell=True)
    p.communicate()



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

def OiioVersion():
    return "{}.{}.{}".format(major, minor, path)

def OiioExtraLibPaths():
    return export_png + export_jpeg + export_jbig + export_openjpeg + export_raw + export_tiff + export_ocio + export_freetype + export_bzip2 + export_openexr + export_zlib + boost_outputs


prjs.append({"name": "oiio",
             "type": "cmake",
             "cmake-opts": oiio_opts,
             "cmake-flags": "-Wno-dev",
             "cmake-cfgs": excons.CollectFiles(["src"], patterns=["CMakeLists.txt"], recursive=True) +
                           excons.CollectFiles(["src/cmake"], patterns=["*.cmake"], recursive=True) +
                           ["CMakeLists.txt"] + oiio_dependencies,
             "cmake-srcs": excons.CollectFiles(["src"], patterns=["*.cpp"], recursive=True),
             "cmake-outputs": map(lambda x: out_incdir + "/OpenImageIO/" + os.path.basename(x), excons.glob("src/include/OpenImageIO/*.h")) +
                              [OiioPath(staticlib)]})

excons.AddHelpOptions(oiio="""OPENIMAGEIO OPTIONS
  oiio-static=0|1        : Toggle between static and shared library build [0]
  oiio-simd=0|<simd>     : Use SIMD directives [0]
                           <simd> should be '0' or a comma separated list of desired SIMD directives.
                           Known directives: %s
  oiio-verbose=0|1       : Print lots of messages while compiling [0]
  oiio-python=0|1        : Make python bind [1]
  oiio-python-dir=<path> : Python prefix ['']
  oiio-python-ver=<ver>  : Python version ['2.7']""" % ", ".join(simd_directives))

excons.DeclareTargets(env, prjs)

SCons.Script.Default("oiio")

SCons.Script.Export("OiioName OiioPath RequireOiio OiioExtraLibPaths OiioVersion")
