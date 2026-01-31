# LLVM packaging guide


## Getting set up

### The `SKIP_COMMON_PACKAGES` variable

In `debian/rules` you'll see an environment variable called `SKIP_COMMON_PACKAGES`.
This variable determines whether or not to build the packages listed in the `debian/packages.common file`, which are all things that the LLVM package could theoretically build, but sometimes doesn't want to, as they are un-versioned.
They are all runtime libraries, and upstream LLVM makes guarantees about ABI stability for them.
That means if we have two LLVM packages, say 19 and 20, and 20 is building those libraries, we can choose to skip them in 19 and just rely on the versions produced by 20 already in the archive.
Generally, we need to set this according to whether we're working on the default version of LLVM or a different one, since we likely want the default package to produce those.

For some reason, the environment variables are harcoded in `debian/rules` after the correct values are detected.  That means we want to find this chunk of code, and where appropriate, comment it out or remove it.

```bash
# set these when building without the common packages
SKIP_COMMON_PACKAGES = yes
NEW_LLVM_VERSION = 21
```

That should allow you to re-generate the `debian/control` file.

### Regenerating the `debian/control` file

This file is not always in the right state upstream, so we need to regenerate it after we implement the common package fix.
The `README` says to use the `preconfigure` target, but it appears to actually be the `stamps/preconfigure` target.
You should just need:

```bash
debian/rules stamps/preconfigure
```

## The `wasi-libc` configure-time dependency

There's a good chance, if you're building directly from the upstream packaging files that you'll get an error when you run preconfigure related to not having `wasi-libc` installed.
There's some conditional code that disables the WASM support for older distros (Jammy, for instance), but when WASI is supported (Noble onwards), it expects that to be installed.
It's a build-dependency, so I find it annoying that it's required here.
You can either install `wasi-libc` on your host system, or you can delete the following lines from `debian/rules`:

```bash
	else \
		if ! dpkg -l|grep -q wasi-libc; then \
			echo "Could not find wasi-libc on the system"; \
			echo "Please check that the package is available on the system"; \
			echo "it might be that the 'hello' package is installed by another constraint"; \
			exit 1; \
		fi; \
	fi
```
