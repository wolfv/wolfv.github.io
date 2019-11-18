---
layout: post
title: "Conda for C++ packages"
author: "Wolf Vollprecht"
summary: "Conda is often thought as a Python package manager, but it works great for C++, too! Here is how:"
categories: posts
---

<div class="subtitle">There exist a multitude of package management solutions
for C++ but none is dominant In this post we're going to argue why we think
conda is the package manager for the future! </div>

### What makes conda strong

Conda is different from your average package manager. It's different in the
following ways:

- It's truly cross-platform: conda works on Windows (!), OS X and Linux. That
  makes it very different from apt, brew, dnf, yum, chocolatey ...
- Conda supports environments natively. You can have multiple environments with
  different versions of some library side-by-side -- and switch between them
  conveniently using `conda activate my_environment`.
- Conda supports distributing binaries. A lot of C++ libraries are difficult to
  build, or take a long time to build. Therefore shipping binaries (on all
  platforms) is a huge feature. You can find big libs such as boost on conda.
- Conda is language-agnostic: you have cargo for Rust, pip for Python and Pkg
  for Julia, Conan for C++ ... where will it stop?
- Contrary to the naive attempt, conda does true dependency solving -- conda
  uses a SAT solver to find a solution to the dependency problem if it exists.

And last but not least, conda has already found widespread use in many
communities. The Python data science landscape and the R community are some of
the heavy conda users out there. One of the largest community engineering
efforts out there is probably the maintenance of conda-forge, a big repository
of Open Source software that's built and maintained completely in the open.

That's why we (at QuantStack) think that conda is the package manager of the
future -- not just for Python or R, but also for C++. In this blog post we'll
show how convenient it is to use conda environments as a C++ user.

### Installing conda

The best way to get a minimal conda installation is using miniconda:
<https://docs.conda.io/en/latest/miniconda.html>. Once installed you can install
new packages using

`$ conda install xtensor xsimd -c conda-forge` <no-indent></no-indent> which
will download the package and install it in the currently activated environemnt
(the default environment is called "base"). The environment (on Linux) mirrors a
standard unix root-environment (with all the `/etc`, `/include`, `/lib`...
folders).

### Installing the conda C++ toolchain

For compatibility reasons, it's advisable to install the conda toolchain for compiling 
(instead of using the system compilers, for example).

To use the [conda compilers](https://docs.conda.io/projects/conda-build/en/latest/resources/compiler-tools.html#compiler-packages) you need
to install

```sh
# Linux
conda install gcc_linux-64 gxx_linux-64 # (optionally) gfortran_linux-64
# OS X
conda install clang_osx-64 clangxx_osx-64 # (optionally) gfortran_osx-64
# Windows
conda install vs2015_win-64 # or the newer vs2017_win-64
```

Note that on OS X you also need to have a [SDK available](https://docs.conda.io/projects/conda-build/en/latest/resources/compiler-tools.html#compiler-packages).

With those packages installed, environment variables such as `$CXX` should 
automatically point to the correct C++ compiler, the same one used to compile
the libraries downloaded from conda/conda-forge.
The compiler activation script also sets the CXXFLAGS variable to come
with the same set of standard C++ flags.[^1]


### Using the C++ library installed with conda

Using the installed C++ libraries is actually surprisingly easy -- given one
uses CMake:

```sh 
cd your_project 
mkdir build && cd build
# make sure your environemnt is activated!
conda activate {base / myenv}
cmake .. -DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX
# now build ...
make
```

Now CMake is going to prioritize the libraries and CMake modules found in the
conda environment.

If you are a Windows user, then the command will look slightly different:

```sh
cd your_project
mkdir build
cd build
# make sure your environemnt is activated!
conda activate {base / myenv}
cmake -G "NMake Makefiles" -DCMAKE_INSTALL_PREFIX=%MINICONDA%\\Library -DBUILD_TESTS=ON ..
# now build ...
nmake
```

And that's it, this should show roughly how conda can be used with CMake for C or C++ projects.

[^1]: All the environment variables that are set can be found in these activation
	  scripts: [Link](https://github.com/conda-forge/ctng-compiler-activation-feedstock/tree/master/recipe)