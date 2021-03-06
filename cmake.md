# CMake Usage                                                     {#cmake_usage}
Here we briefly describe features of CMake directly relevant to Zisa.

## CMake Primer
The most simple and often practical way of using CMake is on the command line to
generate Make files. This can be done by

    $ cmake ${MANY_FLAGS} -DCMAKE_BUILD_TYPE=${BUILD_TYPE} -B ${BUILD_DIR}

which will set up everything in the folder `${BUILD_DIR}` mostly called
`build` and virtually never `.`. Nothing stops you from having several build
folders each using either different dependencies or compilers.

In order to actually compile one can type

    $ cmake --build ${BUILD_DIR} --parallel $(nproc)

Finally, it pays to check if the compilation commands correspond to something
meaningful. Which is achieved by `make VERBOSE=1`.

Equipped with this knowledge using CMake revolves around finding the correct
value for `${MANY_FLAGS}`, see [Project specific flags](@ref cmake_flags) for help
with this problem.

### Globbing source files
The convincing argument for not globbing source files is that if one switches
Git branches files might appear or vanish; and if `CMakeLists.txt` does not
change (very likely), then `cmake` won't be run again, resulting in a
misconfigured build.

Obviously, adding the files manually is unacceptable. However, nothing is
stopping us from have a script add them manually for us. This script is called
`bin/update_cmake.py`. For every subfolder it simply lists the source files
contained in that folder in a new `CMakeLists.txt` located in that subfolder. A
simple `add_subdirectory` pulls in the files.

### Deleting build folders
Personally, when facing strange issues with CMake, I think it helps to point
CMake to a fresh location, either by changing the `${BUILD_DIR}` or by deleting
it.

### `CMAKE_MODULE_PATH` and `CMAKE_PREFIX_PATH`
There are now two ways of finding dependencies. Either the library provides a
`XYZConfig.cmake` which contains all the details about the targets, i.e. the
*new way*. Alternatively, someone can write a `FindXYZ.cmake` module which
contains all the details about how to use the library. Where does CMake look to
find these things?

* `CMAKE_PREFIX_PATH` this is where CMake looks for `XYZConfig.cmake`.
* `CMAKE_MODULE_PATH` this is where CMake looks for `FindXYZ.cmake`.

Well, there and the standard OS dependent locations.

### Installing an modern version
CMake compiles and installs with relative ease. Therefore, if the cluster
doesn't support a modern version, it's easy to install a modern version
locally. The Zisa libraries contain a script which automates all steps involved.

### CUDA
As of `3.18` CUDA works reasonably well with CMake. To use it, we add the
language `CUDA` to the project. This unfortunately must also be done in every
project using a CUDA dependency. Next we find `CUDA` by

    find_package(CUDAToolkit REQUIRED)

which provides the target `CUDA::cudart` and many more. The current situation
isn't without quirks:

* When using `MPI` we found that `CMP0105` must be set to `OLD`. Since
  otherwise liker related flags are passed to the compiler incorrectly.

* Slightly more care is needed when defining `target_compile_options`.  In
  particular, flags that should be passed down to the C++/host compiler
  should be defined as
  ```
  target_compile_options(TARGET
    SCOPE
    $<BUILD_INTERFACE:$<$<COMPILE_LANGUAGE:CXX>:FLAG>>
    $<BUILD_INTERFACE:$<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=FLAG>>
  )
  ```
  where `TARGET` should be replaced by the target, `SCOPE` should be one of
  `INTERFACE`, `PRIVATE` or `PUBLIC`; and `FLAG` be replaced by the desired
  flag. Analogously, one should guard flags that are to be passed only to `nvcc`
  but not the host compiler, e.g., `--expt-relaxed-constexpr` and vice-versa.


### Configuring a file.
CMake has a templating system, to generate files with certain parts substituted
when the project is built/installed. For our purposes it pretty simple, any
CMake variable surrounded by `@` will be replaced by its value. To create the
configured file, we must call
```
configure_file(
  PATH_TO_TEMPLATE
  PATH_TO_SUBSTITUTED
  @ONLY
)
```

### Aliases
It can be very useful to refer to certain targets by two names, one with the
namespace and one without, e.g., `memory` and `Zisa::memory`. This is possible
through
```
add_library(Zisa::memory ALIAS memory)
```
This creates the symmetry required to incorporate Zisa libraries either directly
via `add_subdirectory` or as an external dependency via `find_package`.

## Packaging with CMake
In modern CMake a library is installed along with a `*Config.cmake` file which
describes the targets and their peculiarities. The definition of the targets,
etc. can be done automatically by CMake, e.g., we tell CMake to generate the
file for us called `ZisaTargets.cmake`. This is not the `ZisaConfig.cmake` that
CMake will be looking for. Let's look at that file next. It contains the line
```
include("${CMAKE_CURRENT_LIST_DIR}/ZisaTargets.cmake")
```
which includes the autogenerated file. If the library has no dependencies, we
could simply install the `ZisaConfig.cmake`. Job done.

When dependencies come into play more needs to be done. The first thing you'll
notice about this solution, is that the dependencies must be found again in
every project using our library. This feels redundant. The second issue we have
in Zisa is that we have many optional dependencies. Therefore, this system
makes it easy to forget say a `-DZISA_HAS_HDF5=1` in some project using
ZisaMemory. This results in a broken build. Therefore, there are two more
problems to be solved

1. Automatically finding external dependencies.
2. Propagating certain CMake variables.

The first problem is "solved" by repeating the logic w.r.t. `find_package` in the
`ZisaConfig.cmake` making sure to replace `find_package` with `find_dependency`
just incase someone has an optional dependency on us. The second problem is
solved by moving `ZisaConfig.cmake` to `ZisaConfig.cmake.in` and using
`configure_file` to make the relevant flags persistent. Please look at some of
the `cmake/Zisa*Config.cmake.in`. In particular, the one in `ZisaCore` is
relevant.

We're almost there. What if we have a dependency which can't be found by CMake
out of the box? Typically, we'll also distribute a `FindXYZ.cmake`. Let's make
things concrete. In Zisa this is the case for in ZisaMemory with NetCDF. We
distribute a find module for NetCDF written for VTK. Now, if anyone where to
use ZisaMemory, the `find_dependency` would fail because it can't find NetCDF,
because VTK's `FindNetCDF.cmake` isn't present. Therefore, we must also install
any `FindXYZ.cmake` that we rely on.

**Note:** This solution is currently working smoothly, but hasn't been tested
well enough to make a definite statement. To me it feels very error prone to
force the user of our library to call `find_dependency`. If you're mimicking
the strategy here, consider adding a flag which can turn off the
`find_dependency` for each of you external dependencies. If you're using Zisa
and have exactly this problem. Please report it as an issue.
