arch = $(shell dpkg --print-architecture)
platform = $(shell uname -m)
prefix =

sysconfdir = ${prefix}/etc
exec_prefix = /usr
bindir = ${exec_prefix}/bin
libexecdir = ${exec_prefix}/libexec/allsky
sharedir = ${exec_prefix}/share/allsky

PDIR = $(shell dirname "$$PWD")
MYDIR = $(shell basename "$$PWD")
RIGHTPATH = 1

ifneq (src,$(MYDIR))
  RIGHTPATH = 0
endif

ifneq (1,$(RIGHTPATH))
  $(error Unexpected directory structure. This should be in allsky/src directory.)
else

.DEFAULT_GOAL := all

ROOTCHECK=$(shell id -u)
ifneq ($(ROOTCHECK),0)
  ifeq ($(PKGBUILD),1)
    ROOTCHECK=0
  endif
endif

ifeq ($(PKGBUILD),)
  PKGBUILD=0
endif

PKGPATH=$(shell which pkg-config)

ifneq ($(PKGPATH),)
  USB=$(shell pkg-config --exists libusb-1.0 && pkg-config --cflags --libs libusb-1.0)
  OPENCV = $(shell pkg-config --exists opencv && pkg-config --cflags --libs opencv || (pkg-config --exists opencv4 && pkg-config --cflags --libs opencv4))
endif
DEFS = -D_LIN -D_DEBUG -DGLIBC_20
CFLAGS = -Werror -Wall -Wno-psabi -Wno-unused-result -g -O2 -lpthread -pthread

# Starting with the Feb 2023 Bullseye release, both 32- and 64-bit return
# $(platform) of "aarch64" on 64-bit hardware so we need to check $(arch),
# which returns "armhf" on 32-bit OS and "armhf" on 64-bit OS.
# Ditto i386 arch vs x86_64 platform.

ifeq ($(platform), armv6l)
  CC = arm-linux-gnueabihf-g++
  AR= arm-linux-gnueabihf-ar
  CFLAGS += -march=armv6
  CFLAGS += -lrt
  ZWOSDK = -Llib/armv6 -I./include
else ifeq ($(arch), i386)
  CC = g++
  AR= ar
  ZWOSDK = -Llib/x86 -I./include
else ifeq ($(platform), x86_64)
  CC = g++
  AR= ar
  ZWOSDK = -Llib/x64 -I./include
else ifeq ($(arch), arm64)
	# Ubuntu 20.04 added by Jos Wennmacker.
  CC = g++
  AR= ar
  ZWOSDK = -Llib/armv8 -I./include
else ifeq ($(arch), armhf)
  CC = arm-linux-gnueabihf-g++
  AR= arm-linux-gnueabihf-ar
  ZWOSDK = -Llib/armv7 -I./include
else		# $(platform) = armv7l or $(arch) = armhf
  CC = arm-linux-gnueabihf-g++
  AR= arm-linux-gnueabihf-ar
  CFLAGS += -march=armv7 -mthumb
  ZWOSDK = -Llib/armv7 -I./include
endif


ifeq (,$(CC))
  $(warning Could not identify the proper compiler for your platform.)
  $(error Unknown platform $(platform))
endif

CFLAGS += $(DEFS) $(ZWOSDK)

all:check_deps capture_ZWO capture_RPi startrails keogram sunwait uhubctl
.PHONY : all

ifneq ($(shell id -u), 0)
deps:
	echo This must be run with root permissions.
	echo Please run 'sudo make deps'
else
deps:
	@echo `date +%F\ %R:%S` Installing build dependencies...
	@apt update && apt -y install libopencv-dev libusb-dev libusb-1.0-0-dev ffmpeg lftp imagemagick bc
endif

.PHONY : deps


check_deps:
ifeq (,$(USB))
	  $(error Did not find USB Libraries, try 'sudo make deps')
endif
ifeq (,$(OPENCV))
	  $(error Did not find any OpenCV Libraries, try 'sudo make deps')
endif
.PHONY : check_deps

sunwait:
	@echo `date +%F\ %R:%S` Building sunwait...
	@git submodule init
	@git submodule update	
	@$(MAKE) -C sunwait-src
	@cp sunwait-src/sunwait .
	@echo `date +%F\ %R:%S` Done.

# This is only needed for GitHub to compile
include/allsky_common.h: include/allsky_common.h.repo
	@echo creating fake $@ ...
	@cp include/allsky_common.h.repo $@

allsky_common.o: allsky_common.cpp include/allsky_common.h
	@echo Building $@ ...
	@$(CC) -c  allsky_common.cpp -o $@ $(CFLAGS) $(OPENCV)

mode_mean.o: mode_mean.cpp include/mode_mean.h include/allsky_common.h
	@echo Building $@ ...
	@$(CC) -c  mode_mean.cpp -o $@ $(CFLAGS) $(OPENCV)

capture_RPi.o: capture_RPi.cpp ASI_functions.cpp include/mode_mean.h include/allsky_common.h
	@echo Building $@ ...
	@$(CC) -c  capture_RPi.cpp -o $@ $(CFLAGS) $(OPENCV)

capture_ZWO.o: capture_ZWO.cpp ASI_functions.cpp include/allsky_common.h
	@echo Building $@ ...
	@$(CC) -c capture_ZWO.cpp -o $@ $(CFLAGS) $(OPENCV)

capture_ZWO: capture_ZWO.o allsky_common.o
	@echo `date +%F\ %R:%S` Building $@ program...
	@$(CC) -o $@ $(CFLAGS)  capture_ZWO.o allsky_common.o $(OPENCV) -lASICamera2 $(USB)
	@echo `date +%F\ %R:%S` Done.

capture_RPi:capture_RPi.o allsky_common.o mode_mean.o
	@echo `date +%F\ %R:%S` Building $@ program...
	@$(CC) -o $@ $(CFLAGS) capture_RPi.o allsky_common.o $(OPENCV) mode_mean.o
	@echo `date +%F\ %R:%S` Done.

keogram:keogram.cpp
	@echo `date +%F\ %R:%S` Building $@ program...
	@$(CC) $@.cpp -o $@ $(CFLAGS) $(OPENCV)
	@echo `date +%F\ %R:%S` Done.

startrails:startrails.cpp
	@echo `date +%F\ %R:%S` Building $@ program...
	@$(CC) $@.cpp -o $@ $(CFLAGS) $(OPENCV)
	@echo `date +%F\ %R:%S` Done.

uhubctl:uhubctl.c
	@echo `date +%F\ %R:%S` Building $@ program...
# This comes from the uhubctl Makefile:
	@cc -g -O0 -Wall -Wextra -std=c99 -pedantic -DPROGRAM_VERSION=\"2.5.0\" -I/usr/include/libusb-1.0 $@.c -o $@ -Wl,-zrelro,-znow -lusb-1.0
	@echo `date +%F\ %R:%S` Done.

symlink: all
	@echo `date +%F\ %R:%S` Symlinking binaries...
	@ln -s $$PWD/capture_ZWO ../bin/
	@ln -s $$PWD/capture_RPi ../bin/
	@ln -s $$PWD/keogram ../bin/
	@ln -s $$PWD/startrails ../bin/
	@ln -s $$PWD/uhubctl ../bin/

.PHONY: symlink

ifneq ($(ROOTCHECK), 0)
install uninstall:
	@echo This must be run with root permissions.
	@echo Please run 'sudo make $@'
else
install:
	@echo `date +%F\ %R:%S` Copying binaries...
	@if [ $(PKGBUILD) -eq 1 ]; then \
	  [ ! -e $(DESTDIR)$(bindir) ] && mkdir -p $(DESTDIR)$(bindir); \
	  install capture_ZWO $(DESTDIR)$(bindir); \
	  install capture_RPi $(DESTDIR)$(bindir); \
	  install keogram $(DESTDIR)$(bindir); \
	  install startrails $(DESTDIR)$(bindir); \
	  install uhubctl $(DESTDIR)$(bindir); \
	else \
	  [ ! -e ../bin ] && mkdir -p ../bin; \
	  install -o $(SUDO_USER) -g $(SUDO_USER) capture_ZWO ../bin/; \
	  install -o $(SUDO_USER) -g $(SUDO_USER) capture_RPi ../bin/; \
	  install -o $(SUDO_USER) -g $(SUDO_USER) keogram ../bin/; \
	  install -o $(SUDO_USER) -g $(SUDO_USER) startrails ../bin/; \
	  install -o $(SUDO_USER) -g $(SUDO_USER) uhubctl ../bin/; \
	fi
	@install sunwait $(DESTDIR)$(bindir)

uninstall:
	@echo `date +%F\ %R:%S` Removing binaries...
	@if [ $(PKGBUILD) -eq 1 ]; then \
	  rm -f $(DESTDIR)$(bindir)/capture_ZWO; \
	  rm -f $(DESTDIR)$(bindir)/capture_RPi; \
	  rm -f $(DESTDIR)$(bindir)/keogram; \
	  rm -f $(DESTDIR)$(bindir)/startrails; \
	  rm -f $(DESTDIR)$(bindir)/sunwait; \
	  rm -f $(DESTDIR)$(bindir)/uhubctl; \
	else \
	  rm -f ../bin/capture_ZWO; \
	  rm -f ../bin/capture_RPi; \
	  rm -f ../bin/keogram; \
	  rm -f ../bin/startrails; \
	  rm -f ../bin/uhubctl; \
	fi

endif # sudo / root check
.PHONY : install uninstall

clean:
	rm -f capture_ZWO capture_RPi startrails keogram sunwait uhubctl *.o *.a
.PHONY : clean

endif # Correct directory structure check

%:
	@echo `date +%F\ %R:%S` nothing to do for $@
