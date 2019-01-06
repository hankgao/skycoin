
# Skycoin SWIG interfaces for generating client libraries

[![Build Status](https://travis-ci.org/skycoin/skycoin.svg)](https://travis-ci.org/skycoin/skycoin)

Skycoin SWIG interfaces provide the basic and resubale tools to build Skycoin client libraries
for [SWIG compatible programming languages](http://www.swig.org/compat.html) by linking against
[libskycoin](../cgo/README.md). The libraries themselves provide Skycoin developers with the
means to access core internals and API functions for implementing third-party applications.

## Structure and libraries

- **common** folder includes interfaces that should be reused by all client libraries. It provides
  typemaps for all basic types exported by the `libskycoin` API in order to make them accessible
  to the SWIG compiler used to build the particular client library.
- **dynamic** folder includes extensions meant to be reused by languages (often having dynamic
  type systems) that support functions returning multiple values
  * Used to build [PySkycoin](https://pypi.org/project/pyskycoin/)
- **static** folder includes extensions meant to be reused by languages (often having static
  type systems) that do not support functions returning multiple values
  * Used to build [LibSkycoinNet](https://www.nuget.org/packages/LibskycoinNet)

## Creating new client library projects

Even if SWIG is of invaluable help to create client libraries for multiple programming languages
the process of using it to create and build a new project is not fully automated. The different
characteristics of the languages theselves, but also the particular toolchains involved in each
case, make it almost impossible to achieve that goal. Some steps are recommended though
(in all cases filesystem paths are relative to repository root folder) :

- Create a project workspace for your client library project, which might include
  * version control (e.g. git) repository
  * compiler, linker, linter, make, build tools and the rest of the language-specific toolchain
- Add [skycoin/skycoin](https://github.com/skycoin/skycoin) source code at
  `./gopath/src/github.com/skycoin/skycoin` subfolder.
  * if using `git` then the right way to do it should be
    ```sh
    git submodule add -b master https://github.com/skycoin/skycoin gopath/src/github.com/skycoin/skycoin
    ```
  * feel free to checkout submodule `develop` branch for bleeding edge features or
    `master` branch if using stable interfaces is preferred
- Add a `Makefile` under the root folder of your client library project
- Create a temporary `./build` subfolder and do not track changes for it
  * e.g. [exclude it in .gitignore](https://git-scm.com/docs/gitignore)
    ```sh
    echo './build' >> ./.gitignore
    ```
- Quite likely the following filesystem paths will be needed in your `Makefile` so you might want
  to decare them at the top
  ```makefile
  # Working directory, usually the root folder of your version control repository
  PWD = $(shell pwd)
  # Internal GOPATH directory, needed for compiling libskycoin
  GOPATH_DIR = $(PWD)/gopath
  # Relative path to skycoin/skycoin submodule
  SKYCOIN_DIR = gopath/src/github.com/skycoin/skycoin
  # Relative path to libskycoin temporary build folder
  SKYBUILD_DIR = $(SKYCOIN_DIR)/build
  # Relative path to build libskycoin shared and/or static libraries
  BUILDLIBC_DIR = $(SKYBUILD_DIR)/libskycoin
  # Absolute path to build libskycoin shared and/or static libraries
  FULL_PATH_LIB = $(PWD)/$(BUILDLIBC_DIR)
  # Relative path to libskycoin source code folder
  LIBC_DIR = $(SKYCOIN_DIR)/lib/cgo
  # Relative path to top-level source code folder for skycoin/skycoin SWIG interfaces 
  LIBSWIG_DIR = $(SKYCOIN_DIR)/lib/swig
  # Client library build dir
  BUILD_DIR = build
  # Relative path to folder including Skycoin binaries, sho0uld they be needed
  BIN_DIR = $(SKYCOIN_DIR)/bin
  # Relative path to libskycoin .h library headers
  INCLUDE_DIR = $(SKYCOIN_DIR)/include
  # SWIG language switch to generate your library e.g. -python, -csharp, ...
  SWIG_LANG = '-somelang'
  # Path to folder containing source code generated by SWIG
  SWIG_OUTDIR = 'set/your/own/path'
  # File name of source code file generated by SWIG
  SWIG_OUTFILE = 'set_your_own_path.extension'
  ```
- It is recommended to declare the following variables so as to cache compilation
  results and rebuild libskycoin only when source files change. However this is
  completely up to project author and may be omitted
  ```makefile
  LIB_FILES = $(shell find $(SKYCOIN_DIR)/lib/cgo -type f -name "*.go")
  SRC_FILES = $(shell find $(SKYCOIN_DIR)/src -type f -name "*.go")
  SWIG_FILES = $(shell find $(LIBSWIG_DIR) -type f -name "*.i")
  HEADER_FILES = $(shell find $(SKYCOIN_DIR)/include -type f -name "*.h")
  ```
- Add a `configure` target in `Makefile` that (at least) creates the
  following temp folders
  ```makefile
  configure:
    mkdir -p $(BUILD_DIR)/usr/tmp $(BUILD_DIR)/usr/lib $(BUILD_DIR)/usr/include
	  mkdir -p $(BUILDLIBC_DIR) $(BIN_DIR) $(INCLUDE_DIR)
  ```
- If libskycoin will be statically linked to the client library it is recommended
  to add the following target to build `libskycoin.a`
  ```makefile
  $(BUILDLIBC_DIR)/libskycoin.a: $(LIB_FILES) $(SRC_FILES) $(HEADER_FILES)
  	rm -f $(BUILDLIBC_DIR)/libskycoin.a
  	GOPATH="$(GOPATH_DIR)" make -C $(SKYCOIN_DIR) build-libc-static
  	ls $(BUILDLIBC_DIR)
  	rm -f swig/include/libskycoin.h
  	mkdir -p swig/include
  	grep -v _Complex $(INCLUDE_DIR)/libskycoin.h > swig/include/libskycoin.h
  ```
- If libskycoin will be dynamically linked to the client library it is recommended
  to add the following target to build `libskycoin.so`
  ```makefile
  $(BUILDLIBC_DIR)/libskycoin.so: $(LIB_FILES) $(SRC_FILES) $(HEADER_FILES)
  	rm -f $(BUILDLIBC_DIR)/libskycoin.so
  	GOPATH="$(GOPATH_DIR)" make -C $(SKYCOIN_DIR) build-libc-shared
  	ls $(BUILDLIBC_DIR)
  	rm -f swig/include/libskycoin.h
  	mkdir -p swig/include
  	grep -v _Complex $(INCLUDE_DIR)/libskycoin.h > swig/include/libskycoin.h
  ```
- Add a `build-libc` target in `Makefile` that depends on `configure`,
  `libskycoin.a`, and / or `libskycoin.so` targets e.g.
  ```makefile
  build-libc: configure $(BUILDLIBC_DIR)/libskycoin.so
  ```
- Create a `build-swig` target in `Makefile` for generating library source code
  from SWIG interfaces like this
  ```makefile
  build-swig:
		#Generate structs.i from skytypes.gen.h
		rm -f $(LIBSWIG_DIR)/structs.i
		cp $(INCLUDE_DIR)/skytypes.gen.h $(LIBSWIG_DIR)/structs.i
		{ \
			if [[ "$$(uname -s)" == "Darwin" ]]; then \
				sed -i '.kbk' 's/#/%/g' $(LIBSWIG_DIR)/structs.i ;\
			else \
				sed -i 's/#/%/g' $(LIBSWIG_DIR)/structs.i ;\
			fi \
		}
	  swig $(SWIG_LANG) -Iswig/include -I$(INCLUDE_DIR) -outdir $(SWIG_OUTDIR) -o $(SWIG_OUTDIR)/$(SWIG_OUTFILE) $(LIBSWIG_DIR)/skycoin.i
  ```
- Create a `build` target in `Makefile` dependent on (at least) `build-libc`
  and `build-swig` targets
- Create other targets you might consider appropriate e.g. `install-deps`,
  `install`, `lint`, `test`, ...
- Write a test suite for your client library. Consider
  [libskycoin test suite](https://github.com/skycoin/skycoin/tree/master/lib/cgo/tests)
  as a reference
