#
# EFI Boot Guard
#
# Copyright (c) Siemens AG, 2017
#
# Authors:
#  Claudius Heine <ch@denx.de>
#
# This work is licensed under the terms of the GNU GPL, version 2.  See
# the COPYING file in the top-level directory.
#
# Based on Makefile.am of gummiboot,
# Copyright (C) 2013 Karel Zak <kzak@redhat.com>
#

ACLOCAL_AMFLAGS = -I m4 ${ACLOCAL_FLAGS}
AM_MAKEFLAGS = --no-print-directory

OBJCOPY ?= objcopy

efibootguarddir = $(libdir)/efibootguard

AM_CPPFLAGS = -I$(top_srcdir)/include -I$(top_srcdir) -include config.h

AM_CFLAGS = \
	-Wno-unused-parameter \
	-Wmissing-prototypes \
	-fshort-wchar \
	-DHAVE_ENDIAN_H \
	-D_GNU_SOURCE \
	-D_FILE_OFFSET_BITS=64 \
	-Werror

AM_LDFLAGS = -L$(top_builddir)/

ARFLAGS = cr

EXTRA_DIST = autogen.sh README LICENSE
CLEANFILES =

define filechk
	$(AM_V_at)set -e;			\
	echo '  CHK      $@';			\
	mkdir -p $(dir $@);			\
	$(filechk_$(1)) < $< > $@.tmp;		\
	if [ -r $@ ] && cmp -s $@ $@.tmp; then	\
		rm -f $@.tmp;			\
	else					\
		echo '  UPD      $@';		\
		mv -f $@.tmp $@;		\
	fi
endef

#
# Version header
#
define filechk_version
	$(top_srcdir)/gen_version_h $(top_srcdir)/
endef

GEN_VERSION_H := $(top_builddir)/version.h

$(GEN_VERSION_H): $(top_srcdir)/Makefile.in FORCE
	$(call filechk,version)

CLEANFILES += $(GEN_VERSION_H)

#
# Static libraries
#
lib_LIBRARIES = libebgenv.a

libebgenv_a_SOURCES = \
	env/@env_api_file@.c \
	env/env_api.c \
	env/env_config_file.c \
	env/env_config_partitions.c \
	env/env_disk_utils.c \
	env/uservars.c \
	tools/ebgpart.c

libebgenv_a_CPPFLAGS = \
	$(AM_CPPFLAGS) \
	-I$(top_srcdir)/tools

libebgenv_a_CFLAGS = \
	$(AM_CFLAGS)

pkginclude_HEADERS = \
	include/ebgenv.h

#
# bg_setenv binary
#
bin_PROGRAMS = bg_setenv

bg_setenv_SOURCES = \
	tools/bg_setenv.c

bg_setenv_CFLAGS = \
	$(AM_CFLAGS)

bg_setenv_LDADD = \
	-lebgenv \
	-lz

bg_setenv_DEPENDENCIES = \
	libebgenv.a

install-exec-hook:
	$(LN_S) -f bg_setenv$(EXEEXT) \
		$(DESTDIR)$(bindir)/bg_printenv$(EXEEXT)

#
# EFI compilation
#
# This part of the build system uses custom make rules and bypass regular
# automake to provide absolute control on compiler and linker flags.
#
efi_loadername = efibootguard$(MACHINE_TYPE_NAME).efi

efi_sources = \
	drivers/watchdog/init_array_start.S \
	drivers/watchdog/i6300esb.c \
	drivers/watchdog/atom-quark.c \
	drivers/watchdog/itco.c \
	drivers/watchdog/init_array_end.S \
	env/syspart.c \
	env/fatvars.c \
	utils.c \
	bootguard.c \
	main.c

efi_cppflags = \
	-I$(top_builddir) -include config.h \
	-I$(top_srcdir)/include \
	-I$(GNUEFI_SYS_DIR)/usr/include \
	-I$(GNUEFI_INC_DIR) \
	-I$(GNUEFI_INC_DIR)/$(ARCH) \
	-I$(GNUEFI_INC_DIR)/$(MACHINE_TYPE_NAME)

efi_cflags = \
	-Wall \
	-Wextra \
	-std=gnu99 \
	-nostdinc \
	-ggdb -O0 \
	-fpic \
	-fshort-wchar \
	-ffreestanding \
	-fno-strict-aliasing \
	-fno-stack-protector \
	-Wsign-compare \
	-mno-sse \
	-mno-mmx \
	$(CFLAGS)

if ARCH_X86_64
efi_cflags += \
	-DEFI_FUNCTION_WRAPPER \
	-mno-red-zone
endif

efi_ldflags = \
	-T $(GNUEFI_LIB_DIR)/elf_$(ARCH)_efi.lds \
	-shared \
	-Bsymbolic \
	-nostdlib \
	-znocombreloc \
	--warn-common \
	--no-undefined \
	--fatal-warnings \
	-L $(GNUEFI_LIB_DIR) \
	$(GNUEFI_LIB_DIR)/crt0-efi-$(ARCH).o

efi_objects_pre1 = $(efi_sources:.c=.o)
efi_objects_pre2 = $(efi_objects_pre1:.S=.o)
efi_objects = $(addprefix $(top_builddir)/,$(efi_objects_pre2))
efi_solib = $(top_builddir)/efibootguard$(MACHINE_TYPE_NAME).so

# automake stuff
efibootguard_DATA = $(efi_loadername)
CLEANFILES += $(efi_objects) $(efi_solib) $(efi_loadername)
EXTRA_DIST += $(efi_sources)

$(top_builddir)/%.o: $(top_srcdir)/%.c
	@$(MKDIR_P) $(shell dirname $@)/
	$(AM_V_CC)$(GNUEFI_CC) $(efi_cppflags) $(efi_cflags) -c $< -o $@

$(top_builddir)/env/%.o: $(top_srcdir)/env/%.c
	@$(MKDIR_P) $(shell dirname $@)/
	$(AM_V_CC)$(GNUEFI_CC) $(efi_cppflags) $(efi_cflags) -c $< -o $@

$(top_builddir)/drivers/watchdog/%.o: $(top_srcdir)/drivers/watchdog/%.c
	@$(MKDIR_P) $(shell dirname $@)/
	$(AM_V_CC)$(GNUEFI_CC) $(efi_cppflags) $(efi_cflags) -c $< -o $@

$(top_builddir)/drivers/watchdog/%.o: $(top_srcdir)/drivers/watchdog/%.S
	@$(MKDIR_P) $(shell dirname $@)/
	$(AM_V_CC)$(GNUEFI_CC) $(efi_cppflags) $(efi_cflags) -c $< -o $@

$(top_builddir)/main.o: $(GEN_VERSION_H)

$(efi_solib): $(efi_objects)
	$(AM_V_CCLD)$(LD) $(efi_ldflags) $(efi_objects) \
		-o $@ -lefi -lgnuefi $(shell $(CC) $(CFLAGS) -print-libgcc-file-name); \
	nm -D -u $@ | grep ' U ' && exit 1 || :

$(efi_loadername): $(efi_solib)
	$(AM_V_GEN) $(OBJCOPY) -j .text -j .sdata -j .data -j .dynamic \
	  -j .dynsym -j .rel -j .rela -j .reloc -j .init_array \
	  --target=efi-app-$(ARCH) $< $@

$(top_builddir)/tools/bg_setenv-bg_setenv.o: $(GEN_VERSION_H)

bg_printenvdir = $(top_srcdir)

bg_printenv: $(bg_setenv)
	$(LN_S) -f bg_setenv bg_printenv

all-local: bg_printenv

CLEANFILES += bg_printenv

# Tests depend on libraries being built - start with "."
SUBDIRS = . tools/tests

FORCE:

.PHONY: FORCE
