# Allwinner CedarX Driver for Mainline Linux
### VideoEngine driver based on Allwinner H6 Homlet BSP
### Ion driver based on Google Android Ion

## Kernel versions

The driver is kept building across kernel releases on separate branches:

| Branch | Kernel | Notes |
|---|---|---|
| **`master`** | **6.6** (6.4+) | default |
| `6.1_kernel_updates` | 6.1 | (works on 5.10+, untested) |
| `5.4_kernel_updates` | 5.4 | original baseline |

`master` adapts the VE + ION drivers to the 6.4–6.6 kernel API:
`vma->vm_flags |= …` → `vm_flags_set()`, `zap_page_range()` → `zap_page_range_single()`,
`class_create()` without the `THIS_MODULE`/owner arg, and Kconfig `---help---` → `help`.

## Install

### Put all files in "drivers/staging/media/sunxi/cedar"

### Add source to "drivers/staging/media/sunxi/Kconfig"
```
source "drivers/staging/media/sunxi/cedar/Kconfig"
```
Demo
```
# SPDX-License-Identifier: GPL-2.0
config VIDEO_SUNXI
    bool "Allwinner sunXi family Video Devices"
    depends on ARCH_SUNXI || COMPILE_TEST
    help
      If you have an Allwinner SoC based on the sunXi family, say Y.

      Note that this option doesn't include new drivers in the
      kernel: saying N will just cause Kconfig to skip all the
      questions about Allwinner media devices.

if VIDEO_SUNXI

source "drivers/staging/media/sunxi/cedrus/Kconfig"
source "drivers/staging/media/sunxi/cedar/Kconfig"

endif
```

### Add obj to "drivers/staging/media/sunxi/Makefile"
```
obj-y += cedar/
```
Demo
```
# SPDX-License-Identifier: GPL-2.0
obj-$(CONFIG_VIDEO_SUNXI_CEDRUS)        += cedrus/
obj-y += cedar/
```

## DeviceTree
### Demo for Allwinner V3 / V3s / S3L / S3
```
syscon: syscon@1c00000 {
    compatible = "allwinner,sun8i-v3s-system-controller", "allwinner,sun8i-h3-system-control", "syscon";
    reg = <0x01c00000 0xd0>;
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    sram_c: sram@1d00000 {
        compatible = "mmio-sram";
        reg = <0x01d00000 0x80000>;
        #address-cells = <1>;
        #size-cells = <1>;
        ranges = <0 0x01d00000 0x80000>;

        ve_sram: sram-section@0 {
            compatible = "allwinner,sun8i-v3s-sram-c", "allwinner,sun4i-a10-sram-c1";
            reg = <0x000000 0x80000>;
        };
    };
};

cedarx: video-codec@1c0e000 {
    compatible = "allwinner,sun8i-v3-cedar";
    reg = <0x01c0e000 0x1000>;
    clocks = <&ccu CLK_BUS_VE>, <&ccu CLK_VE>, <&ccu CLK_DRAM_VE>;
    clock-names = "ahb", "mod", "ram";
    resets = <&ccu RST_BUS_VE>;
    interrupts = <GIC_SPI 58 IRQ_TYPE_LEVEL_HIGH>;
    allwinner,sram = <&ve_sram 1>;
    status = "disabled";
};

ion: ion {
    compatible = "allwinner,sunxi-ion";
    status = "disabled";
    heap_cma@0{
        compatible = "allwinner,cma";
        heap-name  = "cma";
        heap-id    = <0x4>;
        heap-base  = <0x0>;
        heap-size  = <0x0>;
        heap-type  = "ion_cma";
    };
};
```
### Demo for Allwinner F1C100s / F1C200s

In drivers/clk/sunxi-ng/ccu-suniv-f1c100s.c

Change

    static SUNXI_CCU_GATE(ve_clk, "ve", "pll-audio", 0x13c, BIT(31), 0);

To

    static SUNXI_CCU_GATE(ve_clk, "ve", "pll-ve", 0x13c, BIT(31), 0);

```
sram-controller@1c00000 {
    compatible = "allwinner,suniv-f1c100s-system-control",
             "allwinner,sun4i-a10-system-control";
    reg = <0x01c00000 0x30>;
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    sram_c: sram@1d00000 {
        compatible = "mmio-sram";
        reg = <0x01d00000 0x80000>;
        #address-cells = <1>;
        #size-cells = <1>;
        ranges = <0 0x01d00000 0x80000>;

        ve_sram: sram-section@0 {
            compatible = "allwinner,suniv-f1c100s-sram-c", "allwinner,sun4i-a10-sram-c1";
            reg = <0x000000 0x80000>;
        };
    };
};

cedarx: video-codec@1c0e000 {
    compatible = "allwinner,suniv-f1c100s-cedar";
    reg = <0x01c0e000 0x1000>;
    clocks = <&ccu CLK_BUS_VE>, <&ccu CLK_VE>, <&ccu CLK_DRAM_VE>;
    clock-names = "ahb", "mod", "ram";
    resets = <&ccu RST_BUS_VE>;
    interrupts = <34>;
    allwinner,sram = <&ve_sram 1>;
    status = "disabled";
};

ion: ion {
    compatible = "allwinner,sunxi-ion";
    status = "disabled";
    heap_cma@0{
        compatible = "allwinner,cma";
        heap-name  = "cma";
        heap-id    = <0x4>;
        heap-base  = <0x0>;
        heap-size  = <0x0>;
        heap-type  = "ion_cma";
    };
};
```
## Compile
### Enable Driver in
```
> Device Drivers > Staging drivers > Media staging drivers
[*]   Allwinner sunXi family Video Devices
<*>     Allwinner CedarX Video Engine Driver
<*>     Allwinner CedarX Ion Driver
```
### Config "DMA Contiguous Memory Allocator"
```
> Library routines
-*- DMA Contiguous Memory Allocator
*** Default contiguous memory area size: ***
(32)  Size in Mega Bytes
Selected region size (Use mega bytes value only)  --->
```
... and here we go.

## Testing (cedar-decode-test)

`test/cedar_decode_test.c` is a standalone validator for the VE + ION driver plus
the libcedarc userspace. It feeds a raw H.264 Annex-B elementary stream straight to
the VideoEngine — **no ffmpeg** — and reports decoded frames / resolution / fps,
optionally rendering to the display. If it prints a `RESULT:` line with a non-zero
frame count, the `/dev/cedar_dev` (VE) + `/dev/ion` chain is working.

### Build
Needs [libcedarc](https://github.com/aodzip/libcedarc) and libdrm. The `--be`/`--drm`
display modes also need the libcedarc fork that exports `ion_alloc_get_dmabuf_fd()`
(only required to link):
```
cd test
make            # gcc cedar_decode_test.c -lvdecoder -lcdc_base -lMemAdapter \
                #     -lVE -lvideoengine $(pkg-config --cflags --libs libdrm) -ldl -lrt -lpthread
```

### Run
```
cedar-decode-test clip.h264                 # decode only -> prints fps
cedar-decode-test clip.h264 --fb            # + render to /dev/fb0 (CPU MB32 de-tile -> RGB)
cedar-decode-test clip.h264 --fb --max 300  # stop after 300 frames
```
Expected:
```
decoder initialized (H.264); decoding to /dev/fb0...
RESULT: decoded 300 frames, 320x240, 5.21s => 57.6 fps (HW)
```

### Options
| Flag | Meaning |
|---|---|
| `--fb` | render each frame to `/dev/fb0` (CPU de-tile of the VE's MB32 YUV → RGB) |
| `--max N` | stop after N frames |
| `--loop` | loop the input file |
| `--drm` | F1C200s display extra: hand the VE's two ION buffers to the sun4i **DEFE front-end** as a zero-copy `NV12 + ALLWINNER_TILED` plane via the atomic API (HW de-tile + CSC, ~0% CPU) |
| `--be` | F1C200s display extra: de-tile into a packed-UYVY DRM dumb buffer for the **back-end** |
| `--fmt uyvy\|yvyu\|yuyv\|vyuy` | pixel order for `--be` |
| `--noscale` | present 1:1 centred instead of scaling to the panel |
| `--hold` | show one frame then freeze (for live `devmem` register poking) |

> The `--drm`/`--be` paths target the Allwinner DE1 display engine on the F1C100s/
> F1C200s and are mainly useful there; plain decode and `--fb` validate the driver
> on any supported SoC.

## Debug
### ION_IOC_ALLOC error / memory alloc fail
Increase
```
CMA_AREAS
CMA_SIZE_MBYTES
```
### Default
Report in issue.

## Userspace library
https://github.com/aodzip/libcedarc
