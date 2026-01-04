# OpenWrt Camera Support Changes

Base: `v25.12.0-rc1` (OpenWrt 25.12.0-rc1)

## Project Goal

**Travel router with built-in baby monitor camera.**

A Raspberry Pi 4 running OpenWrt that serves as both a portable travel router and a crib camera. Connect to hotel WiFi, create a private network, and stream video to a phone/tablet - all from one compact device.

### Network Architecture

```
                         ┌─────────────────┐
                         │   Home Network  │
                         │  (Smart Home)   │
                         └────────┬────────┘
                                  │ WireGuard VPN
                                  │
┌──────────────┐         ┌────────┴────────┐         ┌──────────────┐
│  Hotel WiFi  │◄────────│   Raspberry Pi  │────────►│ Phone/Tablet │
│  (Internet)  │  STA    │     OpenWrt     │   AP    │   (Viewer)   │
└──────────────┘         │                 │         └──────────────┘
                         │  ┌───────────┐  │
                         │  │  Camera   │  │
                         │  │  IMX219   │  │
                         │  └───────────┘  │
                         └─────────────────┘
```

**WiFi Configuration:**
- **Built-in brcmfmac**: Client (STA) mode to hotel/upstream WiFi
- **USB mt7925 dongle**: Access Point (AP) mode for local devices (WiFi 7 capable)

**Camera Stream Path:**
- RTSP stream available locally: `rtsp://<router-ip>:8554/camera`
- Via WireGuard: Stream appears on home network VLAN, integrates with smart home

**Hardware:**
- Netgear A9000 USB WiFi adapter (Mediatek mt7925 chipset, WiFi 7, 6 GHz capable)

## Summary

Custom modifications to enable Raspberry Pi 4 + IMX219 camera support with libcamera for 1080p RTSP streaming.

**Final working solution:** `v4l2-compat.so` + FFmpeg + `h264_v4l2m2m` + MediaMTX (not GStreamer - see January 2026 update below).

---

## Commits on Top of RC1

```
f75232efc6 checkpoint full .config (local use)
49eb96679a rpi4: lock camera + media + luci; pin kernel NEW symbols
```

---

## Feed Additions

### `feeds.conf.default`
Added libcamera feed:
```
src-git libcamera https://github.com/nicholasbalasus/openwrt-feed-libcamera.git
```

---

## libcamera Package Modifications

### `feeds/libcamera/libcamera/Makefile`

| Change | Original | Modified | Reason |
|--------|----------|----------|--------|
| Version format | `v0.5.1` | `0.5.1` | OpenWrt naming convention |
| Build depends | `openssl` | `openssl`, `libevent2` | libevent2 required for `cam` utility event loop |
| Runtime depends | `libyaml`, `libgnutls`, `libstdcpp` | + `libudev-zero`, `libevent2`, `libevent2-pthreads` | libevent2-pthreads for threaded event dispatcher |
| Runtime depends | (none) | + conditional `gst1-plugins-base`, `gstreamer1-libs` | Only when `CONFIG_LIBCAMERA_GSTREAMER_SUPPORT=y` |
| cam utility | disabled | **enabled** (`-Dcam=enabled`) | Debugging tool, also used for pipe-to-ffmpeg workaround |
| strip | (default) | `false` (`-Dstrip=false`) | Prevents IPA module corruption (see below) |
| GStreamer | disabled | conditional (`-Dgstreamer=enabled/disabled`) | Build libcamerasrc plugin when GStreamer support selected |
| Install | libs only | + `/usr/bin/cam`, `/usr/lib/libcamera/ipa/*`, `/usr/lib/gstreamer-1.0/libgstlibcamera.so` | Full package with utilities and plugins |

#### Verified Technical Reasoning

**libevent2 dependency** (verified in `src/apps/cam/meson.build`):
```meson
if opt_cam.disabled() or not libevent.found()
    cam_enabled = false
    subdir_done()
endif
```
The `cam` utility uses libcamera's event loop backed by libevent. If you disable cam (`-Dcam=disabled`), libevent2 is not needed.

**GStreamer conditional deps**: The `libcamerasrc` GStreamer plugin requires gstreamer-video-1.0 at build time. The Makefile uses:
```makefile
DEPENDS:=... $(if $(CONFIG_LIBCAMERA_GSTREAMER_SUPPORT),+gst1-plugins-base +gstreamer1-libs)
```
This ensures GStreamer packages are only pulled in when the feature is enabled.

**Conditional PKG_BUILD_DEPENDS syntax** (required fix for GStreamer support):

The GStreamer plugin requires `gstreamer-video-1.0` at meson configure time. Without proper build dependencies, libcamera would fail with:
```
Run-time dependency gstreamer-video-1.0 found: NO (tried pkgconfig and cmake)
```

We needed to add `gstreamer1-libs` and `gst1-plugins-base` to PKG_BUILD_DEPENDS, but only when GStreamer support is enabled. The syntax is non-obvious:

```makefile
# WRONG - CONFIG variables not available at parse time:
PKG_BUILD_DEPENDS:= $(if $(CONFIG_OPTION),package)
ifeq ($(CONFIG_OPTION),y)
PKG_BUILD_DEPENDS += package
endif

# CORRECT - OpenWrt's native syntax (no CONFIG_ prefix):
PKG_BUILD_DEPENDS:= OPTION_NAME:package
```

Our fix:
```makefile
PKG_BUILD_DEPENDS:= \
    ... \
    LIBCAMERA_GSTREAMER_SUPPORT:gstreamer1-libs \
    LIBCAMERA_GSTREAMER_SUPPORT:gst1-plugins-base
```
Reference: mjpg-streamer uses same pattern (`MJPG_STREAMER_V4L2:v4l-utils`)

**strip=false reasoning** (verified in `src/libcamera/ipa_module.cpp:109-139`):
```cpp
Span<const uint8_t> elfLoadSymbol(Span<const uint8_t> elf, const char *symbol) {
    // ... parses ELF directly to find .dynsym section ...
    if (sHdr->sh_type == SHT_DYNSYM && !strcmp(name, ".dynsym")) {
        dynsym = sHdr;
    }
    if (dynsym == nullptr) {
        LOG(IPAModule, Error) << "ELF has no .dynsym section";
        return {};
    }
```
libcamera manually parses the `.dynsym` ELF section to find `ipaModuleInfo`. OpenWrt's `sstrip` (super strip) can corrupt this section. Regular `strip` preserves `.dynsym` (required for dynamic linking). The `-Dstrip=false` meson option prevents libcamera's build from stripping, letting OpenWrt's (now safe) `strip` handle it uniformly.

### `feeds/libcamera/libcamera/patches/001-fix-musl-v4l2-compat.patch` (NEW)

Fixes v4l2 compatibility layer for musl libc:

1. **Symbol fallbacks**: musl doesn't have `openat64`/`mmap64` (always 64-bit clean)
   ```cpp
   get_symbol(fops_.openat, "openat64");
   if (!fops_.openat)
       get_symbol(fops_.openat, "openat");
   ```

2. **DMA cache sync**: Adds `DMA_BUF_IOCTL_SYNC` for ARM cache coherency
   ```cpp
   struct dma_buf_sync sync = {
       .flags = DMA_BUF_SYNC_START | DMA_BUF_SYNC_READ,
   };
   ioctl(bufFd, DMA_BUF_IOCTL_SYNC, &sync);
   ```

> **Note**: This patch is only needed if using the v4l2 compat layer. GStreamer with `libcamerasrc` bypasses this entirely.

---

## Kernel Patches

### `target/linux/generic/hack-6.12/905-fix-dmabuf-heaps.patch` (NEW)

Fixes OpenWrt's debloat patch breaking dma-buf heaps build:

```diff
-dma-buf-objs-$(CONFIG_DMABUF_HEAPS_SYSTEM)  += system_heap.o
-dma-buf-objs-$(CONFIG_DMABUF_HEAPS_CMA)     += cma_heap.o
+obj-$(CONFIG_DMABUF_HEAPS_SYSTEM)  += system_heap.o
+obj-$(CONFIG_DMABUF_HEAPS_CMA)     += cma_heap.o
```

**Required** for libcamera buffer allocation regardless of frontend (v4l2-compat or GStreamer).

---

## Kernel Config Changes

### `target/linux/bcm27xx/config-6.12`

Key additions for camera support:
- `CONFIG_VIDEO_BCM2835_UNICAM=y` - Unicam CSI-2 receiver
- `CONFIG_VIDEO_IMX219=y` - IMX219 sensor driver
- `CONFIG_VIDEO_ISP_BCM2835=y` - Broadcom ISP
- `CONFIG_DMABUF_HEAPS=y` - DMA-BUF heap allocator
- `CONFIG_DMABUF_HEAPS_CMA=y` - CMA heap
- `CONFIG_DMABUF_HEAPS_SYSTEM=y` - System heap
- `CONFIG_MEDIA_CONTROLLER=y` - Media controller API
- `CONFIG_VIDEO_V4L2_SUBDEV_API=y` - V4L2 subdev API

### `target/linux/bcm27xx/bcm2711/config-6.12`

```
CONFIG_DMABUF_HEAPS=y
CONFIG_DMABUF_HEAPS_CMA=y
CONFIG_DMABUF_HEAPS_SYSTEM=y
```

---

## Package Selection (`.config`)

### Camera/Media
- `libcamera` with RPI VC4 pipeline, v4l2 compat (no GStreamer)
- `ffmpeg` with v4l2 support and h264_v4l2m2m encoder

### WiFi - Built-in (Client/STA)
- `kmod-brcmfmac`, `kmod-mac80211`
- `wpad-openssl` (WPA supplicant + hostapd with full encryption support)

### WiFi - USB Dongle (AP) - Netgear A9000 / mt7925
- `kmod-mt7925u`, `kmod-mt7925-firmware`
- `kmod-mt792x-usb`, `kmod-mt76-usb`
- `iw-full`, `usbutils` (debugging)

### WireGuard VPN
- `kmod-wireguard`, `wireguard-tools`
- `luci-proto-wireguard`

### Web Interface
- `luci-light`, `luci-ssl`, `uhttpd`
- Various luci-app-* and luci-mod-*

### Config Verification Commands

**Check all required packages:**
```bash
egrep '^(CONFIG_PACKAGE_(bcm27xx-gpu-fw|kmod-brcmfmac|kmod-mac80211|kmod-video-core|kmod-media-controller|kmod-video-videobuf2|kmod-i2c-mux|kmod-i2c-mux-pinctrl|kmod-mt76-usb|kmod-mt792x-usb|kmod-mt7925u|kmod-mt7925-firmware|kmod-wireguard|libcamera|ffmpeg|wireguard-tools|luci-proto-wireguard|wpad-openssl|iw-full|usbutils|luci[^=]*|uhttpd[^=]*)|CONFIG_LIBCAMERA_(PIPELINE_RPI_VC4|V4L2))=' .config | sort
```

**Check kernel config:**
```bash
KCFG="build_dir/target-*/linux-bcm27xx_bcm2711/linux-*/.config"
egrep '^(CONFIG_DEVTMPFS|CONFIG_DEVTMPFS_MOUNT|CONFIG_MEDIA_SUPPORT|CONFIG_MEDIA_CONTROLLER|CONFIG_VIDEO_DEV|CONFIG_VIDEO_V4L2_SUBDEV_API|CONFIG_V4L2_FWNODE|CONFIG_V4L2_ASYNC|CONFIG_VIDEO_BCM2835_UNICAM_LEGACY|CONFIG_VIDEO_IMX219|CONFIG_VIDEO_ISP_BCM2835|CONFIG_VIDEO_I2C|CONFIG_REGMAP_I2C|CONFIG_I2C_MUX|CONFIG_I2C_MUX_PINCTRL|CONFIG_DRM|CONFIG_DRM_KMS_HELPER|CONFIG_DRM_VC4|CONFIG_DRM_V3D|CONFIG_DMABUF_HEAPS|CONFIG_DMABUF_HEAPS_CMA|CONFIG_DMABUF_HEAPS_SYSTEM|CONFIG_WIREGUARD)=' $KCFG | sort
```

**Verify GStreamer is disabled:**
```bash
egrep -i 'gstreamer|gst1-' .config | grep '=y' || echo "GStreamer: DISABLED (good)"
```

---

## Usage

### Working Solution: v4l2-compat + FFmpeg + MediaMTX

```bash
# In /boot/config.txt (already set by dtoverlay)
dtoverlay=imx219

# Start MediaMTX RTSP server (download from GitHub releases)
./mediamtx &

# Stream 1080p H.264 via RTSP
LD_PRELOAD=/usr/libexec/libcamera/v4l2-compat.so \
  ffmpeg -f v4l2 -video_size 1920x1080 -framerate 30 -i /dev/video0 \
  -c:v h264_v4l2m2m -b:v 4M -f rtsp rtsp://localhost:8554/camera

# View from any device on the network
# VLC: rtsp://<router-ip>:8554/camera
```

### GStreamer libcamerasrc (DOES NOT WORK)

```bash
# This crashes with IPA serialization error - DO NOT USE
gst-launch-1.0 libcamerasrc ! video/x-raw,width=1920,height=1080 ! ...
# FATAL: A list of V4L2 controls requires a ControlInfoMap
```

### Legacy MMAL (limited to 720p, not recommended)

```bash
# In /boot/config.txt
dtoverlay=imx219,cam0

# Only works up to 720p due to MMAL firmware limitation
ffmpeg -f v4l2 -input_format mjpeg -video_size 1280x720 -framerate 30 \
    -i /dev/video0 -c:v copy -f rtsp rtsp://localhost:8554/cam
```

---

## Architecture Notes

### Why v4l2-compat over GStreamer? (Updated January 2026)

**Original assumption was wrong.** GStreamer's `libcamerasrc` crashes due to an upstream IPA serialization bug. The v4l2-compat layer, with our patches, is the working path.

| Aspect | v4l2-compat (WORKS) | GStreamer libcamerasrc (BROKEN) |
|--------|---------------------|----------------------------------|
| Status | **Working with patches** | Crashes (IPA serialization bug) |
| Patches needed | 3 (musl, array, buffer) | N/A - unfixable without upstream |
| H.264 encoding | Via FFmpeg h264_v4l2m2m | Would need gst-omx (not packaged) |
| RTSP output | Via MediaMTX | Would need gst-rtsp-server |

### Driver Stack

```
┌─────────────────┐
│   Application   │  (gst-launch, ffmpeg, mediamtx)
├─────────────────┤
│   libcamera     │  (userspace camera stack)
├─────────────────┤
│  Unicam + ISP   │  (kernel drivers)
├─────────────────┤
│   IMX219 I2C    │  (sensor driver)
└─────────────────┘
```

---

## Lessons Learned / Debugging Journey

### The Unicam Driver Confusion

We spent significant time trying to figure out why `VIDEO_BCM2835_UNICAM` (mainline) wasn't binding to the camera.

**The four BCM camera driver options:**

| Config | Location | Compatible String |
|--------|----------|-------------------|
| `VIDEO_BCM2835_UNICAM` | `drivers/media/platform/broadcom/` | `brcm,bcm2835-unicam-upstream` |
| `VIDEO_BCM2835_UNICAM_LEGACY` | `drivers/media/platform/bcm2835/` | `brcm,bcm2835-unicam`, `brcm,bcm2835-unicam-legacy` |
| `VIDEO_BCM2835` | staging (MMAL) | N/A (VCHIQ firmware) |
| `VIDEO_ISP_BCM2835` | staging | N/A (VCHIQ firmware) |

**The problem:** RPi's device tree declares the camera as `compatible = "brcm,bcm2835-unicam"`. But RPi's kernel patches the mainline driver to only match `"brcm,bcm2835-unicam-upstream"` to avoid conflicts with their legacy driver.

**Result:** If you enable `VIDEO_BCM2835_UNICAM`, it simply won't bind - the compatible strings don't match. You *must* use `VIDEO_BCM2835_UNICAM_LEGACY`.

**Does it matter for libcamera?** No. libcamera's `rpi/vc4` pipeline abstracts the kernel driver details and configures Media Controller internally regardless of which unicam variant is used.

### The MMAL 720p Limitation

After getting unicam working, we tried the legacy `VIDEO_BCM2835` MMAL driver (`dtoverlay=imx219,cam0`) for its built-in hardware MJPEG encoding. It worked, but:

```
bcm2835_v4l2-0: V4L2 device registered as video0 - stills mode > 1280x720
```

The MMAL driver hard-codes a **720p limit for video streaming**. Above that, it switches to "stills mode" (single-shot only). This is a fundamental limitation of the legacy firmware path.

**Solution:** Use libcamera + GStreamer for 1080p, which goes through unicam/ISP directly.

### The v4l2-compat Layer Issues (ALL FIXED)

We initially thought GStreamer was the right path, but it crashes due to an upstream IPA bug. The v4l2-compat layer, with patches, is actually the **working solution**.

**Issues encountered and fixed:**

1. **Wrong library path:** v4l2-compat.so is in `/usr/libexec/libcamera/`, not `/usr/lib/libcamera/`
   - **Fix:** Use correct path in LD_PRELOAD

2. **musl libc symbols:** musl doesn't have `openat64`/`mmap64` (it's always 64-bit clean)
   - **Fix:** `001-fix-musl-v4l2-compat.patch` - fall back to `openat`/`mmap`

3. **Black frames / cache coherency:** DMA buffers weren't being synced for CPU access on ARM
   - **Fix:** `001-fix-musl-v4l2-compat.patch` - add `DMA_BUF_IOCTL_SYNC` calls

4. **FrameDurationLimits assertion:** Control accessed as scalar but is actually an array
   - **Fix:** `002-fix-v4l2-compat-framedurationlimits-array.patch`

5. **Buffer count mismatch:** FFmpeg requested 256 buffers, kernel max is 32
   - **Fix:** `003-fix-v4l2-compat-buffer-count-limit.patch` - cap at 8

**Lesson:** The v4l2-compat layer works great with patches. GStreamer `libcamerasrc` has an unfixed upstream IPA bug.

### The `cam` Pipe-to-FFmpeg Workaround

When v4l2-compat wasn't working, we tried piping libcamera's `cam` utility output directly to ffmpeg:

```bash
cam --capture --file=- | ffmpeg -f rawvideo -pix_fmt yuyv422 -s 1920x1080 -i - ...
```

This "worked" but was a mess:
- **Color format guessing:** Tried bgra, bgr0, rgb0, yuyv422 - all produced weird results
- **Rolling/striped frames:** Timing sync issues between cam and ffmpeg
- **0.5x speed:** CPU couldn't keep up with raw frame piping + encoding
- **Rainbow artifacts:** Cache coherency issues manifesting as color corruption

We enabled the `cam` utility (`-Dcam=enabled`) specifically for this workaround. It remains useful for debugging (`cam --list`, `cam --capture=1`) even though we're not using it for streaming.

**Lesson:** Don't pipe raw video through stdout. Use proper zero-copy paths (GStreamer + DMA-BUF).

### The Stripping Issue (IPA Module Loading)

This one was subtle and caused intermittent, confusing failures.

**Background:** libcamera is not a monolithic binary. It uses **IPA (Image Processing Algorithm) modules** loaded at runtime:
```
/usr/lib/libcamera/ipa/ipa_rpi_vc4.so
```

**Verified mechanism** (from `src/libcamera/ipa_module.cpp`):

libcamera does NOT use standard `dlsym()` to find module info. Instead, it **manually parses the ELF file** to locate symbols:

```cpp
// ipa_module.cpp:289
Span<const uint8_t> info = elfLoadSymbol(data, "ipaModuleInfo");

// elfLoadSymbol() at line 109-139 does:
// 1. Parse ELF header
// 2. Find .dynsym section by iterating section headers
// 3. Look up symbol in dynamic symbol table
// 4. Return pointer to symbol data

if (sHdr->sh_type == SHT_DYNSYM && !strcmp(name, ".dynsym")) {
    dynsym = sHdr;
}
if (dynsym == nullptr) {
    LOG(IPAModule, Error) << "ELF has no .dynsym section";
    return {};
}
```

**The problem:** OpenWrt uses `sstrip` (super strip) by default, which is more aggressive than regular `strip`. While regular `strip` preserves `.dynsym` (required for dynamic linking to work at all), `sstrip` can corrupt or remove ELF sections that libcamera's manual ELF parser relies on.

**Note on signatures:** Some documentation mentions IPA "signing" - the signatures are stored in separate `.sign` files, not embedded in the ELF. The stripping issue is purely about the `.dynsym` section, not cryptographic signatures.

**Symptoms when IPA modules are over-stripped:**
- Black / green frames
- "Packet corrupt" / weird buffer sizes
- Rolling / timing weirdness
- Intermittent "it worked once" behavior
- Or outright: `"ELF has no .dynsym section"` / `"IPA module has no valid info"`

**Fixes applied:**
1. In `.config`: Changed `CONFIG_USE_SSTRIP=y` to `CONFIG_USE_STRIP=y` (standard strip preserves `.dynsym`)
2. In libcamera Makefile: Added `-Dstrip=false` to meson args (prevents double-stripping; let OpenWrt handle it)

**How to verify IPA modules are intact:**
```bash
# Check .dynsym section exists
readelf -S /usr/lib/libcamera/ipa/ipa_rpi_vc4.so | grep dynsym

# Check dynamic symbols exist
nm -D /usr/lib/libcamera/ipa/ipa_rpi_vc4.so | grep -i ipa

# Should see entries like:
# ipaModuleInfo
# ipaCreate

# If readelf shows no .dynsym or nm errors, the module was over-stripped
```

### The DMA-BUF Heaps Build Failure

libcamera requires DMA-BUF heaps for buffer allocation. OpenWrt's kernel "debloat" patches broke the heaps Makefile:

```makefile
# Broken (OpenWrt debloat)
dma-buf-objs-$(CONFIG_DMABUF_HEAPS_SYSTEM) += system_heap.o

# Fixed
obj-$(CONFIG_DMABUF_HEAPS_SYSTEM) += system_heap.o
```

Without this fix, `cma_heap` and `system_heap` never get compiled, and libcamera fails to allocate buffers at runtime.

---

## Full Configuration Reference

### OpenWrt Package Config (`.config`)

**Camera/Media stack:**
```
CONFIG_PACKAGE_libcamera=y
CONFIG_PACKAGE_kmod-video-core=y
CONFIG_PACKAGE_kmod-media-controller=y
CONFIG_PACKAGE_kmod-video-videobuf2=y
CONFIG_PACKAGE_bcm27xx-gpu-fw=y
```

**I2C (required for IMX219 sensor communication):**
```
CONFIG_PACKAGE_kmod-i2c-mux=y
CONFIG_PACKAGE_kmod-i2c-mux-pinctrl=y
```

**WiFi (for the router part):**
```
CONFIG_PACKAGE_kmod-brcmfmac=y
CONFIG_PACKAGE_kmod-mac80211=y
CONFIG_PACKAGE_wpad-basic-mbedtls=y
```

**LuCI web interface:**
```
CONFIG_PACKAGE_luci-light=y
CONFIG_PACKAGE_luci-ssl=y
CONFIG_PACKAGE_luci-app-firewall=y
CONFIG_PACKAGE_uhttpd=y
```

**libcamera options:**
```
CONFIG_LIBCAMERA_PIPELINE_RPI_VC4=y
CONFIG_LIBCAMERA_GSTREAMER_SUPPORT=y
CONFIG_LIBCAMERA_V4L2=y
```

**Stripping (critical for IPA modules):**
```
CONFIG_USE_STRIP=y
# CONFIG_USE_SSTRIP is not set
```

### Kernel Config (`target/linux/bcm27xx/config-6.12`)

**Camera drivers:**
```
CONFIG_VIDEO_BCM2835_UNICAM_LEGACY=y   # Unicam CSI-2 receiver (the one that binds!)
CONFIG_VIDEO_ISP_BCM2835=y              # Broadcom ISP
CONFIG_VIDEO_IMX219=y                   # IMX219 sensor driver
CONFIG_VIDEO_CODEC_BCM2835=y            # Hardware video codec
CONFIG_VIDEO_I2C=y
```

**Media framework:**
```
CONFIG_MEDIA_SUPPORT=y
CONFIG_MEDIA_CONTROLLER=y
CONFIG_MEDIA_CAMERA_SUPPORT=y
CONFIG_VIDEO_DEV=y
CONFIG_VIDEO_V4L2_SUBDEV_API=y
CONFIG_V4L2_FWNODE=y
CONFIG_V4L2_ASYNC=y
```

**DMA-BUF heaps (critical for buffer allocation):**
```
CONFIG_DMABUF_HEAPS=y
CONFIG_DMABUF_HEAPS_CMA=y
CONFIG_DMABUF_HEAPS_SYSTEM=y
```

**devtmpfs (required for device node creation):**
```
CONFIG_DEVTMPFS=y        # Kernel auto-creates /dev entries
CONFIG_DEVTMPFS_MOUNT=y  # Mount at boot before init
```
Without devtmpfs, `/dev/video*`, `/dev/media*`, etc. won't be created when drivers load. libudev-zero uses these to enumerate cameras.

**Video buffer support:**
```
CONFIG_VIDEOBUF2_CORE=y
CONFIG_VIDEOBUF2_V4L2=y
CONFIG_VIDEOBUF2_DMA_CONTIG=y
CONFIG_VIDEOBUF2_MEMOPS=y
```

**I2C mux (for camera I2C routing):**
```
CONFIG_I2C_MUX=y
CONFIG_I2C_MUX_PINCTRL=y
CONFIG_REGMAP_I2C=y
```

**DRM/Graphics (ISP uses DRM):**
```
CONFIG_DRM=y
CONFIG_DRM_VC4=y
CONFIG_DRM_V3D=y
CONFIG_DRM_KMS_HELPER=y
```

**VCHIQ/MMAL (firmware communication):**
```
CONFIG_BCM2835_VCHIQ=y
CONFIG_BCM2835_VCHIQ_MMAL=y
CONFIG_BCM_VIDEOCORE=y
```

### Kernel Config (`target/linux/bcm27xx/bcm2711/config-6.12`)

**New kernel prompt lockdowns (prevent build interruptions):**
```
CONFIG_SPI_PCI1XXXX=n
CONFIG_SPI_PL022=n
CONFIG_SPI_RP2040_GPIO_BRIDGE=n
CONFIG_GPIO_PL061=n
CONFIG_GPIO_PWM=n
CONFIG_MFD_RP1=n
CONFIG_FB_RPISENSE=n
CONFIG_COMMON_CLK_RP1=n
CONFIG_PWM_RP1=n
```

**I2C mux (duplicated for bcm2711):**
```
CONFIG_I2C_MUX=y
CONFIG_I2C_MUX_PINCTRL=y
```

### GPU Firmware Blobs

The build should produce these in `build_dir/target-*/linux-bcm27xx_bcm2711/`:
```
start4.elf      # Main GPU firmware
start4x.elf     # Extended GPU firmware
start4cd.elf    # Cut-down GPU firmware
fixup4.dat      # Memory fixup
fixup4x.dat
fixup4cd.dat
```

These are required for the VideoCore GPU and camera ISP to function.

---

## MediaMTX RTSP Server

[MediaMTX](https://github.com/bluenviron/mediamtx) provides the RTSP server for viewing the stream on phones/tablets.

### Installation

Download ARM64 binary from GitHub releases and place on device.

### Viewing the Stream

On any device connected to the router's network:
- **VLC**: `rtsp://<router-ip>:8554/cam`
- **iOS/Android**: Any RTSP viewer app

### Auto-start (TODO)

Create init script at `/etc/init.d/mediamtx` for automatic startup on boot.

---

## Update: January 2026 - The Working Solution

After extensive experimentation, we discovered that **GStreamer's libcamerasrc does NOT work** due to an upstream IPA serialization bug. The working solution uses **v4l2-compat + FFmpeg + hardware H.264 encoding**.

### What Works

```bash
LD_PRELOAD=/usr/libexec/libcamera/v4l2-compat.so \
  ffmpeg -f v4l2 -video_size 1920x1080 -framerate 30 -i /dev/video0 \
  -c:v h264_v4l2m2m -b:v 4M -f rtsp rtsp://localhost:8554/camera
```

- **1080p @ 30fps** via libcamera's v4l2 compatibility layer
- **Hardware H.264 encoding** via Pi's VideoCore (`h264_v4l2m2m`)
- **RTSP streaming** via MediaMTX
- **Viewable in VLC** and any standard RTSP client

### What Doesn't Work

**GStreamer libcamerasrc** crashes with IPA serialization error:
```
FATAL Serializer control_serializer.cpp:628 A list of V4L2 controls requires a ControlInfoMap
```
This is an upstream libcamera bug in the IPA isolated process communication. The GStreamer plugin triggers a code path that the v4l2-compat layer avoids.

### Revised Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      VLC / RTSP Client                  │
└───────────────────────────┬─────────────────────────────┘
                            │ RTSP (H.264)
┌───────────────────────────┴─────────────────────────────┐
│                       MediaMTX                          │
└───────────────────────────┬─────────────────────────────┘
                            │ RTSP publish
┌───────────────────────────┴─────────────────────────────┐
│                        FFmpeg                           │
│  (h264_v4l2m2m hardware encoder)                       │
└───────────────────────────┬─────────────────────────────┘
                            │ V4L2 (YUV420)
┌───────────────────────────┴─────────────────────────────┐
│              libcamera v4l2-compat.so                   │
│  (LD_PRELOAD shim layer)                               │
└───────────────────────────┬─────────────────────────────┘
                            │ libcamera API
┌───────────────────────────┴─────────────────────────────┐
│                libcamera + IPA (rpi/vc4)               │
└───────────────────────────┬─────────────────────────────┘
                            │ Kernel drivers
┌───────────────────────────┴─────────────────────────────┐
│        Unicam CSI-2  →  ISP  →  IMX219 sensor          │
└─────────────────────────────────────────────────────────┘
```

### libcamera Version Update

Updated from **0.5.1** to **0.6.0** (Raspberry Pi fork):
```makefile
PKG_SOURCE_URL:=https://github.com/raspberrypi/libcamera.git
PKG_SOURCE_DATE:=2025-12-02
PKG_SOURCE_VERSION:=f0e40f1c50bd0afe65727d6e407d0dcb42666ada
```

### Additional Patches Required

Two new patches for v4l2-compat layer bugs:

#### `002-fix-v4l2-compat-framedurationlimits-array.patch`

Fixes assertion failure when accessing `FrameDurationLimits` control:
```
Assertion failed: !isArray_ (controls.h: get: 190)
```

The `FrameDurationLimits` control is a 2-element array `[min, max]`, but the v4l2-compat code tried to access it as a scalar. The default value can be either scalar or array depending on initialization timing.

```cpp
// Before (crashes)
const int64_t duration = it->second.def().get<int64_t>();

// After (handles both cases)
const ControlValue &def = it->second.def();
int64_t duration;
if (def.isArray()) {
    Span<const int64_t> limits = def.get<Span<const int64_t>>();
    duration = limits[0];
} else {
    duration = def.get<int64_t>();
}
```

#### `003-fix-v4l2-compat-buffer-count-limit.patch`

Fixes buffer allocation failure:
```
ERROR V4L2 v4l2_videodevice.cpp:1330: Not enough buffers provided. Wanted 256, got 32
```

FFmpeg requests 256 buffers, but `VIDEO_MAX_FRAME` kernel limit is 32. Added cap:
```cpp
// Cap buffer count to avoid exceeding hardware limits
streamConfig.bufferCount = std::min(bufferCount, 8U);
```

### Regenerated Patch

#### `0001-libcamera-base-remove-support-for-libdw-and-libunwin.patch`

Regenerated for 0.6.0 (file structure changed). Removes libdw/libunwind dependencies that cause runtime errors when available at build time but not on target.

### GStreamer: Still Needed?

**For the current working solution: NO.** The v4l2-compat + FFmpeg path doesn't use GStreamer at all.

However, the libcamera package was built with `CONFIG_LIBCAMERA_GSTREAMER_SUPPORT=y` which:
- Adds `gst1-plugins-base` and `gstreamer1-libs` as dependencies
- Builds the `libgstlibcamera.so` plugin (which doesn't work due to the IPA bug)

**To reduce image size**, you could disable GStreamer support:
```
# In menuconfig: Multimedia → libcamera
CONFIG_LIBCAMERA_GSTREAMER_SUPPORT=n
```

This would remove ~5MB of GStreamer libraries from the final image.

### Auto-Start Script

Created `/etc/init.d/camera-stream`:
```bash
#!/bin/sh /etc/rc.common

START=99
USE_PROCD=1

start_service() {
    # Start MediaMTX
    procd_open_instance mediamtx
    procd_set_param command /root/mediamtx
    procd_set_param respawn
    procd_close_instance

    sleep 2

    # Start camera stream
    procd_open_instance camera
    procd_set_param env LD_PRELOAD=/usr/libexec/libcamera/v4l2-compat.so
    procd_set_param command /usr/bin/ffmpeg -f v4l2 -video_size 1920x1080 \
        -framerate 30 -i /dev/video0 -c:v h264_v4l2m2m -b:v 4M \
        -f rtsp rtsp://localhost:8554/camera
    procd_set_param respawn
    procd_close_instance
}
```

Enable with:
```bash
chmod +x /etc/init.d/camera-stream
/etc/init.d/camera-stream enable
```

---

## Complete Setup Checklist

### Camera Streaming
- [x] OpenWrt base with RPi4 support
- [x] libcamera with v4l2-compat layer (no GStreamer)
- [x] Kernel camera drivers (unicam, ISP, IMX219)
- [x] DMA-BUF heaps for buffer allocation
- [x] v4l2-compat patches for musl/array/buffer bugs
- [x] **1080p H.264 RTSP streaming working**
- [x] MediaMTX RTSP server
- [x] Auto-start init script

### Networking
- [x] Built-in WiFi (brcmfmac) for upstream/STA
- [x] USB WiFi (mt7925) for AP mode
- [x] WireGuard VPN for home network integration
- [x] LuCI web interface

### Optional/Future
- [ ] Power optimization for travel use
- [ ] Automatic WireGuard reconnection
- [ ] Camera stream health monitoring
