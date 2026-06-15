# zcu102_cam_isp_demo_linux

Port of [zynqmp_cam_isp_demo_linux](https://github.com/bxinquan/zynqmp_cam_isp_demo_linux) to **ZCU102** with **IMX219** camera (Sony 8MP, MIPI CSI-2).

Linux SW stack for Camera + custom ISP/VIP pipeline on Xilinx ZynqMP (ZCU102), using linux-xlnx v2022.1, libcamera 0.0.2, and FPGA Manager overlay flow.

## Hardware

| Component | Original (bxinquan) | This port (ZCU102) |
|-----------|--------------------|--------------------|
| Board     | ZCU104 / ZCU106    | **ZCU102**         |
| Camera    | AR1335 (13MP, I2C 0x36) | **IMX219** (8MP, I2C 0x10) |
| Bayer     | SGRBG10            | **SRGGB10**        |
| Max res   | 2048x1536          | **3280x2464** (full) / 1920x1080 (1080p) |
| MIPI lanes | 2                 | **2**              |

## Software Stack

- **Kernel**: linux-xlnx xilinx-v2022.1 + IMX219 driver (built-in)
- **libcamera**: 0.0.2 + custom ZynqMP pipeline handler
- **libcamera-apps**: 1.0.2

## Directory Structure

```
.
├── linux-xlnx-xilinx-v2022.1/          # Kernel tree patches for ZCU102 + IMX219
│   ├── drivers/media/i2c/
│   │   └── imx219.c                    # IMX219 sensor driver (already in linux-xlnx)
│   └── arch/arm64/boot/dts/xilinx/     # ZCU102 device tree overlays
├── libcamera-0.0.2/                     # libcamera with custom ZynqMP pipeline handler
│   └── src/
│       ├── libcamera/pipeline/zynqmp/  # ZynqMP pipeline handler
│       └── ipa/zynqmp/                 # IPA module for IMX219 + custom ISP
├── libcamera-apps-1.0.2/               # libcamera-apps (unchanged from upstream)
├── prebuild/                            # Prebuilt binaries for ZCU102
├── scripts/                             # Helper scripts
└── Doc/                                 # Documentation and diagrams
```

## Quick Start

### 1. Build Environment

Install Xilinx SDK (xilinx-zynqmp-common-v2022.1):
```bash
source /path/to/xilinx-zynqmp-common-v2022.1/sdk.sh
```

### 2. U-Boot: Configure NFS boot

```
setenv bootargs 'earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/nfs rw nfsroot=192.168.1.10:/tftpboot/zcu102/rootfs,nolock,nfsvers=3 ip=dhcp cma=1000M'
setenv ipaddr 192.168.137.123; setenv netmask 255.255.255.0; setenv gatewayip 192.168.137.1; setenv serverip 192.168.1.10
tftpboot 0x00200000 zcu102/Image; tftpboot 0x00100000 zcu102/system.dtb
booti 0x00200000 - 0x00100000
```

> **Note**: ZCU102 uses `ttyPS0` (UART0 via J83 micro-USB) instead of `ttyPS1`.

### 3. Load PL and DTBO (FPGA Manager overlay)

```bash
mkdir -p /configfs
mount -t configfs configfs /configfs
echo 0 > /sys/class/fpga_manager/fpga0/flags
mkdir /configfs/device-tree/overlays/full
echo -n "pl.dtbo" > /configfs/device-tree/overlays/full/path
```

### 4. Configure IMX219, MIPI, ISP, VIP

```bash
# IMX219 is at I2C address 0x10, uses SRGGB10 Bayer pattern
media-ctl -d /dev/media0 --set-v4l2 '"imx219 3-0010":0[fmt:SRGGB10_1X10/1920x1080 field:none]'
media-ctl -d /dev/media0 --set-v4l2 '"a0020000.mipi_rx_to_video":0[fmt:SRGGB10_1X10/1920x1080 field:none]'
media-ctl -d /dev/media0 --set-v4l2 '"a0020000.mipi_rx_to_video":1[fmt:SRGGB10_1X10/1920x1080 field:none]'
media-ctl -d /dev/media0 --set-v4l2 '"axi:camif_ias1_axis_subsetconv":0[fmt:SRGGB10_1X10/1920x1080 field:none]'
media-ctl -d /dev/media0 --set-v4l2 '"axi:camif_ias1_axis_subsetconv":1[fmt:Y10_1X10/1920x1080]'
media-ctl -d /dev/media1 --set-v4l2 '"a0050000.xil_isp_lite":0[fmt:Y10_1X10/1920x1080 field:none]'
media-ctl -d /dev/media1 --set-v4l2 '"a0060000.xil_vip":1[fmt:UYVY8_1X16/1920x1080 field:none]'
media-ctl -d /dev/media1 --set-v4l2 '"a0070000.xil_vip":1[fmt:UYVY8_1X16/1920x1080 field:none]'
v4l2-ctl -d /dev/video1 -c low_latency_controls=2
v4l2-ctl -d /dev/video2 -c low_latency_controls=2
v4l2-ctl -d /dev/video3 -c low_latency_controls=2
```

Or use the helper script:
```bash
./scripts/setup_imx219_pipeline.sh 1080p
```

### 5. MIPI-RX V4L2 Test (IMX219)

```bash
v4l2-ctl -d /dev/video0 -c exposure=1000,analogue_gain=0x0100
v4l2-ctl -d /dev/video0 -C exposure,analogue_gain
v4l2-ctl -d /dev/video0 \
  --set-fmt-video=width=1920,height=1080,pixelformat=XY10,bytesperline=3840 \
  --stream-mmap=3 --stream-skip=30 --stream-count=60 --stream-poll \
  --stream-to=camera.raw10
```

### 6. ISP-PIPE V4L2 Test

```bash
v4l2-ctl -d /dev/video4 --set-fmt-meta XISP --stream-mmap=3 --stream-poll --stream-loop --stream-to=camera.stat &
v4l2-ctl -d /dev/video1 --set-fmt-video=width=1920,height=1080,pixelformat=YUYV --stream-mmap=3 --stream-poll --stream-to=camera1.yuyv &
v4l2-ctl -d /dev/video2 --set-fmt-video=width=1920,height=1080,pixelformat=YUYV --stream-mmap=3 --stream-poll --stream-to=camera2.yuyv &
v4l2-ctl -d /dev/video3 --set-fmt-video-out=width=1920,height=1080,pixelformat=XY10,bytesperline=3840 \
  --stream-out-mmap=3 --stream-poll --stream-from=camera.raw10
```

### 7. libcamera-cam Test

```bash
cam -c 1 -s width=1920,height=1080,pixelformat=YUYV,role=video,colorspace=rec709 --capture=10 --file
```

### 8. libcamera-apps Test

```bash
modetest -D fd4a0000.display -w 40:alpha:0  # Primary Plane transparent (Overlay Plane visible)
libcamera-hello -t 0
libcamera-hello -t 0 --ev -1.0
libcamera-hello -t 0 --shutter 33333.0 --analoggain 4.0 --awbgains 1.6,1.3 --denoise off --sharpness 0
libcamera-still --mode 1920:1080:10:P --width 1920 --height 1080 -r -o image.jpg
libcamera-still --mode 1920:1080:10:P --width 1920 --height 1080 -r -e yuv420 -o image.yuv
```

### 9. GStreamer Preview

```bash
media-ctl -d /dev/media1 --set-v4l2 '"a0060000.xil_vip":1[fmt:UYVY8_1X16/1920x1080 field:none]'
media-ctl -d /dev/media1 --set-v4l2 '"a0070000.xil_vip":1[fmt:UYVY8_1X16/1920x1080 field:none]'
gst-launch-1.0 libcamerasrc camera-name="/base/axi/i2c@a0000000/camera@10" \
  ! video/x-raw,format=YUY2,width=1920,height=1080,framerate=30/1 \
  ! kmssink bus-id=fd4a0000.display fullscreen-overlay=1 sync=false async=false
```

### 10. GStreamer Record (H.264)

```bash
media-ctl -d /dev/media1 --set-v4l2 '"a0060000.xil_vip":1[fmt:VYYUYY8_1X24/1920x1080 field:none]'
media-ctl -d /dev/media1 --set-v4l2 '"a0070000.xil_vip":1[fmt:VYYUYY8_1X24/1920x1080 field:none]'
gst-launch-1.0 libcamerasrc camera-name="/base/axi/i2c@a0000000/camera@10" \
  ! video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 \
  ! omxh264enc target-bitrate=4000 \
  ! h264parse config-interval=-1 \
  ! mp4mux ! filesink location=video.mp4
```

### 11. Debug Info

```bash
media-ctl --device /dev/media0 --print-topology
media-ctl --device /dev/media1 --print-topology

echo 0xf > /sys/class/video4linux/video0/dev_debug
echo 0xf > /sys/class/video4linux/video1/dev_debug
echo 0xf > /sys/class/video4linux/video2/dev_debug
echo 0xf > /sys/class/video4linux/video3/dev_debug
echo 0xf > /sys/class/video4linux/video4/dev_debug
echo 0xf > /sys/class/video4linux/v4l-subdev0/dev_debug
echo 0xf > /sys/class/video4linux/v4l-subdev1/dev_debug
echo 0xf > /sys/class/video4linux/v4l-subdev2/dev_debug
echo 0xf > /sys/class/video4linux/v4l-subdev3/dev_debug
echo 1 > /sys/module/videobuf2_v4l2/parameters/debug

export LIBCAMERA_IPA_MODULE_PATH=/usr/lib/libcamera
export LIBCAMERA_IPA_PROXY_PATH=/usr/libexec/libcamera
export LIBCAMERA_IPA_CONFIG_PATH=/usr/share/libcamera/ipa
export LIBCAMERA_LOG_LEVELS=*:DEBUG
```

## Key Differences from Original (AR1335 → IMX219)

| Aspect | AR1335 (original) | IMX219 (this port) |
|--------|------------------|--------------------|
| I2C address | `0x36` | `0x10` |
| Bayer pattern | `SGRBG10_1X10` | `SRGGB10_1X10` |
| Max resolution | 2048x1536 | 3280x2464 |
| 1080p mode | N/A (native 2048x1536) | 1920x1080 cropped |
| Camera name in DT | `camera@36` | `camera@10` |
| MIPI clock rate | ~750 MHz | ~456 MHz (1080p30) |
| ISP AWB gains (daylight) | ~1.0, 1.0 | ~1.6 R, 1.3 B |
| Kernel driver | `ar1335.c` | `imx219.c` (built-in) |
| Kernel config | `CONFIG_VIDEO_AR1335` | `CONFIG_VIDEO_IMX219` |

## Board: ZCU102 Notes

- **UART**: `ttyPS0` (UART0) via J83 micro-USB (NOT `ttyPS1` as in some other boards)
- **Display**: `fd4a0000.display` (DisplayPort — same base address as ZCU104)
- **MIPI CSI-2**: Xilinx MIPI CSI-2 RX Subsystem on PL (same IP as original)
- **IMX219 power**: requires XCLK 24MHz, DOVDD 1.8V, AVDD 2.8V, DVDD 1.2V

## References

- [Original zynqmp_cam_isp_demo_linux by bxinquan](https://github.com/bxinquan/zynqmp_cam_isp_demo_linux)
- [zynqmp_cam_isp_demo (HW/PL project)](https://github.com/bxinquan/zynqmp_cam_isp_demo)
- [IMX219 Datasheet (Sony)](https://www.sony-semicon.com/files/62/pdf/p-13_IMX219PQH5-C_Flyer.pdf)
- [linux-xlnx IMX219 driver](https://github.com/Xilinx/linux-xlnx/blob/master/drivers/media/i2c/imx219.c)
- [Xilinx ZCU102 Evaluation Board User Guide (UG1182)](https://docs.xilinx.com/r/en-US/ug1182-zcu102-eval-bd)

## License

MIT License — see [LICENSE](LICENSE)

Based on work by [bxinquan](https://github.com/bxinquan) (MIT License).
