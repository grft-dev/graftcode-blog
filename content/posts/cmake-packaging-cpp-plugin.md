---
title: CMake C++ Plugin Packaging with vcpkg
slug: cmake-packaging-cpp-plugin
date: 2026-11-17T08:04:07.008Z
author: Adam Wasielewski
category: General
readingTime: 25
coverImage: /uploads/cmake-packaging-cpp-plugin/image2.png
---

## TL;DR

* A native plugin is loaded by a host program that the plugin's author does not control, so the build has to produce one shared library, or one flat folder of them, that loads on a machine with no compiler, no vcpkg, and no OpenSSL installed.
* On Windows, a static MSVC runtime needs three settings that agree with each other: policy CMP0091 set to NEW, CMAKE\_MSVC\_RUNTIME\_LIBRARY set to a static value, and a vcpkg triplet that builds ports against the same runtime. If one is missing, a dependency compiles against /MD and the plugin needs the Visual C++ Redistributable again.
* On Linux and macOS, install(RUNTIME\_DEPENDENCY\_SET) finds the libraries a plugin imports and copies the ones the operating system does not provide, and an install rpath of $ORIGIN or @loader\_path lets the loader find the copies beside the plugin.
* The graftcode-extensions repository packages five transport plugins this way, with shared helper files in a cmake/ folder, one release workflow with a 30-job build matrix, and a packaging step that fails when the plugin library is missing from the archive.
* Each release archive is flat, so a client puts the plugin library next to the Gateway and installs no curl, OpenSSL, cloud SDK, or Redistributable, with one documented exception: the two libsasl2 files that ship beside the Linux Kafka plugin.

Native plugins fail in a particular way. The code compiles, the tests pass on the build machine, and the library then refuses to load on a customer's server because a dependency that was present on the build machine is missing there. The cause is rarely the plugin's own code. It is a runtime library that was linked dynamically, an OpenSSL copy that lives in a folder nobody else has, or a bundled library whose search path still points at the build tree.

This piece starts with what CMake and a shared-library plugin are, then walks through a working answer from the graftcode-extensions repository, which builds five transport plugins for six platforms from the same CMake files. It covers the static runtime settings for Windows, the project layout, the install step that collects runtime dependencies, the search path fix for Linux and macOS, a small standalone project you can run yourself, the test setup, and the release workflow that turns the results into archives. It also shows what the flat archive changes for the person running the Gateway, and which failures to expect when you reuse the helpers for your own plugin.

## How CMake Turns One Project Description into Native Builds

CMake does not compile code itself. According to its [manual](https://cmake.org/cmake/help/latest/manual/cmake.1.html), it is a cross-platform buildsystem generator that creates a build system locally for each user through a backend called a generator. A project describes its targets once in CMakeLists.txt files, CMake writes the files that the local build tool understands, and cmake --build runs that tool. The same description therefore produces a Makefile or Ninja build on Linux and macOS and a Visual Studio build on Windows, which is the reason a project that ships for six platforms is written in CMake.

![](/uploads/cmake-packaging-cpp-plugin/image9.png)

Three commands cover most of what the rest of this piece uses. cmake -S . -B build reads the project in the current folder and writes the generated build files into build. cmake --build build runs the native build tool on them. cmake --install build copies the finished files to an install location. Each target in the project, whether a program or a library, has a type, and the type decides how its code reaches the people who run it.

## What Separates a Shared Library from a Static One

The [add\_library reference](https://cmake.org/cmake/help/latest/command/add_library.html) describes a static library as an archive of object files used when linking other targets, so its code is copied into whatever links it. It describes a shared library as a dynamic library that other targets can link against and load at runtime: a .so file on Linux, a .dylib file on macOS, and a .dll file on Windows, where the linker also uses a separate import library.

A plugin, in this article, is a shared library of that second kind. The [graftcode-extensions repository](https://github.com/grft-dev/graftcode-extensions/blob/main/README.md) uses plugins to carry Graft calls over an external channel such as a message broker. Graftcode lets one service call another service's public methods through a generated, strongly typed interface called a Graft, which the calling service uses like a local library. The Graftcode Gateway (gg) is the process that hosts the called service and exposes those methods; [docs.graftcode.com](https://docs.graftcode.com/) describes the full model. The host program, the Gateway, selects a plugin by configuration and looks for the library file in the config file's folder first, then in its current working directory. The host decides when to load the file and where to look for it, and the plugin's author decides neither. That one fact explains most of the build work in the rest of the piece.

## Why a Plugin Has to Bring Its Own Dependencies

A plugin rarely stands alone. The Google Cloud Pub/Sub plugin, which is the running example here, calls the Pub/Sub JSON REST API through libcurl, signs service-account tokens with OpenSSL, and reads and writes JSON with nlohmann/json, according to its [Readme](https://github.com/grft-dev/graftcode-extensions/blob/main/communication-plugins/pubsub/Readme.md). Each of those libraries is a dependency, and a dependency can reach the finished plugin in two ways. It can be linked statically, which copies its code into the plugin, or dynamically, which leaves a reference that the operating system's loader must resolve by finding the library file when the plugin loads.

![](/uploads/cmake-packaging-cpp-plugin/image4.png)

Dynamic references are convenient on the machine that built the plugin, because the libraries are installed there. On the machine that runs it, the same library may be missing, a different version, or in a folder the loader does not search. An application can ask its installer to put the right libraries in place. A plugin cannot, since it is loaded into a program that someone else installed. That is why the failure described in the introduction looks the same every time.

A build that avoids it has three jobs. It links as much as possible into the plugin itself, so the list of remaining references is short. It identifies the remaining references and copies the ones the operating system does not already provide. It also records a search path for the copies relative to the library's own location, since the folder name on the customer's machine is unknown at build time. Each job uses a different mechanism on each operating system, which is why the repository keeps them in shared helper files instead of repeating them in every plugin.

## How the graftcode-extensions Repository Is Organized

The repository holds five plugins in separate folders: RabbitMQ, Azure Service Bus, Kafka, Amazon SQS, and Google Cloud Pub/Sub. Each has its own CMakeLists.txt, its own tests, and its own list of dependencies. A cmake/ folder next to them under communication-plugins/ holds the code they share, and a single release workflow builds all of them.

| File                                                | Job it does                                                                          |
| :-------------------------------------------------- | :----------------------------------------------------------------------------------- |
| cmake/StaticPluginRuntime.cmake                     | Selects the static MSVC runtime and a matching vcpkg triplet on Windows              |
| cmake/InstallPluginRuntime.cmake                    | Installs the plugin, collects the libraries it imports, and fixes their search paths |
| cmake/skip-rdkafka-export.cmake                     | Removes install rules from the librdkafka source that the Kafka plugin does not ship |
| .github/workflows/communication-plugins-release.yml | Builds, tests, and packages every plugin on six platforms                            |

The sections below follow the order in which a build uses these files. The runtime choice comes first, because it is made while the code compiles and applies to every dependency, and the install steps can only work on what the compile step produced. The goal of all of it is stated in the Pub/Sub Readme: the release archive holds only the plugin library, named PubSubPlugin.dll, libPubSubPlugin.so, or libPubSubPlugin.dylib, with nothing else for a client to install.

![](/uploads/cmake-packaging-cpp-plugin/image8.png)

## Why the Windows C Runtime Should Be Static

Every C++ program relies on a runtime library, which supplies the standard C and C++ functions it calls. On Windows, the compiler offers two ways to include it, and the choice is the first thing a Windows plugin build has to settle: the plugin can contain its own copy of the runtime, or it can depend on a runtime DLL that has to be present on the machine. A plugin that depends on the DLL makes every customer install the matching Visual C++ Redistributable first. According to Microsoft's [compiler option reference](https://learn.microsoft.com/en-us/cpp/build/reference/md-mt-ld-use-run-time-library), /MT makes the linker use LIBCMT.lib, a static runtime, while /MD uses the MSVCRT.lib import library and a runtime DLL. The same page adds a rule that explains why the choice has to be consistent: all modules passed to one invocation of the linker must be compiled with the same runtime library option. A plugin built with /MT that links a static library built with /MD breaks that rule.

CMake expresses the choice through CMAKE\_MSVC\_RUNTIME\_LIBRARY. Its [documentation](https://cmake.org/cmake/help/latest/variable/CMAKE_MSVC_RUNTIME_LIBRARY.html) lists MultiThreaded for -MT and MultiThreadedDLL for -MD, shows a generator expression that adds the debug variant per configuration, and states the catch that causes most failed attempts: the variable only takes effect if policy CMP0091 is set to NEW before the first project() call. Policy [CMP0091](https://cmake.org/cmake/help/latest/policy/CMP0091.html) moves the runtime flags out of the default compiler flag cache entries and into that abstraction. With the policy unset, CMake keeps the old behavior and the variable is ignored. A minimal project that selects the static runtime therefore looks like this:

```
cmake_minimum_required(VERSION 3.22)

# Before project(): use the CMP0091 abstraction and ask for the static runtime.
set(CMAKE_POLICY_DEFAULT_CMP0091 NEW)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

project(my_plugin LANGUAGES CXX)
```

The repository wraps both rules in [cmake/StaticPluginRuntime.cmake](https://github.com/grft-dev/graftcode-extensions/blob/main/communication-plugins/cmake/StaticPluginRuntime.cmake), which exposes two macros that bracket the project() call. This is how pubsub/CMakeLists.txt uses them:

```
include("${CMAKE_CURRENT_SOURCE_DIR}/../cmake/StaticPluginRuntime.cmake")
graftcode_prepare_static_runtime()

project("PubSubPlugin" VERSION 1.0.0)

graftcode_apply_static_crt()
```

The first macro runs before project(). It sets CMAKE\_POLICY\_DEFAULT\_CMP0091 to NEW, forces CMAKE\_MSVC\_RUNTIME\_LIBRARY to MultiThreaded$\<$\<CONFIG:Debug>:Debug> in the cache, and, on a Windows host with no triplet chosen, sets VCPKG\_TARGET\_TRIPLET to x64-windows-static, arm64-windows-static, or x86-windows-static, depending on the generator platform or the PROCESSOR\_ARCHITECTURE environment variable. The comment in the file explains the policy default: projects pulled in with FetchContent often pin an older cmake\_minimum\_required, which would leave CMP0091 unset and ignore the runtime variable.

The second macro runs after project(). Its comment says what it repairs: project() fills the CMAKE\_C\_FLAGS\* and CMAKE\_CXX\_FLAGS\* variables with /MD, so the macro rewrites /MDd to /MTd and /MD to /MT in each of them, which covers dependencies that do not honor CMP0091.![](/uploads/cmake-packaging-cpp-plugin/image7.png)

One part of the Windows runtime stays outside the plugin on purpose. Microsoft's [Universal CRT deployment page](https://learn.microsoft.com/en-us/cpp/windows/universal-crt-deployment) describes the Universal CRT as an operating system component, included in Windows 10 and later and Windows Server 2016 and later. The helper's comment says the same thing in its own words: /MT links the vcruntime and the C++ standard library into the plugin, so loading it does not require the VC++ Redistributable, while ucrtbase.dll stays a Windows system DLL. The plugin's own code now uses the static runtime, but the libraries it depends on are built separately, and they need the same setting.

## Choosing a vcpkg Triplet That Matches the Runtime

Most of the Pub/Sub plugin's dependencies come from vcpkg, a C++ package manager that builds each library from source for a named target called a triplet. Microsoft's [triplet documentation](https://learn.microsoft.com/en-us/vcpkg/users/triplets) names VCPKG\_CRT\_LINKAGE and VCPKG\_LIBRARY\_LINKAGE as the settings that decide runtime and library linkage for a triplet, each accepting dynamic or static. With the default Windows triplet, libcurl and OpenSSL would be built against whatever runtime that triplet specifies, which has to match the plugin's /MT or the linker rule from the previous section fails.

The \*-windows-static triplets that the helper selects are the ones the repository relies on to build ports as static libraries with the same static runtime, according to the comment in the helper file. The same comment adds that Linux and macOS vcpkg triplets are already static, with glibc, libstdc++, and libSystem staying on the host. That difference is why the triplet logic only appears in the Windows branch of the helper. For the Pub/Sub plugin, the [build notes](https://github.com/grft-dev/graftcode-extensions/blob/main/communication-plugins/pubsub/BUILD_NOTES.md) state the result: curl, OpenSSL, and zlib are linked into PubSubPlugin.dll, curl still uses Schannel so certificate checks use the Windows store, and the DLL does not import vcruntime140, libcurl, libcrypto, or libssl.

## How a Plugin Project File Is Structured

Every plugin in the repository has the same top-level shape. The Pub/Sub project file, [pubsub/CMakeLists.txt](https://github.com/grft-dev/graftcode-extensions/blob/main/communication-plugins/pubsub/CMakeLists.txt), is short enough to read in full. It runs inside a checkout of the repository, because it includes the helpers by a relative path:

```
set(CMAKE_MIN 3.22)
cmake_minimum_required(VERSION ${CMAKE_MIN})
set(CMAKE_POLICY_VERSION_MINIMUM ${CMAKE_MIN})
cmake_policy(VERSION ${CMAKE_MIN})

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_POLICY_DEFAULT_CMP0135 NEW)

include("${CMAKE_CURRENT_SOURCE_DIR}/../cmake/StaticPluginRuntime.cmake")
graftcode_prepare_static_runtime()

project("PubSubPlugin" VERSION 1.0.0)

graftcode_apply_static_crt()

add_definitions(-DUNICODE)
enable_testing()

if(MSVC)
    add_compile_options(/utf-8)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
endif()

add_subdirectory("PubSubPlugin")
add_subdirectory("PubSubPluginTest")

include("${CMAKE_CURRENT_SOURCE_DIR}/../cmake/InstallPluginRuntime.cmake")
graftcode_install_plugin(PubSubPlugin)
```

Three details are worth reading closely. The minimum version is 3.22, held in one variable that feeds both cmake\_minimum\_required and CMAKE\_POLICY\_VERSION\_MINIMUM, so the two values cannot drift apart. Position-independent code is switched on globally, because the Kafka plugin's CMake file notes that distribution archives of zlib and zstd are not built that way and so cannot be linked into a shared plugin. And the file ends with the two lines that every plugin shares: the include of InstallPluginRuntime.cmake and a call to graftcode\_install\_plugin.

The library target itself is in PubSubPlugin/CMakeLists.txt. It declares a SHARED library from five source files, finds curl, OpenSSL, nlohmann/json, and Threads, and adds one conditional that is easy to overlook. This excerpt comes from that file, where target\_name is the variable holding the library target:

```
get_target_property(_curl_type CURL::libcurl TYPE)
if(_curl_type STREQUAL "STATIC_LIBRARY")
    target_compile_definitions(${target_name} PRIVATE CURL_STATICLIB)
endif()
```

The conditional exists because a static libcurl needs the CURL\_STATICLIB definition, or the Windows compiler decorates its symbols for import from a DLL and the link fails. The plugin does not assume which kind of libcurl it receives. It asks the imported target, which lets the same source file build against a system libcurl on a developer's Linux machine and against the static vcpkg port in CI.

## Where Each Plugin Gets Its Dependencies

Dependencies arrive in different ways across the plugins, and the differences show how much freedom the helpers leave:

| Plugin            | Declared dependencies                                               | How they arrive                                                                                                               |
| :---------------- | :------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------- |
| Pub/Sub           | curl, openssl, nlohmann-json                                        | vcpkg manifest, with the toolchain file on every platform in CI                                                               |
| Amazon SQS        | aws-sdk-cpp (SQS feature only, default features off), nlohmann-json | vcpkg manifest                                                                                                                |
| Azure Service Bus | azure-core-amqp-cpp, nlohmann-json                                  | vcpkg manifest                                                                                                                |
| Kafka             | openssl on Windows only                                             | vcpkg for Windows; apt or Homebrew OpenSSL on Linux and macOS; librdkafka, zlib, zstd, and nlohmann/json through FetchContent |

The Kafka row shows the other end of the range. Its CMake file builds zlib and zstd from source with position-independent code, and it patches librdkafka's own install rules with a script, [cmake/skip-rdkafka-export.cmake](https://github.com/grft-dev/graftcode-extensions/blob/main/communication-plugins/cmake/skip-rdkafka-export.cmake), that removes an install(EXPORT) block pulling a static zlib into an export set the plugin does not ship. Whatever the source of the dependencies, every plugin ends with the same call to graftcode\_install\_plugin, which the next section explains.

## Installing the Plugin Together with Its Runtime Dependencies

The install helper, [cmake/InstallPluginRuntime.cmake](https://github.com/grft-dev/graftcode-extensions/blob/main/communication-plugins/cmake/InstallPluginRuntime.cmake), builds on a CMake feature added in version 3.21. According to the [install command reference](https://cmake.org/cmake/help/latest/command/install.html), the RUNTIME\_DEPENDENCY\_SET option on install(TARGETS) adds all runtime dependencies of the installed shared library to a named set, and a separate install(RUNTIME\_DEPENDENCY\_SET) call installs that set. The sub-arguments of the second call, including DIRECTORIES, PRE\_EXCLUDE\_REGEXES, and POST\_EXCLUDE\_REGEXES, are forwarded to file(GET\_RUNTIME\_DEPENDENCIES).

Both install rules in the helper carry a component name, plugin-runtime. The documentation states that during a component-specific installation only the rules tied to the named component run, and the release workflow installs with --component plugin-runtime. Other install rules in the same build, such as the headers and static archives that FetchContent dependencies register, never reach the archive. The destination . puts everything in one flat directory, which is the layout a client extracts next to the Gateway.

The second rule is where the scan happens. The scan has to be told where to look for each imported library, and the helper assembles that list of directories from the plugin's output directory, the vcpkg installed tree, and OpenSSL locations. For vcpkg, it uses debug/bin and debug/lib for the Debug configuration and bin and lib otherwise. For OpenSSL, it reads the parent directories of the LIB\_EAY and SSL\_EAY cache entries, then checks a short list of familiar locations: C:/Program Files/OpenSSL/bin, C:/Program Files/OpenSSL-Win64/bin, and the Homebrew prefixes for openssl and openssl\@3. Finding libraries is half the job. The other half is deciding which ones stay behind.

![](/uploads/cmake-packaging-cpp-plugin/image6.png)

## Deciding Which Libraries Stay on the Host

Some libraries must never be bundled. The C library, the C++ standard library, and the dynamic loader belong to the host operating system, and a copy from the build machine can conflict with the one on the target. The helper answers with two lists of regular expressions:

| List         | Pattern                                                                                               | Why it is excluded                                         |
| :----------- | :---------------------------------------------------------------------------------------------------- | :--------------------------------------------------------- |
| Pre-exclude  | api-ms-win-.\*, ext-ms-.\*                                                                            | Windows API set forwarders                                 |
| Pre-exclude  | libc.so, libm.so, libdl.so, libpthread.so, librt.so, libgcc\_s.so, libstdc++.so, ld-linux, linux-vdso | Linux C and C++ runtime and loader, which stay on the host |
| Pre-exclude  | libSystem., libc++., libc++abi.                                                                       | macOS system libraries                                     |
| Post-exclude | System32, SysWOW64, WinSxS, C:\Windows, Windows Kits, Microsoft Visual Studio                         | Windows system and toolchain directories                   |
| Post-exclude | /System/                                                                                              | macOS system tree                                          |

A comment in the helper explains the matching rule for the Linux entries: they are matched against the library's SONAME, so libssl and libcrypto under /usr/lib or Homebrew are still installed even though they live in system-looking directories. A blanket rule for system directories would exclude OpenSSL on a distribution that installs it there, and a plugin built that way would load only on machines with the same OpenSSL version. The standalone project later in this piece shows the effect of the list on a real install.

## Setting Library Search Paths After Installation

A copied library is only useful if the loader can find it from its new location. The helper handles this in two layers. First, it sets the plugin's install rpath: $ORIGIN on Linux and @loader\_path on Apple platforms. CMake's [INSTALL\_RPATH documentation](https://cmake.org/cmake/help/latest/prop_tgt/INSTALL_RPATH.html) describes the property as the rpath used by installed targets, notes that ${ORIGIN} is escaped in the generated install script so CMake does not read it as its own variable, and explains that targets are edited or relinked at install time to apply it. In a flat folder, the value tells the loader to look in the directory that holds the library itself.

The second layer covers the bundled copies, which keep whatever load path they had when CMake copied them. On Linux the helper runs [patchelf](https://github.com/NixOS/patchelf), a utility for modifying existing ELF executables and libraries, over every .so and .so.\* file in the install prefix, skipping symbolic links, with --set-rpath "\\$ORIGIN". The comment gives the reason a plain CMake call is not enough: file(RPATH\_SET) only rewrites an existing RPATH or RUNPATH, and the system OpenSSL and libsasl have none, so patchelf has to add one. The goal, in the comment's words, is that libssl finds libcrypto beside it on a machine that does not have that OpenSSL. The install code stops with an error message if patchelf is not on the path, which is why the Linux entries in the CI package lists include it.

macOS needs more steps, and the helper performs them in order for each .dylib in the prefix:

1. Set the library's install name to @loader\_path/\<file name> with install\_name\_tool -id.
2. List its dependencies with otool -L, and for each dependency whose file name matches another bundled library, rewrite the reference to @loader\_path/\<file name> with install\_name\_tool -change.
3. If either rewrite fails, remove the existing signature with codesign --remove-signature and retry.
4. Re-sign the library with an ad hoc signature, using codesign --force --sign -, since editing a Mach-O file invalidates its signature.

![](/uploads/cmake-packaging-cpp-plugin/image1.png)

Windows has no rpath at all. The helper's comment describes the install as flat, with consumers extracting the archive next to the host executable, so the install step needs nothing more there. The next section puts the pieces together in a project small enough to run on a laptop.

## Trying the Install Pattern in a Small Standalone Project

The project below reproduces the pattern without any repository helpers: a plugin that imports a third-party shared library, an install step that bundles the library, and a rpath that finds it. It targets Linux and macOS, since the host program uses dlopen. It needs CMake 3.22 or newer and a C++17 compiler. Create this layout:

```
hello-plugin/
  greeter/CMakeLists.txt
  greeter/greeter.h
  greeter/greeter.cpp
  hello/CMakeLists.txt
  hello/plugin.cpp
  hello/host.cpp
```

The greeter project stands in for a third-party library such as libcurl. It is built and installed on its own, so the plugin finds it the way it would find a vcpkg or Homebrew package.

```
# greeter/CMakeLists.txt
cmake_minimum_required(VERSION 3.22)
project(greeter LANGUAGES CXX)

add_library(greeter SHARED greeter.cpp)
install(TARGETS greeter LIBRARY DESTINATION lib)
install(FILES greeter.h DESTINATION include)
// greeter/greeter.h
#pragma once
const char* greeting();
// greeter/greeter.cpp
#include "greeter.h"

const char* greeting() { return "hello from the bundled dependency"; }
```

The plugin calls into greeter and uses std::string, so it imports both a third-party library and the C++ runtime. The host program loads whatever plugin file it is given.

```
// hello/plugin.cpp
#include "greeter.h"

#include <string>

extern "C" const char* plugin_message()
{
    static const std::string message = std::string("plugin says: ") + greeting();
    return message.c_str();
}
// hello/host.cpp
#include <dlfcn.h>
#include <cstdio>

int main(int argc, char** argv)
{
    if (argc < 2) {
        std::puts("usage: host <path to plugin library>");
        return 2;
    }
    void* lib = dlopen(argv[1], RTLD_NOW);
    if (lib == nullptr) {
        std::printf("load failed: %s\n", dlerror());
        return 1;
    }
    using Fn = const char* (*)();
    auto fn = reinterpret_cast<Fn>(dlsym(lib, "plugin_message"));
    if (fn == nullptr) {
        std::puts("plugin_message not found");
        return 1;
    }
    std::puts(fn());
    dlclose(lib);
    return 0;
}
```

The plugin's project file holds the three build jobs from earlier: find the dependency, set the install rpath, and install the plugin with its dependency set.

```
# hello/CMakeLists.txt
cmake_minimum_required(VERSION 3.22)
project(hello_plugin LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Find the dependency the way a vcpkg or Homebrew package would be found.
find_library(GREETER_LIB greeter REQUIRED)
find_path(GREETER_INCLUDE greeter.h REQUIRED)
get_filename_component(GREETER_DIR "${GREETER_LIB}" DIRECTORY)

# The plugin: a shared library that the host loads by file name.
add_library(hello_plugin SHARED plugin.cpp)
target_include_directories(hello_plugin PRIVATE "${GREETER_INCLUDE}")
target_link_libraries(hello_plugin PRIVATE "${GREETER_LIB}")

# The host program that loads the plugin at runtime.
add_executable(host host.cpp)
target_link_libraries(host PRIVATE ${CMAKE_DL_LIBS})

# After install, look for dependencies in the folder that holds the plugin.
if(APPLE)
  set(plugin_origin "@loader_path")
else()
  set(plugin_origin "$ORIGIN")
endif()
set_target_properties(hello_plugin PROPERTIES INSTALL_RPATH "${plugin_origin}")

# Install the plugin and record the libraries it imports.
install(TARGETS hello_plugin
  RUNTIME_DEPENDENCY_SET plugin_deps
  LIBRARY DESTINATION . COMPONENT plugin-runtime
  RUNTIME DESTINATION . COMPONENT plugin-runtime)

# Copy the imports that the host operating system does not provide.
install(RUNTIME_DEPENDENCY_SET plugin_deps
  DIRECTORIES "${GREETER_DIR}"
  PRE_EXCLUDE_REGEXES
    [[libc\.so.*]] [[libm\.so.*]] [[libgcc_s\.so.*]] [[libstdc\+\+\.so.*]]
    [[ld-linux.*]] [[linux-vdso.*]] [[libSystem\..*]] [[libc\+\+\..*]]
  LIBRARY DESTINATION . COMPONENT plugin-runtime
  RUNTIME DESTINATION . COMPONENT plugin-runtime)
```

Run these commands from the hello-plugin folder. On macOS, use .dylib in place of .so in the file names:

```
# Build and install the stand-in dependency into ./deps
cmake -S greeter -B greeter/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="$PWD/deps"
cmake --build greeter/build
cmake --install greeter/build

# Build the plugin and the host, then run the plugin from the build tree
cmake -S hello -B hello/build -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="$PWD/deps"
cmake --build hello/build
./hello/build/host ./hello/build/libhello_plugin.so

# Install the runtime component into ./package and list the result
cmake --install hello/build --component plugin-runtime --prefix "$PWD/package"
ls package

# Load the installed plugin with an empty environment
env -i ./hello/build/host ./package/libhello_plugin.so
```

The two runs of the host print plugin says: hello from the bundled dependency. On Ubuntu, ls package lists exactly two files: libgreeter.so and libhello\_plugin.so. The plugin imports libgreeter.so, libstdc++.so.6, libgcc\_s.so.1, and libc.so.6, and the exclusion list keeps the last three out of the folder, which is the behavior the previous section describes. On Linux, readelf -d package/libhello\_plugin.so | grep -E "NEEDED|RUNPATH" shows those four imports and a runpath of $ORIGIN.

Two quick experiments show what each piece does. Copy only libhello\_plugin.so into an empty folder and run the host against it: the load fails with libgreeter.so: cannot open shared object file, because the bundled dependency stayed behind. Then delete the PRE\_EXCLUDE\_REGEXES block from hello/CMakeLists.txt, rebuild, and install again into a new prefix. On an Ubuntu x86\_64 machine the folder then also holds ld-linux-x86-64.so.2, libc.so.6, libgcc\_s.so.1, libm.so.6, and two libstdc++ files, which is a folder of C and C++ runtime copies that would conflict with the target's own. The exact file names differ by distribution. With the install pattern shown on a small scale, the next section turns to how the Gateway uses the resulting folder.

## How the Flat Archive Reaches Gateway Operators

Everything above exists so that a person running the Graftcode Gateway never has to think about it. The repository's README describes a plugin as a way to carry Graft calls over an external channel instead of the Gateway's built-in servers, and states how a release archive is used: put gg, the hosted module, and the plugin library in one folder, then start the Gateway with --config and a JSON file that describes the plugin. To get gg, install it from a terminal (on Windows, iwr [https://grft.dev/get](https://grft.dev/get) | iex; on Linux and macOS, curl -fsSL [https://grft.dev/get](https://grft.dev/get) | sh), use the Docker image, or download a binary from the [Gateway releases page](https://github.com/grft-dev/graftcode-gateway/releases). With Docker, copy the plugin archive into the image next to the config file, as the communication-plugin-samples/\*/server/Dockerfile files in the repository do. The Gateway still runs the hosted module and exposes its public methods, and the Graft is still the typed interface in the calling service. The plugin changes only the channel between them, which keeps a team's existing broker, queues, and IAM setup in place. The calling service loads the same plugin library through its own config (configurations, then the package name, then plugin), so both sides need the plugin library and a matching config. The README shows this for a .NET client.

![](/uploads/cmake-packaging-cpp-plugin/image3.png)

The archive layout is the visible result of the build work. On Windows the archive holds the plugin DLL alone. The static runtime removes the Visual C++ Redistributable requirement, and curl, OpenSSL, and zlib are linked in. On Linux and macOS the archive holds the plugin .so or .dylib alone, with glibc, libstdc++, libc++, and libSystem staying on the host. The README states that a client does not install OpenSSL, curl, zlib, the AWS SDK, the Azure SDK, or the VC++ Redistributable to load any of these plugins.

One documented exception shows the limits of static linking. The Linux Kafka archive also contains libsasl2.so.2 and libsasl2.so.2.0.25, and the Readme says why: Cyrus SASL loads its mechanism modules at runtime, so that library stays shared, and both files have to sit next to libKafkaPlugin.so. The CMake design makes this case routine instead of special. The Kafka plugin uses the same install helper, and the helper's header comment lists librdkafka's libsasl among the libraries the scan installs, because it is not on the exclusion list. The patchelf step then points it at its own folder. A plugin author who has to ship a shared dependency can do it with the same helper and a documented note in the Readme.

## Running the Plugin's Tests Before It Ships

Each plugin has a test project that builds two executables. The Pub/Sub test CMake file fetches GoogleTest v1.17.0 through FetchContent and builds PubSubPluginTests from the plugin sources directly, so unit tests can reach internal functions without exporting them from the shared library. A second executable, PubSubPluginSmoke, links the real plugin target and runs as a CTest test with a 60-second timeout. It calls the plugin's exported entry points: it creates a server and a transport from a sample configuration and fails the test if either comes back null.

Test discovery uses gtest\_discover\_tests(PubSubPluginTests DISCOVERY\_TIMEOUT 60). The [GoogleTest module documentation](https://cmake.org/cmake/help/latest/module/GoogleTest.html) explains why the explicit timeout is there: discovery runs the test executable with --gtest\_list\_tests after the build, and the default discovery timeout is 5 seconds, after which discovery and the build fail. A test executable that takes longer than that to list its tests would fail the build before any test ran, so the project sets the timeout to 60 seconds.

The repository's Readme states which tests need a service. The hermetic tests, covering invalid configuration, base64 handling, JWT signing, and the transport factory, always run. A round-trip test against Pub/Sub runs only when PUBSUB\_EMULATOR\_HOST or PUBSUB\_LIVE is set, so a laptop build with no emulator still passes.

## Copying Runtime DLLs Next to Windows Test Executables

Windows adds a problem that Linux and macOS solve with an rpath. The test executables need the plugin and any DLLs it imports to sit next to them, or the loader raises a missing-DLL dialog on a CI runner. The test CMake file adds a post-build step for each test target on Windows, passing the target's runtime DLLs to a small script, copy\_runtime\_dlls.cmake, joined with |, which the script splits back into a list. The script creates the destination folder, then copies each DLL with copy\_if\_different, and stops with an error if a listed DLL does not exist. Tests answer whether the plugin works on the machine that built it. Whether the same files arrive intact in an archive is the job of the release workflow.

## How the Release Workflow Packages Each Build

The workflow in [.github/workflows/communication-plugins-release.yml](https://github.com/grft-dev/graftcode-extensions/blob/main/.github/workflows/communication-plugins-release.yml) runs a matrix of five plugins, RabbitMQ, Service Bus, Kafka, SQS, and Pub/Sub, across six runners: windows-latest, windows-11-arm, ubuntu-22.04, ubuntu-22.04-arm, macos-26-intel, and macos-14. That is thirty build jobs, with fail-fast set to false so one failing platform does not cancel the other twenty-nine. The workflow triggers on pushes to main, tags that match v\*, pull requests, and manual dispatch, and its path filter covers communication-plugins/\*\*, which includes the cmake/ helpers, so a change to a helper file rebuilds every plugin.

Each job resolves its metadata first, setting the package suffix from the runner (windows-x86\_64, windows-arm64, linux-x86\_64, linux-arm64, macos-x86\_64, macos-arm64) and the configure arguments from the plugin. Pub/Sub, SQS, and Service Bus pass the vcpkg toolchain file from the runner's VCPKG\_INSTALLATION\_ROOT. Kafka passes it on Windows only, and installs libssl-dev and libsasl2-dev from apt on Linux, or OpenSSL from Homebrew on macOS. Every Linux package list in the workflow includes patchelf.

For Pub/Sub on Linux, the build, test, and install steps are equivalent to these commands, run from the repository root, with VCPKG\_INSTALLATION\_ROOT pointing at a vcpkg checkout:

```
cd communication-plugins/pubsub
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE="$VCPKG_INSTALLATION_ROOT/scripts/buildsystems/vcpkg.cmake"
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure --timeout 180
cmake --install build --config Release --prefix "$PWD/../package-staging" --component plugin-runtime
```

The packaging step that follows is where the repository guards against an empty or incomplete archive. It copies every .dll, .dylib, .so, and .so.\* file in the staging folder into a flat folder, and then checks two conditions. If the folder holds no shared libraries, the job exits with an error and prints the staging tree. If none of the expected plugin file names is present, it exits with a different error that names the missing library. Only then does it create \<plugin>-\<suffix>.tar.gz and upload it as an artifact with a 30-day retention.

![](/uploads/cmake-packaging-cpp-plugin/image5.png)

## Requiring Manual Approval Before a Release Is Published

Publishing is separated from building. A job named approve\_release references a GitHub environment named release and runs only for events other than pull requests. GitHub's [environment documentation](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment) states that a job referencing an environment must follow the environment's protection rules before it runs, and that required reviewers are one optional rule. The final release job needs that approval, downloads every artifact, and creates the GitHub release with all thirty archives attached. A pull request therefore builds and tests all thirty combinations but can never publish.

The tag for the release comes from the event that started the run. A manual dispatch uses the release\_tag input, a pushed v\* tag uses its own name, and a plain push to main produces a tag of the form build-\<short sha>. Each case produces a release whose archives follow the same \<plugin>-\<suffix>.tar.gz naming, so a client can predict the file name for any plugin and platform.

## Troubleshooting Failures in Builds That Reuse the Helpers

Most failures trace back to one of four conditions, and each has a clear signal in the files above.

* **The Windows plugin still imports a Redistributable DLL.** Check that graftcode\_prepare\_static\_runtime() runs before project() and graftcode\_apply\_static\_crt() runs after it. If either macro is missing, CMP0091 or the flag rewrite is skipped, and a dependency can compile against /MD. List the DLL's imports with a PE dump tool and compare them with the build notes' list for Pub/Sub, which names vcruntime140, libcurl, libcrypto, and libssl as absent.
* **Linux install stops with a patchelf error.** The install code raises a fatal error when patchelf is not found, with a message that it is required to set $ORIGIN on bundled shared libraries. Install it before running cmake --install, as the CI package lists do.
* **The archive step reports no shared libraries.** The install ran with the wrong component, or graftcode\_install\_plugin was never called. The workflow's check prints the staging tree, which shows what was installed. Confirm the project file ends with the include of InstallPluginRuntime.cmake and the call with the target name, and note that the helper stops with an error if the target does not exist.
* **A Windows test fails on a missing DLL dialog.** The post-build copy step is attached per test target. A new test executable needs its own add\_custom\_command that passes its runtime DLLs to copy\_runtime\_dlls.cmake.

A fifth source of trouble is the vcpkg baseline. The Pub/Sub, SQS, and Kafka manifests set a builtin-baseline hash, and Microsoft's [versioning documentation](https://learn.microsoft.com/en-us/vcpkg/users/versioning) describes it as the baseline for the built-in registry: a global version floor that sets the default version for ports. The Service Bus manifest in the repository has no baseline line, so its port versions follow the classic mode algorithm described on the same page, and a change in the vcpkg checkout can change the versions it builds against.

## Where Should You Start with CMake Plugin Packaging

A plugin that loads cleanly on a machine you do not control needs three decisions made at build time. The runtime has to be static where the platform allows it, which on Windows takes the policy, the runtime variable, and a matching triplet together. The remaining imports have to be found by a scan with an exclusion list that spares OpenSSL and excludes the C and C++ runtime. And the copied libraries need an rpath that points at their own folder, applied by patchelf on Linux and install\_name\_tool on macOS. The graftcode-extensions repository puts those decisions in a few helper files and one workflow, which is why five plugins share one packaging path.

To try it, run the standalone project above, then compare it with the repository. Clone the [graftcode-extensions repository](https://github.com/grft-dev/graftcode-extensions), build the Pub/Sub plugin with the steps in its Readme, run cmake --install with the plugin-runtime component and a --prefix pointing at an empty folder, and list what lands there. Compare that listing with the release archive for the same platform. The [Graftcode Academy](https://academy.graftcode.com/) covers Gateway setup. The plugin config fields are documented in the [graftcode-extensions README](https://github.com/grft-dev/graftcode-extensions/blob/main/README.md) and in the Plugin server config section of the [Gateway README](https://github.com/grft-dev/graftcode-gateway#plugin-server-config).

## FAQ

### Why does the plugin use /MT instead of shipping the Visual C++ Redistributable?

A static runtime removes a prerequisite from every machine that loads the plugin. The repository's helper links the vcruntime and the C++ standard library into the DLL, and leaves only the Universal CRT, which Microsoft's deployment documentation describes as part of Windows 10 and later. The trade-off is the rule that every static dependency must use the same runtime, which the triplet and the flag rewrite enforce.

### Do I need vcpkg to build one of the plugins?

The documented Pub/Sub build uses the vcpkg toolchain file, and the CI passes it for Pub/Sub, SQS, and Service Bus on every platform. Kafka differs: on Linux and macOS its build uses the system or Homebrew OpenSSL and fetches librdkafka, zlib, and zstd with FetchContent, and only the Windows build uses vcpkg. Check the plugin's own Readme before choosing a toolchain.

### Why does the Linux install need patchelf when CMake sets an install rpath?

CMake's install rpath applies to the plugin target. The bundled copies of other libraries keep their original load paths, and the helper's comment notes that file(RPATH\_SET) only rewrites an existing RPATH or RUNPATH, while system OpenSSL and libsasl have none. patchelf --set-rpath adds $ORIGIN to each copy so they find each other in the flat folder.

### What stops an incomplete archive from reaching a release?

The packaging step fails the job when the flattened folder holds no shared libraries, and fails again when the expected plugin file name is missing. A separate approval job that references the release environment sits between the builds and the publish job, and pull requests skip it, so they build and test without publishing.

### Can I reuse the helper files for a plugin outside the repository?

The helpers are plain CMake files that take a target name, so another project can include them and call graftcode\_install\_plugin(\<target>) after defining a shared library. The plugin projects include them through a relative path to the sibling cmake/ folder, so copy the folder or adjust the include path. The install helper requires CMake 3.21 or newer for runtime dependency sets and, on Linux, a patchelf executable.
