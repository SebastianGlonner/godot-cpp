#!python

import os, subprocess, platform, sys
import pprint

def add_sources(env, sources, dir, extension):
  for f in os.listdir(dir):
      if f.endswith('.' + extension):
          sources.append(env.SharedObject(dir + '/' + f))

# Try to detect the host platform automatically
# This is used if no `platform` argument is passed
if sys.platform.startswith('linux'):
    host_platform = 'linux'
elif sys.platform == 'darwin':
    host_platform = 'osx'
elif sys.platform == 'win32':
    host_platform = 'windows'
else:
    raise ValueError('Could not detect platform automatically, please specify with platform=<platform>')

opts = Variables([], ARGUMENTS)

opts.Add(EnumVariable('platform', 'Target platform', host_platform,
                    allowed_values=('linux', 'osx', 'windows', 'android'),
                    ignorecase=2))
opts.Add(EnumVariable('bits', 'Target platform bits', '64', ('default', '32', '64')))
opts.Add(BoolVariable('use_llvm', 'Use the LLVM compiler - only effective when targeting Linux', False))
opts.Add(BoolVariable('use_mingw', 'Use the MinGW compiler - only effective on Windows', False))
# Must be the same setting as used for cpp_bindings
opts.Add(EnumVariable('target', 'Compilation target', 'debug',
                    allowed_values=('debug', 'release'),
                    ignorecase=2))
opts.Add(PathVariable('headers_dir', 'Path to the directory containing Godot headers', 'godot_headers', PathVariable.PathIsDir))
opts.Add(BoolVariable('use_custom_api_file', 'Use a custom JSON API file', False))
opts.Add(PathVariable('custom_api_file', 'Path to the custom JSON API file', None, PathVariable.PathIsFile))
opts.Add(BoolVariable('generate_bindings', 'Generate GDNative API bindings', False))
opts.Add('extra_suffix', "Custom extra suffix added to the base filename of all generated binary files", '')

import build_android
platform_opts = build_android.get_opts();

for o in platform_opts:
    opts.Add(o)

unknown = opts.UnknownVariables()
if unknown:
    print ("Unknown variables:", unknown.keys())
    Exit(1)

custom_tools = ['default']
platform_arg = ARGUMENTS.get("platform", ARGUMENTS.get("p", False))

if os.name == "nt" and (platform_arg == "android" or ARGUMENTS.get("use_mingw", False)):
    custom_tools = ['mingw']

target_architecture = '';
if ARGUMENTS.get("platform", '') == 'windows':
    if ARGUMENTS.get("bits", '') == '64':
        target_architecture ='amd64'
    elif ARGUMENTS.get("bits", '') == '32':
        target_architecture ='x86'

if target_architecture != '' and not ARGUMENTS.get("use_mingw", False):
    # This makes sure to keep the session environment variables on Windows
    # This way, you can run SCons in a Visual Studio 2017 prompt and it will find all the required tools
    env = Environment(TARGET_ARCH=target_architecture)
else:
    env = Environment(tools=custom_tools)

opts.Update(env)
Help(opts.GenerateHelpText(env))

env.extra_suffix = ''

is64 = sys.maxsize > 2**32
if env['bits'] == 'default':
    env['bits'] = '64' if is64 else '32'

if env['platform'] == 'linux':
    if env['use_llvm']:
        env['CXX'] = 'clang++'

    env.Append(CCFLAGS=['-fPIC', '-g', '-std=c++14', '-Wwrite-strings'])
    env.Append(LINKFLAGS=["-Wl,-R,'$$ORIGIN'"])

    if env['target'] == 'debug':
        env.Append(CCFLAGS=['-Og'])
    elif env['target'] == 'release':
        env.Append(CCFLAGS=['-O3'])

    if env['bits'] == '64':
        env.Append(CCFLAGS=['-m64'])
        env.Append(LINKFLAGS=['-m64'])
    elif env['bits'] == '32':
        env.Append(CCFLAGS=['-m32'])
        env.Append(LINKFLAGS=['-m32'])

elif env['platform'] == 'osx':
    if env['bits'] == '32':
        raise ValueError('Only 64-bit builds are supported for the macOS target.')

    env.Append(CCFLAGS=['-g', '-std=c++14', '-arch', 'x86_64'])
    env.Append(LINKFLAGS=['-arch', 'x86_64', '-framework', 'Cocoa', '-Wl,-undefined,dynamic_lookup'])

    if env['target'] == 'debug':
        env.Append(CCFLAGS=['-Og'])
    elif env['target'] == 'release':
        env.Append(CCFLAGS=['-O3'])

elif env['platform'] == 'windows':
    if host_platform == 'windows' and not env['use_mingw']:
        # MSVC
        env.Append(LINKFLAGS=['/WX'])
        if env['target'] == 'debug':
            env.Append(CCFLAGS=['/EHsc', '/D_DEBUG', '/MDd'])
        elif env['target'] == 'release':
            env.Append(CCFLAGS=['/O2', '/EHsc', '/DNDEBUG', '/MD'])
    else:
        # MinGW
        # following commands does not exist in my environment, probably for linux cross compilation?
        # if env['bits'] == '64':
        #     env['CXX'] = 'x86_64-w64-mingw32-g++'
        # elif env['bits'] == '32':
        #     env['CXX'] = 'i686-w64-mingw32-g++'

        env.Append(CCFLAGS=['-g', '-O3', '-std=c++14', '-Wwrite-strings'])
        env.Append(LINKFLAGS=['--static', '-Wl,--no-undefined', '-static-libgcc', '-static-libstdc++'])

elif env['platform'] == 'android':

    build_android.to_build_android(env)



env.Append(CPPPATH=['.', env['headers_dir'], 'include', 'include/gen', 'include/core'])

# Generate bindings?
json_api_file = ''

if env['use_custom_api_file']:
    json_api_file = env['custom_api_file']
else:
    json_api_file = os.path.join(os.getcwd(), 'godot_headers', 'api.json')

if env['generate_bindings']:
    # Actually create the bindings here

    import binding_generator

    binding_generator.generate_bindings(json_api_file)

# source to compile
sources = []
add_sources(env, sources, 'src/core', 'cpp')
add_sources(env, sources, 'src/gen', 'cpp')

file_suffix = '.' + env['bits']
if env['platform'] == 'android':
    file_suffix = '.a'

if env['extra_suffix'] != '':
    file_suffix = env['extra_suffix'] + file_suffix

#myDict = env.Dictionary()
#pp = pprint.PrettyPrinter()
#pp.pprint(myDict)
#sys.exit("exit")

library = env.StaticLibrary(
    target='bin/' + 'libgodot-cpp.{}.{}{}'.format(env['platform'], env['target'], file_suffix), source=sources
)

Default(library)
