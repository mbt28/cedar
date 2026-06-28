# Build cedar-decode-test: a standalone validator for the Cedar VE + ION driver
# (this repo) plus the libcedarc userspace. Decodes a raw H.264 Annex-B stream
# straight through the VideoEngine -- no ffmpeg -- and reports frames / fps.
#
# Requires:
#   - libcedarc (https://github.com/aodzip/libcedarc) headers + libs
#   - the libcedarc fork that exports ion_alloc_get_dmabuf_fd() (only needed to
#     LINK; used by the --be/--drm display modes)
#   - libdrm
CC        ?= gcc
PKG_CONFIG ?= pkg-config
CFLAGS    ?= -O2 -Wall
LIBS       = -lvdecoder -lcdc_base -lMemAdapter -lVE -lvideoengine -ldl -lrt -lpthread

cedar-decode-test: cedar_decode_test.c
	$(CC) $(CFLAGS) $(shell $(PKG_CONFIG) --cflags libdrm) $< -o $@ \
		$(LIBS) $(shell $(PKG_CONFIG) --libs libdrm)

clean:
	rm -f cedar-decode-test

.PHONY: clean
