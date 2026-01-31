(how-to-bootstrap-llvm)=
# How to bootstrap the LLVM package

This guide focuses on building the LLVM package from upstream sources, entirely bootstrapped, without any assumption of packages being available in the archive. Usually, you would not go through this process, as you can grab everything you need with `git-ubuntu` and probably bootstrap with the packages synced directly from Debian, as Ubuntu carries no delta on these packages. However, for the sake of understanding the package more deeply, this guide walks through all the bootstrapping steps and does not depend on any packages already being in the archive. This could be useful to understand if you encounter a problematic Debian merge.

It's worth noting that there are some upstream instructions, provided via the [official LLVM website](https://apt.llvm.org/building-pkgs.php). However, those instructions are focused on building and releasing snapshots, and assume basic familiarity with the package.

(llvm-getting-started)=
## Getting started

### Debian package files

If you are an LLVM maintainer, or a prospective one, you will want to have multiple versions of the package available easily. While not strictly necessary, having multiple single-branch checkouts in directories named after those branches lends itself to a good workflow, as you won't need to worry about things like a `debian/files` file causing the wrong things to be included in your source packages.  In this guide, we're going to be building version 21 of LLVM, so we'll check out that branch directly.

```bash
$ mkdir llvm && cd llvm
$ git clone https://salsa.debian.org/pkg-llvm-team/llvm-toolchain.git -b 21 21
```

Feel free to poke around in the package.  It can be helpful to scan the `README` file, but like the documentation linked above, it targets a different audience.

### LLVM source tree and tarball

Take a quick look at the `orig-tar.sh` script in the files you just checked out. You don't need to worry about all the details, just note that it points to the official LLVM Github repository, does a bunch of version parsing and mangling, and can automate the git steps to get a clean checkout of the correct LLVM source tree.  We'll use it to grab the source we need and to generate the orig tarball.

To keep things simple, we'll just build whatever the latest version in the changelog is. Add a new entry to the changelog with `dch -i`. Set the version and target as needed. In this guide we'll just building for a PPA, so the version has `~ppa0` appended and `UNRELEASED` was changed to `noble`.

You should also run `update-maintainer` to ensure that the maintainer information is correct.

After that change, the changelog shows version `1:21.1.8-3ubuntu1~ppa0`. Focusing just on the upstream LLVM version, that means we need to grab the source for version `21.1.8`. Now we're ready to run `orig-tar.sh`. Disregard the comment in the script that says to provide the version number twice for stable versions and the example showing that you should run it from within your cloned repo directory.  Instead, rom the top-level `llvm` directory you created before, do the following:

```bash
$ sh 21/debian/orig-tar.sh 21.1.8
```

This will kick off a full clone of the LLVM project, which will take some time. Once it finishes, we're ready to build.


## Building the package

Since LLVM is such a complex codebase, we need a build with multiple stages. This is due to circular dependencies such as llvm requiring llvm-spirv, which in turn requires llvm. If you're building a new patch release, or if the new version you want was synced to devel, that won't be a problem because you can use the existing version to bootstrap the new one. But because we are intentionally doing things the long way, we need to manually go through the bootstrapping stages. In some scenarios, like backports, you might be able to just do a binary copy of the package to your PPA.

Unfortunately, Launchpad doesn't support Debian build profiles, so the bootstrapping is a manual process. You can either manually modify `debian/control` so that it excludes any dependencies marked as `<!stage1>`, or you can use an experimental profile application script available in [the Foundations Sandbox](https://github.com/canonical/foundations-sandbox/blob/main/karljs/apply-build-profile.py).

[TODO] Put helper script somewhere public.

```{warning}
**Other Requirements** LLVM is big and low-level, so it's possible that it won't build cleanly. When creating this guide, LLVM 21 on Noble encountered an issue with PPC64 floating point representations, where the newer LLVM only support IEEE formats but the older dependencies were all built with the old IBM format. That's out of scope here, but just beware that such issues come up often.
```

```bash
$ cd 21
$ python3 ../apply-build-profile.py stage1
$ dpkg-buildpackage -S -nc
```

You're now ready to build this `stage1` package in a PPA.

```bash
$ dput ppa:<username>/<ppa_name> llvm-toolchain-21_21.1.8-3ubuntu1~ppa0_source.changes
```

The build will take some time, depending on which architectures you have enabled in your PPA. The next step will be to build the `spirv-llvm-translator` package, but it also has some dependencies that might not be available on your target distribution. Because you may need to backport a number of packages, this guide will not show them all. Try to grab as many as possible using `git-ubuntu` from devel or other archives that may have them.

Anything you can't find in the archive is likely on Debian Salsa. For instance:
- https://salsa.debian.org/xorg-team/vulkan/spirv-tools.git
- https://salsa.debian.org/xorg-team/vulkan/spirv-headers.git

Building LLVM 21 for Noble required new versions of both of those. Unfortunately grabbing the upstream tarballs didn't match those already in the archive, so there are likely some nuances known to the original maintainers. If that happens, just get the correct orig tarball from the archive, as shown.

### The `spirv-headers` package

```bash
$ git clone https://salsa.debian.org/xorg-team/vulkan/spirv-headers.git
$ cd spirv-headers
$ dch -i
$ update-maintainer
$ cd ..
$ wget https://launchpad.net/ubuntu/+archive/primary/+sourcefiles/spirv-headers/1.6.1+1.4.335.0-1/spirv-headers_1.6.1+1.4.335.0.orig.tar.gz
$ cd spirv-headers && dpkg-buildpackage -S -nc
$ dput ppa:<username>/<ppa_name> spirv-headers_1.6.1+1.4.335.0-1ubuntu1~ppa0_source.changes
```

### The `spirv-tools` package

Follow the same instructions as above, except you can use `uscan --download-current-version` this time to get the orig tarball.


### The `spirv-llvm-translator` package

This is the reason we're doing all this dependency porting. In building this guide, this package didn't build cleanly.  It was built with GCC 15, which is not available on Noble. Since it's a shared library, there's a Debian symbols file, which is deeply tied to the compiler in question, as well as a bunch of tests that depend on the version of clang we're trying to build.

The easiest way to continue making progress may be to modify the package to skip the tests for now.  Once we have the complete version of clang bootstrapped, we can rebuild this package with tests enabled.

The changes that were required for version 21 on Noble included:

In the control file
```diff
 Build-Depends: debhelper-compat (= 13),
  dh-sequence-pkgkde-symbolshelper,
  cmake,
- gcc (>= 4:15),
+ gcc (>= 4:13),
```

And in the rules file, disable the tests and the symbol checking:
```diff
+
+override_dh_auto_test:
+       @echo "Skipping tests for PPA backport"
+
+override_dh_makeshlibs:
+       dh_makeshlibs -- -c0
```

### The final `llvm-toolchain` package

With all those dependencies built, we're ready to build `llvm-toolchain-21` without the `<stage1>` restriction. If you used the script from earlier, you should be able to restore the original by doing:

```bash
$ cd 21
$ python3 ../apply-build-profile.py --restore
```

From there, bump the version in the changelog and ensure maintainer settings are still correct.  Re-build the source package and upload it.

```bash
$ dch -r
$ dpkg-buildpackage -S -nc
$ cd ..
$ dput ppa:<username>/<ppa_name> llvm-toolchain-21_21.1.8-2ubuntu1~24.04.5_source.changes
```
