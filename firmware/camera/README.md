# Camera firmware

The camera is the monitoring system's sensor and alarm unit. It records motion-triggered images and audio for the server, provides live video directly to the LCD viewer over Wi-Fi, and plays audio sent from the web dashboard. An optional firmware profile recognizes enrolled household members on the camera.

## Hardware and its role

These are the firmware's configured connections. The camera and the LCD viewer use separate ESP32 boards. Confirm the camera board revision, sensor, Flash capacity, PSRAM type, and wiring before choosing a build or flashing.

| Hardware | Role and firmware configuration |
| --- | --- |
| ESP32-S3 and camera sensor | The firmware uses the ESP32-S3-EYE pin map. A compatible sensor captures 1920 × 1080 JPEGs for events and 320 × 240 JPEGs for live preview. The driver uses sensor-specific standby only when it detects an OV5640. |
| PIR motion sensor | An active-high signal on GPIO 1 starts event capture after about 30 seconds of warm-up and an inactive reading. |
| Analog light sensor and white light | The reading on GPIO 2 determines whether to turn on the white light on GPIO 21 during capture. Calibrate the threshold for the sensor and installation. |
| INMP441 microphone | Records event audio at 16 kHz in mono. I2S0 uses GPIO 39 for BCLK, 40 for WS, and 41 for data in. The microphone runs at 3.3 V on the left slot. |
| MAX98357 amplifier and speaker | Play browser talkback audio. I2S1 uses GPIO 14 for BCLK, 38 for WS, and 42 for data out. |
| Red alarm light and buzzer | GPIO 48 and GPIO 47 drive the configured alarm outputs when the camera receives a `suspicious` result. Use suitable external drivers and power for the loads. |
| PSRAM and optional face storage | PSRAM holds camera frame buffers, copied JPEGs, and event audio. The identity profile uses a dedicated `face_store` Flash partition to retain local face templates across restarts. |

For the full sensor pin map, see [camera_pins.h](main/camera_pins.h). For peripheral pins and timing, see [project_settings.h](main/project_settings.h).

## How the camera operates

- **Event capture.** The camera attempts up to five FHD JPEGs and records about four seconds of audio. It sends photos in order, followed by the WAV recording, over HTTPS. The preview profile uses the V1 JPEG endpoint. The identity profile sends the original JPEG and identity metadata together using V2.
- **Live preview.** The LCD opens one authenticated session over HTTP on camera port 80 and receives QVGA MJPEG directly on port 81. The lease lasts 30 seconds and can be renewed. Event capture and local identity work pause preview. A still-valid session can resume while upload and server analysis continue.
- **Optional identity.** An assigned camera running the identity profile can receive enrollment and revocation commands from the server. It collects three consistent face samples at distinct poses, then confirms enrollment after saving the template locally. The server controls family exemptions, which are disabled by default.
- **Talkback.** Either build profile can play browser audio through the camera speaker.

## Configure the camera

Register and assign the camera in the [server dashboard](../../server/README.md). From the repository root, create the local configuration if it is absent.

```powershell
if (-not (Test-Path 'firmware/camera/main/project_local.h')) {
    Copy-Item 'firmware/camera/main/project_local.example.h' 'firmware/camera/main/project_local.h'
}
```

Edit the Git-ignored `firmware/camera/main/project_local.h`.

| Setting | Value |
| --- | --- |
| `PROJECT_WIFI_SSID`, `PROJECT_WIFI_PASSWORD` | Credentials for a 2.4 GHz Wi-Fi network. For preview, the LCD and camera must be able to reach each other on the local network. |
| `PROJECT_DEVICE_ID`, `PROJECT_DEVICE_TOKEN` | The exact device ID registered on the server and the upload token issued when the device was assigned. The camera uses this token for server requests. |
| `PROJECT_UPLOAD_URL` | A device-reachable URL such as `https://your-server.example/api/device/upload`. Use this suffix for both profiles. The identity firmware derives the V2 URL from it. |
| `PROJECT_PREVIEW_TOKEN` | An independent random pairing token using 24–128 URL-safe ASCII characters. Enter the same token on the [LCD viewer](../lcd_viewer/README.md). |
| `PROJECT_DEEP_SLEEP` | Keep this at `0` for on-demand preview. Setting it to `1` enables PIR-wake operation and disables the LAN preview service. |

Server requests require working network time (SNTP) and an HTTPS certificate chain trusted by the firmware's built-in public CA bundle. Keep local credentials and personal media out of Git.

## Build and prepare the firmware

Use Python 3, Arduino CLI, and Arduino-ESP32 **3.3.12**. The build defaults to ESP32-S3 with octal (OPI) PSRAM. Select the Flash capacity that matches the board. The preview profile defaults to 4 MB Flash with a 3 MB application partition. The identity profile requires at least 8 MB Flash and defaults to a 6 MB application partition plus `face_store`. A 16 MB layout is also available. Windows build paths should avoid the `+` character.

Run this command from the repository root to build the 4 MB preview profile.

```powershell
python firmware/camera/tools/build_arduino.py --cli arduino-cli --flash-size 4
```

For the identity profile on an 8 MB board, prepare the ESP-DL models and verify that `human_face_detect.espdl` and `human_face_feat.espdl` exist and are nonempty in `firmware/camera/vendor/esp-dl-models/` before building. Preparation requires Python and network access and downloads model wrappers, weights, and their license from a pinned Espressif commit.

```powershell
python firmware/camera/tools/prepare_face_models.py
python firmware/camera/tools/build_arduino.py --cli arduino-cli --identity --flash-size 8
```

If Arduino CLI uses a separate SDK configuration, pass `--config-file <path-to-arduino-cli.yaml>`. Build outputs are in `firmware/camera/build-camera_preview/output/` or `firmware/camera/build-camera_identity/output/`. Verify the board, serial port, and partition layout before flashing. A merged factory image can overwrite saved configuration and face templates.

Enrollment requires an initialized `face_store`. Automatic formatting is off by default. For deliberate first-time provisioning of that partition, enable `PROJECT_FACE_ALLOW_FORMAT=1` temporarily and restore it to `0` afterward. Back up existing face data before any operation that formats the partition.

## Delivery and alarm limits

- Unsent event media is held in RAM for up to about 60 seconds of upload attempts and is lost on reset. The server's separate collection window defaults to 30 seconds from the first accepted frame. Incomplete events enter manual review when that window expires.
- After the upload phase, the camera polls the event result every three seconds for up to 60 seconds. It alarms when it receives `suspicious` during that window. A `review` response ends polling, so a later manual decision may update the dashboard without triggering the camera.
- The camera handles one event at a time. Motion requests may expire while uploads or result polling are in progress.

See the [project README](../../README.md) for the system architecture and startup order.
