---
layout: post
title: "DIY. ESP32-CAM Live Video Streaming"
subtitle: "Building an IP camera with HTTP MJPEG streaming, PSRAM optimization, and web-based interface"
date:   2026-03-01 12:00:00 -0700
categories: diy
tags: [esp32, camera, iot, mjpeg, streaming, embedded, webserver]
author: Yurii Chukhrai
---
## Introduction

The ESP32-CAM is a low-cost microcontroller board with integrated camera capability, making it ideal for IoT projects requiring video streaming. This article covers building a live streaming IP camera system using the AI-Thinker ESP32-CAM module, PlatformIO, and Arduino framework.

![0.jpg](../resources/diy/2026-03-01-esp32cam_live_streaming/images/0.jpg){: .mx-auto.d-block :}
**_Pic#1_**. Board with camera
{: style="text-align:center;"}


## Hardware Overview

The **AI-Thinker ESP32-CAM** board includes:
- **Microcontroller**: ESP32 (dual-core, 240 MHz)
- **Camera Sensor**: OV2640 or OV3660 with advanced image processing
- **Memory**: 4MB flash + PSRAM variant for dual frame buffering
- **Connectivity**: WiFi 802.11 b/g/n (integrated antenna)
- **USB Interface**: CH340 serial-to-USB converter

The PSRAM (Pseudo Static RAM) is crucial for smooth video streaming, enabling:
- Dual frame buffering for artifact-free frames
- Higher JPEG compression quality (10-12 vs 30-63 without PSRAM)
- Better performance under network congestion

![3.jpg](../resources/diy/2026-03-01-esp32cam_live_streaming/images/3.jpg){: .mx-auto.d-block :}
  **_Pic#2_**. ESP32S2-CAM board with PSRAM
  {: style="text-align:center;"}

## Architecture & Features

### MJPEG Streaming
The project implements MJPEG (Motion JPEG) streaming at the `/stream` endpoint:
- **Frame Rate**: 10-15 fps at SVGA (800×600)
- **Bandwidth**: 500 KB/s to 2 MB/s depending on motion and compression
- **Quality**: Configurable JPEG quality (10=best, 63=worst)

### Web-Based Interface
A responsive HTML5 UI embedded in `camera_index.h` provides:
- Full-screen live video preview
- Frame capture functionality
- Mobile-friendly design

### Key Endpoints
- `/` — HTML web interface
- `/stream` — MJPEG video stream (use with VLC or browsers)
- `/capture` — Single frame JPEG snapshot

## Technical Implementation

### Sensor Support
The firmware includes optimizations for multiple OV sensor variants:
- **OV3660**: Vertical flip, brightness/saturation adjustments
- **OV2640**: Standard configuration
- **OV5640**: Model-dependent adjustments

### PSRAM Optimization
The project automatically detects PSRAM and optimizes accordingly:
- Allocates dual frame buffers when PSRAM is available
- Sets JPEG quality to 10 for best image fidelity
- Applies cache fixes for PSRAM stability (`-mfix-esp32-psram-cache-issue`)

### HTTP Server Configuration
- **Framework**: ESP IDF HTTP Server
- **Max Request Header Length**: 1024 bytes
- **Streaming Protocol**: Multipart JPEG with boundary markers
- **WiFi Mode**: Static IP for reliable connectivity

## Getting Started

### Prerequisites
- PlatformIO CLI or VS Code extension
- USB-to-Serial driver (CH340)
- Python 3.6+ for esptool
- 2.4 GHz WiFi network

### Configuration Steps

![1.jpg](../resources/diy/2026-03-01-esp32cam_live_streaming/images/1.jpg){: .mx-auto.d-block :}
**_Pic#3_**. Assembled board with camera and WiFi antenna
{: style="text-align:center;"}

1. **Update WiFi credentials** in `src/main.cpp`:
```cpp
const char* ssid = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";
IPAddress local_IP(192, 168, 0, 151);
IPAddress gateway(192, 168, 0, 1);
IPAddress subnet(255, 255, 255, 0);
```

2. **Build and upload**:
```bash
pio run              # Build
pio run -t upload    # Flash to board
pio device monitor   # View serial output
```

3. **Access the camera**:
- Open browser: `http://192.168.0.151/`
- View MJPEG stream at: `http://192.168.0.151/stream`
- Capture frame: `http://192.168.0.151/capture`

![4.png](../resources/diy/2026-03-01-esp32cam_live_streaming/images/4.png){: .mx-auto.d-block :}
**_Pic#4_**. Browser interface showing live video stream
{: style="text-align:center;"}

[GitHub code](https://github.com/YuriiChukhrai/electronics/tree/main/IoT-4)


## Performance Optimization

### With PSRAM
- Dual frame buffering ensures continuous smooth playback
- JPEG quality: 10 (excellent image quality)
- Minimal frame artifacts even under network congestion

### Without PSRAM
- Single frame buffering uses less memory
- JPEG quality: 12 (still good quality)
- More efficient for bandwidth-constrained networks

### Bandwidth Considerations
- **SVGA (800×600)**: ~500 KB/s to 2 MB/s streaming bandwidth
- **QVGA (320×240)**: ~100-300 KB/s for lower resolution
- Adjust frame size in `platformio.ini` if bandwidth is limited

## Troubleshooting Guide

| Issue | Cause | Solution |
|-------|-------|----------|
| WiFi connection fails | Incorrect credentials or 5GHz network | Update SSID/password; ensure 2.4GHz WiFi |
| Camera not detected | Missing driver or board defect | Install CH340 USB driver; reset via USB |
| Stream choppy or freezes | Insufficient bandwidth or memory | Reduce JPEG quality or lower frame resolution |
| No PSRAM detected | PSRAM IC not soldered on board | Check board variant; firmware uses fallback mode |
| Static IP conflicts | Incorrect network configuration | Verify subnet and gateway match your network |

## Key Improvements Made

This project builds upon the official Espressif and Arduino-ESP32 examples with several important refinements:

1. **Refactored `app_httpd.cpp`** to prevent streaming freezing that occurred every 15 minutes in base implementations
2. **Added PSRAM auto-detection** with dynamic optimization
3. **Improved frame buffering** for 24/7 reliable operation
4. **Enhanced HTTP server configuration** for stable long-duration streams

## Performance Metrics

Under typical operating conditions:
- **Frame Rate**: 10-15 fps at SVGA (800×600)
- **Latency**: 100-300ms (network dependent)
- **Memory Usage**: ~400KB RAM without PSRAM, ~800KB with dual buffering
- **Power Consumption**: 200-400mA during active streaming

## Future Enhancements

Planned features for future versions:
- RTSP streaming support (broader device compatibility)
- Motion detection with SD card recording capability
- OTA firmware updates for remote maintenance
- Multi-camera mesh networking
- MQTT telemetry for monitoring system health
- Web-based camera settings UI for real-time adjustments

## Conclusion

The ESP32-CAM combined with PlatformIO provides an accessible and cost-effective platform for building networked video streaming applications. The combination of integrated camera hardware, embedded HTTP server, and optional PSRAM support enables smooth, reliable live video delivery suitable for diverse IoT applications.

Whether you're building a baby monitor, wildlife camera, network-connected security device, or home automation project, this implementation demonstrates practical video streaming on resource-constrained embedded systems. The modular design allows easy customization for different sensor types, resolutions, and network conditions.

The refactored code ensures stable long-duration operation, a critical requirement for always-on surveillance and monitoring applications.

## References

- [GitHub code](https://github.com/YuriiChukhrai/electronics/tree/main/IoT-4)
- [Arduino-ESP32 GitHub Repository](https://github.com/espressif/arduino-esp32/)
- [Espressif ESP32 Camera Examples](https://github.com/espressif/arduino-esp32/tree/master/libraries/ESP32/examples/Camera)
- [PIO-ESP32CAM Platform Configuration](https://github.com/maxgerhardt/pio-esp32cam)
- [Espressif IoT Development Framework](https://github.com/espressif/esp-idf)
- [PlatformIO Documentation](https://docs.platformio.org/)
- [AI-Thinker ESP32-CAM Product Page](https://www.ai-thinker.com/)
- [OmniVision OV2640 Sensor Documentation](https://www.ovt.com/)
