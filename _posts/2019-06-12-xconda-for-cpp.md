---
layout: post
title: "Conda for C++ packages"
author: "Wolf Vollprecht"
categories: literature
---

## There exist a multitude of package management solutions for C++ but none is dominant

In this post we're going to argue why we think conda is the package manager for the
future! 

### What makes conda strong

Conda is different from your average package manager. It's different in the following ways:

- It's truly cross-platform: conda works on Windows (!), OS X and Linux. That makes it very different from apt, brew, dnf, yum, chocolatey ...
- Conda supports environments natively. You can have multiple environments with different versions of some library side-by-side -- and switch between them conveniently using `conda activate my_environment`.
- Conda supports distributing binaries. A lot of C++ libraries are difficult to build, or take a long time to build. Therefore shipping binaries (on all platforms) is a huge feature. You can find big libs such as boost on conda.
- Conda is language-agnostic: you have cargo for Rust, pip for Python and Pkg for Julia, Conan for C++ ... where will it stop?
- Contrary to the naive attempt, conda does true dependency solving -- conda uses a SAT solver to find a solution to the dependency problem if it exists.

And last but not least, conda has already found widespread use in many communities. The Python data science landscape and the R community are some of the heavy conda users out there. One of the largest community engineering efforts out there is probably the maintenance of conda-forge, a big repository of Open Source software that's built and maintained completely in the open.

That's why we (at QuantStack) think that conda is the package manager of the future -- not just for Python or R, but also for C++. In this blog post we'll show how convenient it is to use conda environments as a C++ user.

### Installing conda

The best way to get a minimal conda installation is using miniconda: 
<https://docs.conda.io/en/latest/miniconda.html>.
Once installed you can install new packages using 

`$ conda install xtensor xsimd -c conda-forge`
<no-indent></no-indent>
which will download the package and install it in the currently activated environemnt (the default environment is called "base").
The environment (on Linux) mirrors a standard unix root-environment (with all the `/etc`, `/include`, `/lib`... folders).

### Using the C++ library installed with conda

Using the installed C++ libraries is actually surprisingly easy -- given one uses CMake:

```sh
cd your_project
mkdir build && cd build
# make sure your environemnt is activated!
conda activate {base / myenv}
cmake .. -DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX 
```

Now CMake is going to prioritize the libraries and CMake modules found in the conda environment. 

If you are a Windows user, then the command will look slightly different:

```sh
cd your_project
mkdir build
cd build
# make sure your environemnt is activated!
conda activate {base / myenv}
cmake -G "NMake Makefiles" -DCMAKE_INSTALL_PREFIX=%MINICONDA%\\LIBRARY -DBUILD_TESTS=ON ..
```