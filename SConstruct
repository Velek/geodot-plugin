#!python
import os
from posixpath import normpath
import subprocess
import sys
from shutil import copyfile

opts = Variables([], ARGUMENTS)
opts.Add(
    'ANDROID_NDK_ROOT',
    'Path to your Android NDK installation. By default, uses ANDROID_NDK_ROOT from your defined environment variables.',
    os.environ.get("ANDROID_NDK_ROOT", None)
)
opts.Add(
    'android_api_level',
    'Target Android API level',
    '18' if ARGUMENTS.get("android_arch", 'armv7') in ['armv7', 'x86'] else '21'
)
opts.Add(EnumVariable(
    'android_arch',
    'Target Android architecture',
    'arm64v8',
    ['armv7','arm64v8','x86','x86_64']
))
opts.Add(PathVariable(
    'osgeo_path',
    'Path to GDAL headers and libraries',
    None,
    PathVariable.PathIsDir
))

# Gets the standard flags CC, CCX, etc.
try:
    env = DefaultEnvironment(tools=['default', 'compilation_db'])
except:
    print("Compilation Database creation is not supported. Please consider upgrading to SCons 4.x.x!")
    env = DefaultEnvironment()

# Local dependency paths, adapt them to your setup
godot_headers_path = "godot-cpp/godot_headers/"
cpp_bindings_path = "godot-cpp/"
cpp_library = "libgodot-cpp"

rte_cpp_path = "src/raster-tile-extractor/"
rte_libpath = "src/raster-tile-extractor/build/"
rte_library = "libRasterTileExtractor"

vector_cpp_path = "src/vector-extractor/"
vector_libpath = "src/vector-extractor/build/"
vector_library = "libVectorExtractor"

demo_path = "demo/addons/geodot/"

# Try to detect the host platform automatically.
# This is used if no `platform` argument is passed
if sys.platform.startswith('linux'):
    host_platform = 'linux'
elif sys.platform == 'darwin':
    host_platform = 'osx'
elif sys.platform == 'win32' or sys.platform == 'msys':
    host_platform = 'windows'
elif sys.platform == 'android':
    host_platform = 'android'
else:
    host_platform = "Unknown platform: " + sys.platform

# Define our options
opts.Add(EnumVariable('target', "Compilation target", 'debug', ['d', 'debug', 'r', 'release']))
opts.Add(EnumVariable('platform', 'Compilation platform', host_platform,
                      allowed_values=('linux', 'osx', 'windows', 'android'), ignorecase=2))
opts.Add(BoolVariable('use_llvm', "Use the LLVM / Clang compiler", 'no'))
opts.Add(BoolVariable('compiledb', "Build a Compilation Database, e.g. for live error reporting in VSCodium", 'no'))
opts.Add(PathVariable('target_path', 'The path where the lib is installed.', demo_path))
opts.Add(PathVariable('target_name', 'The library name.', 'libgeodot', PathVariable.PathAccept))
opts.Add(PathVariable('osgeo_path', "(Windows only) path to OSGeo installation", "", PathVariable.PathAccept))

# only support 64 at this time..
bits = 64

# Updates the environment with the option variables.
opts.Update(env)

# Process some arguments
if env['use_llvm']:
    env['CC'] = 'clang'
    env['CXX'] = 'clang++'

if env['compiledb']:
    env.CompilationDatabase()

if env['platform'] == '':
    print("No valid target platform selected.")
    quit()


# Build the extractor libraries
subprocess.call(
    "cd " + rte_cpp_path + " && scons platform=" + env['platform'] + " osgeo_path=" + env['osgeo_path'],
    shell=True)
subprocess.call(
    "cd " + vector_cpp_path + " && scons platform=" + env['platform'] + " osgeo_path=" + env['osgeo_path'],
    shell=True)

# For reference:
# - CCFLAGS are compilation flags shared between C and C++
# - CFLAGS are for C-specific compilation flags
# - CXXFLAGS are for C++-specific compilation flags
# - CPPFLAGS are for pre-processor flags
# - CPPDEFINES are for pre-processor defines
# - LINKFLAGS are for linking flags

env.Append(CXXFLAGS=['-std=c++17'])

# Check our platform specifics
if env['platform'] == "android":
    cpp_library += '.android'
    gdal_include_path = os.path.join(env['osgeo_path'], "include")
    gdal_lib_path = os.path.join(env['osgeo_path'], "lib")    
    gdal_lib_name = 'gdal'

    if host_platform == 'windows':
        env = env.Clone(tools=['mingw'])
        env["SPAWN"] = mySpawn

    # Verify NDK root
    if not 'ANDROID_NDK_ROOT' in env:
        raise ValueError("To build for Android, ANDROID_NDK_ROOT must be defined. Please set ANDROID_NDK_ROOT to the root folder of your Android NDK installation.")

    # Validate API level
    api_level = int(env['android_api_level'])
    if env['android_arch'] in ['x86_64', 'arm64v8'] and api_level < 21:
        print("WARN: 64-bit Android architectures require an API level of at least 21; setting android_api_level=21")
        env['android_api_level'] = '21'
        api_level = 21

    # Setup toolchain
    toolchain = env['ANDROID_NDK_ROOT'] + "/toolchains/llvm/prebuilt/"
    if host_platform == "windows":
        toolchain += "windows"
        import platform as pltfm
        if pltfm.machine().endswith("64"):
            toolchain += "-x86_64"
    elif host_platform == "linux":
        toolchain += "linux-x86_64"
    elif host_platform == "osx":
        toolchain += "darwin-x86_64"
    env.PrependENVPath('PATH', toolchain + "/bin") # This does nothing half of the time, but we'll put it here anyways

    # Get architecture info
    arch_info_table = {
        "armv7" : {
            "march":"armv7-a", "target":"armv7a-linux-androideabi", "tool_path":"arm-linux-androideabi", "compiler_path":"armv7a-linux-androideabi",
            "ccflags" : ['-mfpu=neon']
            },
        "arm64v8" : {
            "march":"armv8-a", "target":"aarch64-linux-android", "tool_path":"aarch64-linux-android", "compiler_path":"aarch64-linux-android",
            "ccflags" : []
            },
        "x86" : {
            "march":"i686", "target":"i686-linux-android", "tool_path":"i686-linux-android", "compiler_path":"i686-linux-android",
            "ccflags" : ['-mstackrealign']
            },
        "x86_64" : {"march":"x86-64", "target":"x86_64-linux-android", "tool_path":"x86_64-linux-android", "compiler_path":"x86_64-linux-android",
            "ccflags" : []
        }
    }
    arch_info = arch_info_table[env['android_arch']]

    # Setup tools
    env['CC'] = toolchain + "/bin/" + arch_info['target'] + env['android_api_level'] + "-clang"
    env['CXX'] = toolchain + "/bin/" + arch_info['target'] + env['android_api_level'] + "-clang++"
    print(env['CXX'])
    env['AR'] = toolchain + "/bin/" + arch_info['tool_path'] + "-ar"
    env.Append(CPPPATH=[gdal_include_path])

    env.Append(CCFLAGS=['--target=' + arch_info['target'] + env['android_api_level'], '-march=' + arch_info['march'], '-fPIC'])#, '-fPIE', '-fno-addrsig', '-Oz'])
    env.Append(CCFLAGS=arch_info['ccflags'])
    env.Append(LINKFLAGS='-march=' + arch_info['march'])
    env.Append(LINKFLAGS='-v')
elif env['platform'] in ('x11', 'linux'):
    env['target_path'] += 'x11/'
    cpp_library += '.linux'
    gdal_lib_name = 'gdal'

    env.Append(LINKFLAGS=[
        '-Wl,--disable-new-dtags,-rpath,\'$$ORIGIN\''
    ])

    if env['target'] in ('debug', 'd'):
        env.Append(CCFLAGS=['-fPIC', '-g3', '-Og'])
    else:
        env.Append(CCFLAGS=['-fPIC', '-g', '-O3'])
    
    # Arch needs different includes!
    import distro
    if distro.like() == "arch" or distro.id() == "arch":
        env.Append(CPPDEFINES=["_ARCH"])

elif env['platform'] == "windows":
    env['target_path'] += 'win64/'
    env['target_name'] += ".dll"
    env.Append(LINKFLAGS=['-static-libgcc', '-static-libstdc++', '-static'])
    cpp_library += '.windows'
    gdal_lib_name = 'gdal.dll'

    # Set the compiler to MinGW (is this command valid on native Windows too?)
    env.Replace(CXX=['x86_64-w64-mingw32-g++'])

    gdal_include_path = os.path.join(env['osgeo_path'], "include")
    env.Append(CPPPATH=[gdal_include_path])


if env['target'] in ('debug', 'd'):
    cpp_library += '.debug'
else:
    cpp_library += '.release'

cpp_library += '.arm64v8'

# make sure our binding library is properly includes
env.Append(CPPPATH=['.', godot_headers_path, cpp_bindings_path + 'include/',
                    cpp_bindings_path + 'include/core/', cpp_bindings_path + 'include/gen/', rte_cpp_path, vector_cpp_path])
env.Append(LIBPATH=[cpp_bindings_path + 'bin/', rte_libpath, vector_libpath, os.path.join(env['osgeo_path'], "lib")])
env.Append(LIBS=[cpp_library, rte_library, vector_library, gdal_lib_name])

# tweak this if you want to use different folders, or more folders, to store your source code in.
env.Append(CPPPATH=['src/'])
env.Append(CPPPATH=['src/global/'])
sources = Glob('src/*.cpp')

library = env.SharedLibrary(target=env['target_path'] + env['target_name'], source=sources)

# Generates help for the -h scons option.
Help(opts.GenerateHelpText(env))
