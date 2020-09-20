---
layout: post
title: "boa, a conda build alternative and ideas for source distribution"
author: "Wolf Vollprecht"
summary: boa is a new conda-build alternative that produces conda packages -- and also aims to enable a "source distribution" and easy re-configurability of conda packages
image: /assets/images/2020/boa_header.png
categories: posts
---

boa is a new conda-build alternative that produces conda packages with mamba as solver -- and also aims to enable a "source distribution" and easy re-configurability of conda packages. Here I am dumping some ideas on how this could work. You can find the [source code here on github](https://github.com/mamba-org/boa).

![boa](/assets/images/2020/boa_header.png)

This facilitates reproducibility (as we also archive the source code used to produce the binary artifacts) and adaptability to many different configurations and makes it easy to create conda environments with custom compiler specifications or specialized configurations. This will be especially interesting for High-Performance computing where each cluster uses their own compiler stack and specialized flags. In the future you will be able to re-build conda packages on your machine easily!

This will be enabled on the server by two components:

- A _source_ "subdir" that will contain the required source files. This only works for source-tarballs. The structure of these files will be `conda-forge/source/curl-7.20.0-<SHA256>` where SHA256 is the full SHA256 sum that is used as "build"-string. The source files intentionally will overlap between different build numbers _for the same version_.
- A _recipe_ "subdir" that contains a package which includes the recipes used to build the final package. This recipe package contains the `meta.yaml`, the build scripts and the required patches.

A difference between these new channels and the regular ones is that package with the same name but different versions can be installed side-by-side without clobbering: recipes will go into a `recipe/curl/7.20.0-4/...` subdir, and sources will go to `source/curl/7.20.0/tarball.tar.gz` folder without _clobbering_ between versions.

With the new recipe spec we also have the chance to add a features mechanism to conda-recipes. I am very excited about this as it comes as a quite natural addition to the current conda recipes & packages. A new section will be added to the recipes called "features" that contains a dictionary of optional features that can be enabled (or disabled!) at build time.

Let's take cURL as an example: it comes with many different build-time features that might be desirable (or not). For example at build time, one can choose between many different SSL implementations, support for FTP transfer, optionally enable LDAP, or fancy HTTP2 support. For a _binary_ distribution it might make sense to enable all these features (or most of them) as conda-forge or Fedora & Ubuntu do. It is great for a _source_ distribution to be able to change these configurations -- for example if one wants to build a minimal binary that will be statically linked into another program (which we coincidentally want to do for micromamba).

Note: we are also working on a new recipe-spec which will only support a limited set of Jinja2 and won't contain selectors-as-comments as the two main differences. Also working with multiple outputs will be improved (you can find the current work in progress spec here with lots of discussion here: https://github.com/mamba-org/conda-specs/blob/master/proposed_specs/recipe.md)

```yaml
context:
  version: "7.72.0"

package:
  name: curl
  version: "{{ "{{ version " }} }}"

source:
  url: http://curl.haxx.se/download/curl-7.72.0.tar.bz2
  sha256: 9d52a4d80554f9b0d460ea2be5d7be99897a1a9f681ffafe739169afd6b4f224

build:
  number: 0

requirements:
  build:
    - compiler-cxx
    - make
    - sel(unix): perl
    - sel(unix): pkg-config
  host:
    - zlib
    - libssh2
    - openssl
    - zlib

###
# Here the features are added
###

features:
  - name: ldap
    default: off
  - name: http2
    description: use libnghttp2 for http2 support
    requirements:
      host:
        - libnghttp2
    default: on
  - name: krb5
    description: use krb5 (gss) for FTP
    requirements:
      host:
        - krb5
    default: on
```

The requirements listed for each feature are simply added to the requirements for the full package. With these feature descriptions, special environment variables are going to be added to the `build.sh` / `bld.bat` scripts, so that the `./configure` script call can look something like this:

```bash
./configure \
    --prefix=${PREFIX} \
    --host=${HOST} \
    --with-ca-bundle=${PREFIX}/ssl/cacert.pem \
    --without-ssl \
    --with-zlib=${PREFIX} \
    --with-libssh2=${PREFIX} \
    $(has_feature FEATURE_LDAP --enable-ldap --disable-ldap) \
    $(has_feature FEATURE_KRB5 --with-gssapi=${PREFIX} "") \
    $(has_feature FEATURE_HTTP2 --with-nghttp2=${PREFIX} --without-nghttp2)
```

Then one will be able to create a custom build of curl by running 

`boa build curl[http2, ~ldap, krb5]`

Prefixing the feature name with `~` will negate (turn off) that specific feature.

A special feature will be always added to the features called `static` that will add a variable to the build script that requests a static build.

The activated (and deactivated) features will be added to the _hash_ of a package. This will make it possible to cache the compiled packages either locally, or on a quetz instance to save compilation time.

It will also be possible to add a "feature" package to the outputs of a regular package, so that working with multiple outputs becomes straight-forward and builds up on top of the same feature mechanism. For example you could imagine the following outputs section:

```yaml
outputs:
  - name: curl
    uses_features: [nghttp2, krb5]
  - name: curl-static
    uses_features: [static, nghttp2, krb5]
...
```

The last step will be to allow the specification of dependencies with features in the meta.yaml of other packages. For example, a future micromamba meta.yaml file could specify requirements like this:

```yaml
requirements:
  host:
    - curl[static, ~krb5, ~ldap, http2]
    - libarchive[static, bz2, xz]
    - yaml-cpp[static]
```

This would request minimal, statically linked libraries for the dependencies. The solver would first check if it can find binary archives on the cache for the most recent version, and if not, it would pull the recipe and source from the server and start building according to these feature specifications.

## The road ahead

We already have an initial implementation for the new recipe spec, as well as the "features" described in this post in boa (PR: [https://github.com/mamba-org/boa/pull/26](https://github.com/mamba-org/boa/pull/26)). A lot of work remains to be done, in `quetz` (to support the mentioned additional subdirs), `mamba` (to support installing same-name but different version packages side-by-side), and `boa`!
Additionally, we already used boa to build lots of robotics packages in the RoboStack channel with great success!
In the future we want make boa super-stable and finalize the specs and features implementation.

Another area of work that I am very interested in is to make it as simple as possible to cross-compile packages to different operating systems (from Linux to OS X and Windows) and CPU architectures (such as ARMv8 (aarch64), ppc64le and s390x).