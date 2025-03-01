[![Build Status](https://travis-ci.com/TheLartians/CPM.cmake.svg?branch=master)](https://travis-ci.com/TheLartians/CPM.cmake)
[![Actions Status](https://github.com/TheLartians/CPM.cmake/workflows/MacOS/badge.svg)](https://github.com/TheLartians/CPM.cmake/actions)
[![Actions Status](https://github.com/TheLartians/CPM.cmake/workflows/Windows/badge.svg)](https://github.com/TheLartians/CPM.cmake/actions)
[![Actions Status](https://github.com/TheLartians/CPM.cmake/workflows/Ubuntu/badge.svg)](https://github.com/TheLartians/CPM.cmake/actions) [![Join the chat at https://gitter.im/TheLartians/CPM.cmake](https://badges.gitter.im/TheLartians/CPM.cmake.svg)](https://gitter.im/TheLartians/CPM.cmake?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<p align="center">
  <img src="./logo/CPM.png" height="100" />
</p>

# Setup-free CMake dependency management

CPM.cmake is a CMake script that adds dependency management capabilities to CMake.
It's built as a thin wrapper around CMake's [FetchContent](https://cmake.org/cmake/help/latest/module/FetchContent.html) module that adds version control, caching, a simple API [and more](#comparison-to-pure-fetchcontent--externalproject).

## Manage everything

Any downloadable project or resource can be added as a version-controlled dependency though CPM, it is not necessary to modify or package anything.
Projects using modern CMake are automatically configured and their targets can be used immediately.
For everything else, the targets can be created manually after the dependency has been downloaded (see the [snippets](#snippets) below for examples).

## Usage

After `CPM.cmake` has been [added](#adding-cpm) to your project, the function `CPMAddPackage` or `CPMFindPackage` can be used to fetch and configure a dependency.
Afterwards, any targets defined in the dependency can be used directly.
`CPMFindPackage` and `CPMAddPackage` take the following named parameters.

```cmake
CPMAddPackage(
  NAME          # The unique name of the dependency (should be the exported target's name)
  VERSION       # The minimum version of the dependency (optional, defaults to 0)
  OPTIONS       # Configuration options passed to the dependency (optional)
  DOWNLOAD_ONLY # If set, the project is downloaded, but not configured (optional)
  [...]         # Origin parameters forwarded to FetchContent_Declare, see below
)
```

The origin may be specified by a `GIT_REPOSITORY`, but other sources, such as direct URLs, are [also supported](https://cmake.org/cmake/help/v3.11/module/ExternalProject.html#external-project-definition).
If `GIT_TAG` hasn't been explicitly specified it defaults to `v(VERSION)`, a common convention for git projects.
On the other hand, if `VERSION` hasn't been explicitly specified, CPM can automatically identify the version from the git tag in some common cases. 
`GIT_TAG` can also be set to a specific commit or a branch name such as `master` to always download the most recent version.
The optional argument `FIND_PACKAGE_ARGUMENTS` can be specified to a string of parameters that will be passed to `find_package` if enabled (see below).

After calling `CPMAddPackage` or `CPMFindPackage`, the following variables are defined in the local scope, where `<dependency>` is the name of the dependency.

- `<dependency>_SOURCE_DIR` is the path to the source of the dependency.
- `<dependency>_BINARY_DIR` is the path to the build directory of the dependency.
- `<dependency>_ADDED` is set to `YES` if the dependency has not been added before, otherwise it is set to `NO`.

The difference between `CPMFindPackage` and `CPMAddPackage` is that `CPMFindPackage` will try to find a local dependency via CMake's `find_package` and fallback to `CPMAddPackage` if the dependency is not found.
This behaviour can be also modified globally via [CPM options](#options).

See [this medium article](https://medium.com/swlh/cpm-an-awesome-dependency-manager-for-c-with-cmake-3c53f4376766) for a short tutorial on using CPM.cmake.

## Full CMakeLists Example

```cmake
cmake_minimum_required(VERSION 3.14 FATAL_ERROR)

# create project
project(MyProject)

# add executable
add_executable(tests tests.cpp)

# add dependencies
include(cmake/CPM.cmake)

CPMAddPackage(
  NAME Catch2
  GITHUB_REPOSITORY catchorg/Catch2
  VERSION 2.5.0
)

# link dependencies
target_link_libraries(tests Catch2)
```

See the [examples directory](https://github.com/TheLartians/CPM.cmake/tree/master/examples) for complete examples with source code and check [below](#snippets) or in the [wiki](https://github.com/TheLartians/CPM.cmake/wiki/More-Snippets) for example snippets.

## Adding CPM

To add CPM to your current project, simply add the [latest release](https://github.com/TheLartians/CPM.cmake/releases/latest) of `CPM.cmake` or `get_cpm.cmake` to your project's `cmake` directory.
The command below will perform this automatically.

```bash
mkdir -p cmake
wget -O cmake/CPM.cmake https://github.com/TheLartians/CPM.cmake/releases/latest/download/get_cpm.cmake
```

You can also download CPM.cmake directly from your project's `CMakeLists.txt`. See the [wiki](https://github.com/TheLartians/CPM.cmake/wiki/Downloading-CPM.cmake-in-CMake) for more details.

## Updating CPM

To update CPM to the newest version, update the script in the project's root directory, for example by running the command above.
Dependencies using CPM will automatically use the updated script of the outermost project.

## Advantages

- **Small and reusable projects** CPM takes care of all project dependencies, allowing developers to focus on creating small, well-tested libraries.
- **Cross-Platform** CPM adds projects directly at the configure stage and is compatible with all CMake toolchains and generators.
- **Reproducible builds** By versioning dependencies via git commits or tags it is ensured that a project will always be buildable.
- **Recursive dependencies** Ensures that no dependency is added twice and all are added in the minimum required version.
- **Plug-and-play** No need to install anything. Just add the script to your project and you're good to go.
- **No packaging required** Simply add all external sources as a dependency.
- **Simple source distribution** CPM makes including projects with source files and dependencies easy, reducing the need for monolithic header files or git submodules.

## Limitations

- **No pre-built binaries** For every new build directory, all dependencies are initially downloaded and built from scratch. To avoid extra downloads it is recommend to set the [`CPM_SOURCE_CACHE`](#CPM_SOURCE_CACHE) environmental variable. Using a caching compiler such as [ccache](https://github.com/TheLartians/Ccache.cmake) can drastically reduce build time.
- **Dependent on good CMakeLists** Many libraries do not have CMakeLists that work well for subprojects. Luckily this is slowly changing, however, until then, some manual configuration may be required (see the snippets [below](#snippets) for examples). For best practices on preparing projects for CPM, see the [wiki](https://github.com/TheLartians/CPM.cmake/wiki/Preparing-projects-for-CPM.cmake). 
- **First version used** In diamond-shaped dependency graphs (e.g. `A` depends on `C`@1.1 and `B`, which itself depends on `C`@1.2 the first added dependency will be used (in this case `C`@1.1). In this case, B requires a newer version of `C` than `A`, so CPM will emit a warning. This can be easily resolved by adding a new version of the dependency in the outermost project, or by introducing a [package lock file](#package-lock).

For projects with more complex needs and where an extra setup step doesn't matter, it may be worth to check out an external C++ package manager such as [vcpkg](https://github.com/microsoft/vcpkg), [conan](https://conan.io) or [hunter](https://github.com/ruslo/hunter).
Dependencies added with `CPMFindPackage` should work with external package managers.
Additionally, the option [`CPM_USE_LOCAL_PACKAGES`](#cpmuselocalpackages) will enable `find_package` for all CPM dependencies.

## Comparison to FindPackage

The usual way to add libraries in CMake projects is to call `find_package(<PackageName>)` and to link against libraries defined in a `<PackageName>_LIBRARIES` variable.
While simple, this may lead to unpredictable builds, as it requires the library to be installed on the system and it is unclear which version of the library has been added.
Additionally, it is difficult to cross-compile projects (e.g. for mobile), as the dependencies will need to be rebuilt manually for each targeted architecture.

CPM.cmake allows dependencies to be unambiguously defined and builds them from source.
Note that the behaviour differs from `find_package`, as variables exported to the parent scope (such as  `<PackageName>_LIBRARIES`) will not be visible after adding a package using CPM.cmake.
The behaviour can be [achieved manually](https://github.com/TheLartians/CPM.cmake/issues/132#issuecomment-644955140), if required.

## Comparison to pure FetchContent / ExternalProject

CPM.cmake is a wrapper for CMake's FetchContent module and adds a number of features that turn it into a useful dependency manager.
The most notable features are:

- A simpler to use API
- Version checking: CPM.cmake will check the version number of any added dependency and omit a warning if another dependency requires a more recent version.
- Options: any Options passed to a dependency are stored and compared on later use, so if another dependency tries to add an existing dependency with incompatible options a warning will be emitted to the user.
- Offline builds: CPM.cmake will override CMake's download and update commands, which allows new builds to be configured while offline if all dependencies [are available locally](#cpm_source_cache).
- Automatic shallow clone: if a version tag (e.g. `v2.2.0`) is provided and `CPM_SOURCE_CACHE` is used, CPM.cmake will perform a shallow clone of the dependency, which should be significantly faster while using less storage than a full clone.
- Overridable: all `CPMAddPackage` can be configured to use `find_package` by setting a [CMake flag](#cpm_use_local_packages), making it easy to integrate into projects that may require local versioning through the system's package manager.
- [Package lock files](#package-lock) for easier transitive dependency management.
- Dependencies can be overridden [per-build](#local-package-override) using CMake CLI parameters.

ExternalProject works similarly as FetchContent, however waits with adding dependencies until build time. 
This has a quite a few disadvantages, especially as it makes using custom toolchains / cross-compiling very difficult and can lead to problems with nested dependencies.

## Options

### CPM_SOURCE_CACHE

To avoid re-downloading dependencies, CPM has an option `CPM_SOURCE_CACHE` that can be passed to CMake as `-DCPM_SOURCE_CACHE=<path to an external download directory>`.
This will also allow projects to be configured offline, as long as the dependencies have been added to the cache before.
It may also be defined system-wide as an environmental variable, e.g. by exporting `CPM_SOURCE_CACHE` in your `.bashrc` or `.bash_profile`.

```bash
export CPM_SOURCE_CACHE=$HOME/.cache/CPM
```

Note that passing the variable as a configure option to CMake will always override the value set by the environmental variable.

### CPM_DOWNLOAD_ALL

If set, CPM will forward all calls to `CPMFindPackage` as `CPMAddPackage`.
This is useful to create reproducible builds or to determine if the source parameters have all been set correctly.
This can also be set as an environmental variable.

### CPM_USE_LOCAL_PACKAGES

CPM can be configured to use `find_package` to search for locally installed dependencies first by setting the CMake option `CPM_USE_LOCAL_PACKAGES`.
If the option `CPM_LOCAL_PACKAGES_ONLY` is set, CPM will emit an error if the dependency is not found locally.
These options can also be set as environmental variables.

## Local package override

Library developers are often in the situation where they work on a locally checked out dependency at the same time as on a consumer project.
It is possible to override the consumer's dependency with the version by supplying the CMake option `CPM_<dependency name>_SOURCE` set to the absolute path of the local library.
For example, to use the local version of the dependency `Dep` at the path `/path/to/dep`, the consumer can be built with the following command. 

```bash
cmake -Bbuild -DCPM_Dep_SOURCE=/path/to/dep
```

## Package lock

In large projects with many transitive dependencies, it can be useful to introduce a package lock file.
This will list all CPM.cmake dependencies and can be used to update dependencies without modifying the original `CMakeLists.txt`.
To use a package lock, add the following line directly after including CPM.cmake.

```cmake
CPMUsePackageLock(package-lock.cmake)
```

To create or update the package lock file, build the `cpm-update-package-lock` target.

```bash
cmake -Bbuild
cmake --build build --target cpm-update-package-lock
```

See the [wiki](https://github.com/TheLartians/CPM.cmake/wiki/Package-lock) for more info.


## Built with CPM.cmake

Some amazing projects that are built using the CPM.cmake package manager. 
If you know others, feel free to add them here through a PR.

<table>
  <tr>
    <td>
      <a href="https://otto-project.github.io">
        <p align="center">
          <img src="https://avatars2.githubusercontent.com/u/40357059?s=200&v=4"  alt="otto-project" height=100pt width=100pt />
        </p>
        <p align="center"><b>OTTO - The Open Source GrooveBox</b></p>
      </a>
    </td>
    <td>
      <a href="https://maphi.app">
        <p align="center">
          <img src="https://user-images.githubusercontent.com/4437447/89892751-71af8000-dbd7-11ea-9d87-4fe107845069.png"  alt="maphi" height=100pt width=100pt />
        </p>
        <p align="center"><b>Maphi - the Math App</b></p>
      </a>
    </td>
    <td>
      <a href="https://github.com/TheLartians/ModernCppStarter">
        <p align="center">
          <img src="https://repository-images.githubusercontent.com/254842585/4dfa7580-7ffb-11ea-99d0-46b8fe2f4170"  alt="modern-cpp-starter" height=100pt width=200pt />
        </p>
        <p align="center"><b>ModernCppStarter</b></p>
      </a>
    </td>
  </tr>
</table>

## Snippets

These examples demonstrate how to include some well-known projects with CPM.
See the [wiki](https://github.com/TheLartians/CPM.cmake/wiki/More-Snippets) for more snippets.

### [Catch2](https://github.com/catchorg/Catch2)

```cmake
CPMAddPackage(
  NAME Catch2
  GITHUB_REPOSITORY catchorg/Catch2
  VERSION 2.5.0
)
```

### [Boost (via boost-cmake)](https://github.com/Orphis/boost-cmake)

```CMake
CPMAddPackage(
  NAME boost-cmake
  GITHUB_REPOSITORY Orphis/boost-cmake
  VERSION 1.67.0
)
```

### [cxxopts](https://github.com/jarro2783/cxxopts)

```cmake
CPMAddPackage(
  NAME cxxopts
  GITHUB_REPOSITORY jarro2783/cxxopts
  VERSION 2.2.0
  OPTIONS
    "CXXOPTS_BUILD_EXAMPLES Off"
    "CXXOPTS_BUILD_TESTS Off"
)
```

### [Yaml-cpp](https://github.com/jbeder/yaml-cpp)

```CMake
CPMAddPackage(
  NAME yaml-cpp
  GITHUB_REPOSITORY jbeder/yaml-cpp
  # 0.6.2 uses deprecated CMake syntax
  VERSION 0.6.3
  # 0.6.3 is not released yet, so use a recent commit
  GIT_TAG 012269756149ae99745b6dafefd415843d7420bb 
  OPTIONS
    "YAML_CPP_BUILD_TESTS Off"
    "YAML_CPP_BUILD_CONTRIB Off"
    "YAML_CPP_BUILD_TOOLS Off"
)
```

### [google/benchmark](https://github.com/google/benchmark)

```cmake
CPMAddPackage(
  NAME benchmark
  GITHUB_REPOSITORY google/benchmark
  VERSION 1.4.1
  OPTIONS
    "BENCHMARK_ENABLE_TESTING Off"
)

if (benchmark_ADDED)
  # compile with C++17
  set_target_properties(benchmark PROPERTIES CXX_STANDARD 17)
endif()
```

### [nlohmann/json](https://github.com/nlohmann/json)

```cmake
CPMAddPackage(
  NAME nlohmann_json
  VERSION 3.6.1  
  # the git repo is incredibly large, so we download the archived include directory
  URL https://github.com/nlohmann/json/releases/download/v3.6.1/include.zip
  URL_HASH SHA256=69cc88207ce91347ea530b227ff0776db82dcb8de6704e1a3d74f4841bc651cf
)

if (nlohmann_json_ADDED)
  add_library(nlohmann_json INTERFACE IMPORTED)
  target_include_directories(nlohmann_json INTERFACE ${nlohmann_json_SOURCE_DIR})
endif()
```

### [Range-v3](https://github.com/ericniebler/range-v3)

```Cmake
CPMAddPackage(
  NAME range-v3
  URL https://github.com/ericniebler/range-v3/archive/0.5.0.zip
  VERSION 0.5.0
  # the range-v3 CMakeLists screws with configuration options
  DOWNLOAD_ONLY True
)

if(range-v3_ADDED) 
  add_library(range-v3 INTERFACE IMPORTED)
  target_include_directories(range-v3 INTERFACE "${range-v3_SOURCE_DIR}/include")
endif()
```

### [Lua](https://www.lua.org)

```cmake
CPMAddPackage(
  NAME lua
  GIT_REPOSITORY https://github.com/lua/lua.git
  VERSION 5.3.5
  DOWNLOAD_ONLY YES
)

if (lua_ADDED)
  # lua has no CMake support, so we create our own target

  FILE(GLOB lua_sources ${lua_SOURCE_DIR}/*.c)
  list(REMOVE_ITEM lua_sources "${lua_SOURCE_DIR}/lua.c" "${lua_SOURCE_DIR}/luac.c")
  add_library(lua STATIC ${lua_sources})

  target_include_directories(lua
    PUBLIC
      $<BUILD_INTERFACE:${lua_SOURCE_DIR}>
  )
endif()
```

For a full example on using CPM to download and configure lua with sol2 see [here](examples/sol2).

### Full Examples

See the [examples directory](https://github.com/TheLartians/CPM.cmake/tree/master/examples) for full examples with source code and check out the [wiki](https://github.com/TheLartians/CPM.cmake/wiki/More-Snippets) for many more example snippets.
