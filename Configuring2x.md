# Configuring Duktape 2.x for build

## Overview

Duktape distributable contains prepared source files and a multi-platform
autodetecting `duk_config.h` for Duktape default configuration:

```
$ tar xvfJ duktape-2.0.0.tar.xz; cd duktape-2.0.0
$ ls -l src
total 3400
-rw-rw-r-- 1 duktape duktape  114927 Aug 30 03:57 duk_config.h
-rw-rw-r-- 1 duktape duktape   48651 Aug 30 03:57 duk_source_meta.json
-rw-rw-r-- 1 duktape duktape 3238266 Aug 30 03:57 duktape.c
-rw-rw-r-- 1 duktape duktape   68213 Aug 30 03:57 duktape.h
```

You can include these in your build directly: ensure include path provides
access to `duk_config.h` and `duktape.h`, and add `duktape.c` to the set of
C files built.

If you want to use custom Duktape configuration options, use
`tools/configure.py`.  For example:

```
$ tar xvfJ duktape-2.0.0.tar.xz; cd duktape-2.0.0

# Create an empty temporary directory
$ rm -rf /tmp/output; mkdir /tmp/output

# Custom configuration:
#   - Low memory configuration
#   - Enable fastint support on top of low memory configuration
#   - Enable ROM built-in support
#   - DLL mode, affects Windows build (declspec declarations)
$ python tools/configure.py \
      --output-directory /tmp/output \
      --source-directory src-input \
      --config-metadata config \
      --option-file profiles/config/low_memory.yaml \
      -DDUK_USE_FASTINT \
      --rom-support \
      --dll

# The output directory will get prepared config and sources.
# Include them in your build.
$ ls -l /tmp/output
total 4324
-rw-rw-r-- 1 duktape duktape   91440 Aug 30 04:42 duk_config.h
-rw-rw-r-- 1 duktape duktape   48653 Aug 30 04:42 duk_source_meta.json
-rw-rw-r-- 1 duktape duktape 4213240 Aug 30 04:42 duktape.c
-rw-rw-r-- 1 duktape duktape   68201 Aug 30 04:42 duktape.h
```

The `configure.py` tool has a lot of options, use `--help` to view option list.
The minimum options (to create sources with default options) are:

```
# Default configuration, combined (amalgamated) sources.
$ python tools/configure.py \
      --output-directory /tmp/output \
      --source-directory src-input \
      --config-metadata config
```

See also:

* https://github.com/svaarala/duktape/tree/master/config/config-options:
  metadata for available config options.

* https://github.com/svaarala/duktape/blob/master/src/builtins.yaml:
  built-in object metadata, and a description of how to write metadata
  files to edit built-in objects or add new built-in bindings.

## Differerences to 1.x:

* All configuration options are expressed in `duk_config.h`, and the compiler
  command line `DUK_OPT_xxx` feature options have been removed.

* The `configure.py` tool subsumes `genconfig.py`: it generates both a
  `duk_config.h` and prepared sources matching the configuration.  This makes
  it possible to modify built-in Ecmascript objects, modify Unicode tables,
  etc, from the end user distributable without having to use the main repo.

* You can still use `genconfig.py` as a separate tool.  However, it's
  recommended to use `configure.py` instead.

## Common options

### Fatal error handler (RECOMMENDED)

It is **strongly recommended** that you provide a default fatal error
handler:

* `-DDUK_USE_FATAL_HANDLER`

Example fatal error handler macro:

```
'-DDUK_USE_FATAL_HANDLER(udata,msg)=do { const char *fatal_msg = (msg); fprintf(stderr, "*** FATAL ERROR: %s\n", fatal_msg ? fatal_msg : "no message"); abort(); } while (0)'
```

### Windows DLL

When building a Windows DLL, use the `--dll` option in configure.py.

### Avoid packed duk_tval

On some platforms the IEEE 754 bit representation tricks used for the
8-byte packed `duk_tval` won't work.  You can force unpacked duk_tval
representation using:

* `-UDUK_USE_PACKED_TVAL`

### Troubleshooting

* `-DDUK_USE_ASSERTIONS`: enable assertions.  Assertion failures trigger a
  fatal error, so you should also defined a fatal error handler.  Note that
  assertions have a large impact on footprint and performance, so don't keep
  them enabled in production builds.

* `-DDUK_USE_SELF_TESTS`: enable self tests, often very useful in diagnosis.
  Self test failures cause debug prints and assertion failures.

* `-DDUK_USE_DEBUG`, `-DDUK_USE_DEBUG_LEVEL=0`, and `-DDUK_USE_DEBUG_WRITE`:
  enable debug logging.

Example debug log writer option:

```
'-DDUK_USE_DEBUG_WRITE(level,file,line,func,msg)=do {fprintf(stderr, "D%ld %s:%ld (%s): %s\n", (long) (level), (file), (long) (line), (func), (msg));} while(0)'
```
