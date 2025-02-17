WEBR_ROOT = $(abspath ..)
ROOT = $(abspath .)

DOWNLOAD = $(ROOT)/download
BUILD = $(ROOT)/build
DIST = $(WEBR_ROOT)/dist
TOOLS = $(WEBR_ROOT)/tools
HOST = $(WEBR_ROOT)/host
WASM = $(WEBR_ROOT)/wasm

# This is symlinked at configure-time
include $(TOOLS)/fortran.mk

R_VERSION = $(shell cat R-VERSION)
export R_VERSION

R_TARBALL = $(DOWNLOAD)/R-$(R_VERSION).tar.gz
R_SOURCE = $(BUILD)/R-$(R_VERSION)
R_URL = https://cran.r-project.org/src/base/R-4/R-$(R_VERSION).tar.gz

R_HOST = $(HOST)/R-$(R_VERSION)
R_WASM = $(WASM)/R-$(R_VERSION)

# Configure your local environment in this file
-include ~/.webr-config.mk
export WEBR_REPO
export WEBR_LIB

WASM_OPT ?= -Oz
WASM_OPT_LDADD ?= $(WASM_OPT)
WASM_OPT_LDADD += -s DECLARE_ASM_MODULE_EXPORTS=0

WASM_CFLAGS := $(WASM_CFLAGS)
WASM_CFLAGS += -fPIC -fno-exceptions -fno-rtti $(WASM_OPT)

.PHONY: all
all: R

.PHONY: R
R: $(BUILD)/state/r-stage2

$(R_TARBALL):
	mkdir -p $(DOWNLOAD)
	wget $(R_URL) -O $@

$(BUILD)/state/r-patched: $(R_TARBALL)
	rm -rf $(R_SOURCE)
	mkdir -p $(BUILD)
	tar -C $(BUILD) -xf $(R_TARBALL)
	cp -r "$(WEBR_ROOT)/patches/R-$(R_VERSION)/." "$(R_SOURCE)/patches"
	cd "$(R_SOURCE)/patches" && quilt push -a
	mkdir -p $(BUILD)/state
	touch $@

STAGE1_CC ?= clang
STAGE1_CXX ?= clang++
STAGE1_FC ?= gfortran

# Stage 1: Build a native version of R so we can compile the default packages
$(BUILD)/state/r-stage1-configured: $(BUILD)/state/r-patched
	mkdir -p $(R_SOURCE)/build-stage1/doc
# Workaround for the lack of LaTeX packages
	cd $(R_SOURCE)/build-stage1/doc && \
	  touch NEWS NEWS.pdf NEWS.rds NEWS.2.rds NEWS.3.rds
	cd $(R_SOURCE)/build-stage1 && \
	  FC="$(STAGE1_FC)" \
	  CXX="$(STAGE1_CXX)" \
	  CC="$(STAGE1_CC)" \
	  FC="$(STAGE1_FC)" \
	  CPPFLAGS="$(STAGE1_CPPFLAGS)" \
	  CFLAGS="$(STAGE1_CFLAGS)" \
	  LDFLAGS="$(STAGE1_LDFLAGS)" \
	  ../configure \
	    --prefix="$(R_HOST)" \
	    --with-x=no \
	    --with-aqua=no \
	    --with-readline=no \
	    --with-jpeglib=no \
	    --with-cairo=no \
	    --disable-openmp \
	    --with-recommended-packages=no \
	    --enable-R-profiling=no \
	    --with-pcre2 \
	    --disable-nls \
	    --enable-byte-compiled-packages=no \
	    --enable-long-double=no
	touch $@

$(BUILD)/state/r-stage1: $(BUILD)/state/r-stage1-configured
	cd $(R_SOURCE)/build-stage1 && \
	  $(MAKE) R && \
	  $(MAKE) install
	touch $@

STAGE2_CPPFLAGS := $(STAGE2_CPPFLAGS)
STAGE2_CPPFLAGS += -I$(WASM)/include
STAGE2_CPPFLAGS += -DEXPEL_OLD_TO_NEW=1
STAGE2_CPPFLAGS += -s USE_BZIP2=1
STAGE2_CPPFLAGS += -s USE_ZLIB=1

STAGE2_CFLAGS := $(STAGE2_CFLAGS)
STAGE2_CFLAGS += $(STAGE2_CPPFLAGS) $(WASM_CFLAGS)

STAGE2_LDFLAGS := $(STAGE2_LDFLAGS)
STAGE2_LDFLAGS += -L$(WASM)/lib

MAIN_LDFLAGS  = -s MAIN_MODULE=1
MAIN_LDFLAGS += -s WASM=1
MAIN_LDFLAGS += -s WASM_BIGINT
MAIN_LDFLAGS += -s ALLOW_MEMORY_GROWTH=1
MAIN_LDFLAGS += -s STACK_SIZE=1MB
MAIN_LDFLAGS += -s EXIT_RUNTIME=1
MAIN_LDFLAGS += -s ERROR_ON_UNDEFINED_SYMBOLS=0
MAIN_LDFLAGS += -s EXPORTED_RUNTIME_METHODS=['allocateUTF8','UTF8ToString','callMain']
MAIN_LDFLAGS += -s FETCH=1
MAIN_LDFLAGS += --extern-pre-js $(WEBR_ROOT)/src/webR/pre.js
MAIN_LDFLAGS += $(FORTRAN_WASM_LDADD)
MAIN_LDFLAGS += $(WASM_OPT_LDADD)

SHLIB_LDFLAGS  = -s SIDE_MODULE=1
SHLIB_LDFLAGS += -s WASM_BIGINT $(WASM_OPT_LDADD)

# Stage 2: Reconfigure and build for wasm32-unknown-emscripten target
$(BUILD)/state/r-stage2-configured: $(BUILD)/state/r-patched $(FORTRAN_WASM_LIB)
	mkdir -p $(R_SOURCE)/build
	cd $(R_SOURCE)/build && \
	  EM_PKG_CONFIG_PATH="$(WASM)/lib/pkgconfig" \
	  MAIN_LDFLAGS="$(MAIN_LDFLAGS)" \
	  SHLIB_LDFLAGS="$(SHLIB_LDFLAGS)" \
	  CPPFLAGS="$(STAGE2_CPPFLAGS)" \
	  CFLAGS="$(STAGE2_CFLAGS)" \
	  LDFLAGS="$(STAGE2_LDFLAGS)" \
	  FFLAGS="" \
	  FPICFLAGS="-fPIC" \
	  FC="$(EMFC)" \
	  emconfigure ../configure \
	    --prefix="$(R_WASM)" \
	    --with-x=no \
	    --with-readline=no \
	    --with-jpeglib=no \
	    --with-cairo=no \
	    --disable-openmp \
	    --with-recommended-packages=no \
	    --enable-R-profiling=no \
	    --with-pcre2 \
	    --disable-nls \
	    --enable-byte-compiled-packages=no \
	    --enable-static=yes \
	    --host=wasm32-unknown-emscripten \
	    --with-internal-tzcode
# Disable umask which doesn't work well within Emscripten. Fixes
# permission issues when extracting tarballs.
	sed -i.bak '/D\["HAVE_UMASK"\]/d' $(R_SOURCE)/build/config.status
# Disable some unsupported features when running under Emscripten
	sed -i.bak '/D\["HAVE_WCSFTIME"\]/d' $(R_SOURCE)/build/config.status
	sed -i.bak '/D\["HAVE_SIGALTSTACK"\]/d' $(R_SOURCE)/build/config.status
	sed -i.bak '/D\["HAVE_GETRUSAGE"\]/d' $(R_SOURCE)/build/config.status
	sed -i.bak '/D\["HAVE_GETRLIMIT"\]/d' $(R_SOURCE)/build/config.status
	rm $(R_SOURCE)/build/config.status.bak
	cd $(R_SOURCE)/build && ./config.status
	touch $@

STAGE2_BUILD = $(R_SOURCE)/build

MAKE_WASM := $(MAKE)
MAKE_WASM += R_EXE="$(R_HOST)/bin/R --vanilla --no-echo"

MAKE_WASM_BUILD := $(MAKE_WASM)
MAKE_WASM_BUILD += R_INSTALL_LIBRARY="$(STAGE2_BUILD)/library"
MAKE_WASM_BUILD += R_ENABLE_JIT="false"
MAKE_WASM_BUILD += WEBR_HOST_METHODS="$(R_HOST)/lib/R/library/methods"
MAKE_WASM_BUILD += FORTRAN_WASM_LDADD="$(FORTRAN_WASM_LDADD)"

MAKE_WASM_INSTALL := $(MAKE_WASM)

$(BUILD)/state/r-stage2: $(BUILD)/state/r-stage1 $(BUILD)/state/r-stage2-configured
	cd $(WEBR_ROOT)/packages && \
	  $(MAKE_WASM) clean && $(MAKE_WASM) all
	cd $(STAGE2_BUILD) && \
	  $(MAKE_WASM_BUILD) R
	touch $@

.PHONY: Rprofile
Rprofile: $(BUILD)/state/r-stage2
	mkdir -p "$(R_WASM)/lib/R/etc/"
	echo "options(expressions=400)" > "$(R_WASM)/lib/R/etc/Rprofile.site"

.PHONY: install
install: install-tests Rprofile
	mkdir -p $(R_SOURCE)/build/doc
	cd $(R_SOURCE)/build/doc && \
	  touch NEWS NEWS.pdf NEWS.rds NEWS.2.rds NEWS.3.rds
	cd $(R_SOURCE)/build && \
	  $(MAKE_WASM_INSTALL) install-wasm
	cp -r "$(R_WASM)/dist/." $(DIST)

.PHONY: install-tests
install-tests: $(BUILD)/state/r-stage2
	cd $(R_SOURCE)/build && \
	  $(MAKE_WASM_BUILD) docs && \
	  $(MAKE_WASM_INSTALL) install-tests

.PHONY: rebuild-extra-pkgs
rebuild-extra-pkgs: $(BUILD)/state/r-stage2
	rm -f $(BUILD)/state/r-stage2
	cd $(WEBR_ROOT)/packages && \
	  $(MAKE_WASM) clean && $(MAKE_WASM) all
	cd $(STAGE2_BUILD)/src/main && \
	  $(MAKE_WASM_INSTALL) install-wasm
	cp -r "$(R_WASM)/dist/." $(DIST)

.PHONY: rebuild-core
rebuild-core: $(BUILD)/state/r-stage2
	rm -f $(BUILD)/state/r-stage2
	cd $(STAGE2_BUILD)/src/unix && \
	  $(MAKE_WASM_BUILD)
	cd $(STAGE2_BUILD)/src/main && \
	  $(MAKE_WASM_BUILD) R && \
	  $(MAKE_WASM_INSTALL) install-wasm
	cp -r "$(R_WASM)/dist/." $(DIST)

.PHONY: rebuild-modules
rebuild-modules: $(BUILD)/state/r-stage2
	rm -f $(BUILD)/state/r-stage2
	cd $(STAGE2_BUILD)/src/modules && \
	  $(MAKE_WASM_BUILD) && \
	  $(MAKE_WASM_BUILD) install
	cd $(STAGE2_BUILD)/src/main && \
	  $(MAKE_WASM_BUILD) R && \
	  $(MAKE_WASM_INSTALL) install-wasm
	cp -r "$(R_WASM)/dist/." $(DIST)

.PHONY: clean
clean:
	rm -rf $(BUILD) $(DOWNLOAD)

# Print Makefile variable
.PHONY: print-%
print-%  : ; @echo $* = $($*)
