# libv4l-rkmpp

A V4L2 plugin that wraps [rockchip-mpp](http://opensource.rock-chips.com/wiki_Mpp) for the chromium's V4L2 video decoder/VEA (requires custom patches to enable those features).

The original idea comes from [v4l-gst](https://github.com/igel-oss/v4l-gst).

## About this fork

This fork of [JeffyCN/libv4l-rkmpp](https://github.com/JeffyCN/libv4l-rkmpp) adds **10bit video decoding support** (e.g. HEVC Main10 and VP9 Profile 2), with the decoded frames down-converted to 8bit NV12 for display.

### Background

The upstream plugin only outputs 8bit NV12. When a 10bit stream is decoded, MPP reports `MPP_FMT_YUV420SP_10BIT` (NV15) at the info-change stage, which trips an assert in `rkmpp_apply_info_change()` and kills the caller (in chromium, the whole GPU process — and after three such crashes chromium disables hardware video decoding entirely until restarted).

The RKVDEC hardware cannot dither 10bit streams down to 8bit by itself: on RK3588, `MPP_DEC_SET_OUTPUT_FORMAT` has no effect and the decoder always produces NV15 for 10bit content.

### How it works

When an info change reports a 10bit format, the plugin now:

1. Detaches the capture buffers from MPP (`MPP_DEC_SET_EXT_BUF_GROUP(NULL)`), letting MPP decode into its own internal NV15 buffers.
2. Still reports NV12 to the client, with the horizontal stride aligned to 64 pixels.
3. For every decoded frame, grabs an unused NV12 capture buffer and converts the NV15 frame into it using the RGA hardware (librga's `imcvtcolor`), so the conversion costs no CPU time (~4ms per 4K frame on RK3588, fast enough for 4K60).

8bit streams are unaffected and keep the original zero-copy path, where MPP decodes directly into the capture buffers.

Note: since the output is 8bit NV12, HDR content is displayed without tone mapping (the extra 2 bits are simply dropped by RGA).

### Requirements

* A SoC whose RGA supports NV15 input (e.g. RK3588/RK3588S with RGA3).
* [librga](https://github.com/airockchip/librga) (see Dependencies below).

## Dependencies

* [v4l-utils](https://git.linuxtv.org/v4l-utils.git) - with this patch:  
  [0001-libv4l2-Support-mmap-to-libv4l-plugin.patch](https://github.com/JeffyCN/meta-rockchip/blob/release-1.3.0_20200915/recipes-multimedia/v4l2apps/v4l-utils/0001-libv4l2-Support-mmap-to-libv4l-plugin.patch)
* [rockchip-mpp](https://github.com/rockchip-linux/mpp)
* [librga](https://github.com/airockchip/librga) - for converting 10bit frames(NV15) into NV12

## Building

```
   $ meson build
   $ meson compile -C build
```

## Quick Start

1. Install libv4l-rkmpp.so into /usr/lib/libv4l/plugins/

2. Create dummy V4L2 device files for chromium V4L2 video decoder/VEA in boot service:
```
   # echo dec > /dev/video-dec0
   # chmod 666 /dev/video-dec0
   # echo enc > /dev/video-enc0
   # chmod 666 /dev/video-enc0
```

3. Configure codec capabilities

   The codec capabilities (depends on chip spec) are configurable in device files:
```
   # cat /dev/video-dec0
   log-fps=1
   log-level=2
   type=dec
   codecs=VP8:VP9:H.264:H.265:AV1
   max-height=1920
   max-width=1080
```

4. Run with chromium browser:  
```
   export XDG_RUNTIME_DIR=/run/user/0
   chromium --no-sandbox --gpu-sandbox-start-early --ignore-gpu-blacklist
```
   This plugin is tested with [custom chromium](https://github.com/JeffyCN/meta-rockchip/tree/chromium-dunfell/dynamic-layers/recipes-browser/chromium) on rk3588 EVB.

## Limitation

1. There're a lot of chromium related hacks in it, might not work for other apps.  

## FAQ

1. MPP reports errors?  

   Try the newest [MPP](https://github.com/rockchip-linux/mpp) release branch or develop branch or the commit with the closest commit date.  

   Also test with the [mpi_dec_test](https://github.com/rockchip-linux/mpp/blob/release/test/mpi_dec_test.c) to check if the MPP works:
```
# mpi_dec_test -t 7 -i test-25fps.h264
```  

2. How to get more verbose logs?  

   For chromium, use these command line flags to change the log level: [--enable-logging --vmodule=*/media/gpu*=4](https://www.chromium.org/for-testers/enable-logging)  

   For libv4l-rkmpp, set the "LIBV4L_RKMPP_LOG_LEVEL" environment variable to change the log level. And set "LIBV4L_RKMPP_LOG_FPS" to enable logging fps.  

   For MPP, set the environment variable "mpp_debug", "rkv_h264d_debug", "mpp_dec_debug", "mpi_debug", etc. to change the modules' log levels.  

   For vpu driver, write verbose log level to "/sys/module/rk_vcodec/parameters/debug".

3. What about the performance?  

   The performance should be much the same as other MPP based decoders/encoders (e.g. mpi_dec_test and gstreamer MPP plugin).  

   And the performance would mostly related to the video's attributes (e.g. resolution and bitrate) and the vpu clock rates.

## Maintainers

* Jeffy Chen `<jeffy.chen@rock-chips.com>`
