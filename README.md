# RG40XX V Stock Firmware Application Development Guide

This guide documents application packaging, runtime behaviour and hardware interfaces tested on an Anbernic RG40XX V using the stock firmware environment referred to here as **TF1**.

This guide distinguishes between:

- **Verified behaviour**, observed directly on the tested device.
- **Recommended practice**, based on those verified results.
- **Unresolved behaviour**, which should not be assumed by applications.

These findings apply to the tested TF1 environment. Other firmware releases may use different paths, libraries, mappings, display initialisation or audio behaviour.

## 1. Verified environment

```text
Architecture:       aarch64
Operating system:   Ubuntu 22.04 base
Python:             3.10.12
glibc:              2.35
Kernel:             Linux 4.9.170
Display surface:    640 x 480
SDL video driver:   mali
SDL renderer:       opengles2
SDL joystick:       ANBERNIC-keys
```

## 2. Application discovery and package layout

TF1 discovers application launchers placed directly in:

```text
/mnt/mmc/Roms/APPS
```

A folder containing only a launcher did not appear in the stock menu. A shell script placed directly in `APPS` did appear.

Use a top-level launcher and a matching application folder:

```text
/mnt/mmc/Roms/APPS/
├── My_App.sh
└── My_App/
    ├── main.py
    ├── audio_worker.py
    ├── app/
    ├── assets/
    ├── modules/
    ├── lib/
    ├── config/
    ├── data/
    └── logs/
```

Include only the directories required by the application.

### Directory roles

- `My_App.sh`: launcher shown by the TF1 menu.
- `main.py`: application entry point.
- `audio_worker.py`: optional isolated SDL audio process.
- `app/`: substantial application modules.
- `assets/`: packaged fonts, images and sounds.
- `modules/`: application-local pure-Python dependencies.
- `lib/`: application-local AArch64 shared libraries.
- `config/`: persistent settings.
- `data/`: saves, cache, generated frames, state and reports.
- `logs/`: runtime logs.

Keep the source structure cohesive. Add a new file only when it represents a substantial separate subsystem. The audio worker is separate because the verified audio shutdown path requires an isolated process.

## 3. Complete launcher

Save the launcher as:

```text
/mnt/mmc/Roms/APPS/My_App.sh
```

```bash
#!/bin/bash

APP_DIR="/mnt/mmc/Roms/APPS/My_App"
PYTHON="/usr/bin/python3"
DATA_DIR="$APP_DIR/data"
CONFIG_DIR="$APP_DIR/config"
LOG_DIR="$APP_DIR/logs"
LOG_FILE="$LOG_DIR/app.log"

mkdir -p "$DATA_DIR" "$CONFIG_DIR" "$LOG_DIR"

export HOME="$DATA_DIR"
export XDG_CONFIG_HOME="$CONFIG_DIR"
export XDG_DATA_HOME="$DATA_DIR"
export SDL_NOMOUSE=1

if [ -d "$APP_DIR/modules" ]; then
    export PYTHONPATH="$APP_DIR/modules:${PYTHONPATH:-}"
fi

if [ -d "$APP_DIR/lib" ]; then
    export LD_LIBRARY_PATH="$APP_DIR/lib:${LD_LIBRARY_PATH:-}"
fi

cd "$APP_DIR" || exit 1
"$PYTHON" "$APP_DIR/main.py" >> "$LOG_FILE" 2>&1
STATUS=$?

sync
exit "$STATUS"
```

Validate and normalise the launcher before installation:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
sed -i 's/\r$//' /mnt/mmc/Roms/APPS/My_App.sh
```

Do not install packages, upgrade system components or modify firmware from the launcher.

## 4. SDL2 application baseline

The tested graphical path uses:

```text
Python 3
Pillow-generated 640 x 480 frames
SDL2 loaded through ctypes
mali SDL video driver
opengles2 accelerated renderer
```

### SDL constants

```python
SDL_INIT_VIDEO = 0x00000020
SDL_INIT_JOYSTICK = 0x00000200

SDL_WINDOW_FULLSCREEN = 0x00000001
SDL_WINDOWPOS_UNDEFINED = 0x1FFF0000

SDL_RENDERER_SOFTWARE = 0x00000001
SDL_RENDERER_ACCELERATED = 0x00000002
SDL_RENDERER_PRESENTVSYNC = 0x00000004

SDL_QUIT = 0x100
SDL_JOYAXISMOTION = 0x600
SDL_JOYHATMOTION = 0x602
SDL_JOYBUTTONDOWN = 0x603
SDL_JOYBUTTONUP = 0x604
```

### Loading SDL2

```python
import ctypes
import ctypes.util

candidates = [
    ctypes.util.find_library("SDL2"),
    "libSDL2-2.0.so.0",
    "/usr/lib/aarch64-linux-gnu/libSDL2-2.0.so.0",
]

for candidate in filter(None, candidates):
    try:
        sdl = ctypes.CDLL(candidate)
        break
    except OSError:
        pass
else:
    raise RuntimeError("SDL2 library not found")
```

Open joystick index `0` and verify the reported device name. The tested device reports:

```text
ANBERNIC-keys
```

Request the accelerated renderer with present synchronisation first. If renderer creation fails, retry with the software renderer.

## 5. Verified physical input mapping

The following mapping was verified by direct testing on the console.

### D-pad

```text
Up:       hat 0, value 1
Right:    hat 0, value 2
Down:     hat 0, value 4
Left:     hat 0, value 8
```

### Buttons

```text
A:              button 0
B:              button 1
Y:              button 2
X:              button 3
L1:             button 4
R1:             button 5
Select:         button 6
Start:          button 7
Stick press:    button 9
L2:             button 10
R2:             button 11
Menu (M):       button 13
Volume Down:    button 15
Volume Up:      button 16
```

Buttons `8`, `12` and `14` remain unassigned. Do not bind application actions to those numbers until a physical or firmware function is verified.

The console also has physical power and reset controls. Their SDL mapping has not been verified. The reset control may interrupt execution before an application can record an event.

### Hat parsing

```python
hat_index = raw[12]
hat_value = raw[13]
```

Combined diagonal values are possible:

```text
0    centred
1    up
2    right
3    up + right
4    down
6    down + right
8    left
9    up + left
12   down + left
```

## 6. Analogue stick input

The physical stick press is `button 9`.

The tested Linux input inventory exposes `ANBERNIC-keys` through both `js0` and `event1`. SDL button and hat events work, but analogue movement was not reliably observed through the initial SDL event parser. A reliable implementation reads analogue movement non-blockingly from:

```text
/dev/input/js0
```

SDL continues to handle display, buttons and D-pad events.

### Linux joystick event format

Linux joystick events are eight-byte packets:

```python
import os
import struct

JS_EVENT_BUTTON = 0x01
JS_EVENT_AXIS = 0x02
JS_EVENT_INIT = 0x80

fd = os.open("/dev/input/js0", os.O_RDONLY | os.O_NONBLOCK)
packet = os.read(fd, 8)

time_ms, value, event_type, number = struct.unpack("<IhBB", packet)
base_type = event_type & ~JS_EVENT_INIT

if base_type == JS_EVENT_AXIS:
    print(number, value)
```

Production code must:

- Use non-blocking reads.
- Buffer partial reads.
- Process every complete eight-byte packet.
- Ignore or separately handle initialisation events.
- Close the descriptor during shutdown.
- Avoid blocking the SDL event and render loop.

### Verified primary axes

```text
Axis 0: horizontal
  Left:  negative
  Right: positive

Axis 1: vertical
  Up:    negative
  Down:  positive
```

Enumerate every available axis rather than assuming that only axes `0` and `1` exist. Axes `2` and `3` have been reported but remain physically unassigned.

A practical initial dead zone is:

```python
DEAD_ZONE = 8000
```

Applications requiring precise analogue input should support calibration instead of treating this value as universal.

## 7. Verified internal-speaker audio

The internal SDL output device is enumerated as:

```text
audiocodec
```

### Verified format

```text
Sample rate:    48,000 Hz
Format:         0x8010, signed 16-bit little-endian
Channels:       2
SDL samples:    1,024 frames
Buffer size:    4,096 bytes
Silence byte:   0
```

A 120 ms, 440 Hz tone at 2% waveform amplitude was audible through the internal speaker.

### Required worker sequence

Use an isolated audio worker:

1. Initialise SDL audio.
2. Enumerate output devices.
3. Select the device whose name begins with `audiocodec`.
4. Open it with the verified desired format.
5. Queue PCM audio.
6. Unpause the device.
7. Wait until the queued byte count reaches zero or a parent-side timeout expires.
8. Pause the device.
9. Clear the queue.
10. Close the device.
11. Exit the worker with `os._exit()`.

Do not call global `SDL_Quit()` from the isolated audio worker. That call blocked during testing after the device had closed. The graphical parent process can still perform normal SDL video and input cleanup.

Do not use blocking `aplay` from the graphical application. The tested approach became unresponsive and required a forced power-off.

### Useful audio validation patterns

Useful validation patterns include:

- Left-channel test.
- Right-channel test.
- Left, right and both-channel sequence.
- Frequency-range test from 100 Hz through 16 kHz.
- Output-level progression from 1% through 25% waveform amplitude.
- Low-frequency resonance test from 60 Hz through 315 Hz.
- Optional logging of enumerated devices, obtained format and playback result.

## 8. Battery telemetry

TF1 exposes battery telemetry through the AXP2202 power-supply interface:

```text
/sys/class/power_supply/axp2202-battery
```

### Verified battery attributes

```text
capacity
capacity_level
health
present
status
temp
voltage_now
```

Read these attributes without modifying them, and handle missing or temporarily unreadable values.

### Unit conversions

```text
voltage_now: microvolts; divide by 1,000,000 for volts
temp: tenths of a degree Celsius; divide by 10
```

### Safe reader

```python
from pathlib import Path


def read_sysfs(path, default="Unavailable"):
    try:
        return Path(path).read_text(encoding="utf-8").strip()
    except (OSError, UnicodeError):
        return default
```

Applications can use these attributes to display:

- Charge level.
- Charging or discharging state.
- Health.
- Voltage.
- Battery temperature.
- Capacity level.
- Battery presence.

Treat the power-supply sysfs attributes documented here as read-only.

## 9. System information, USB power and thermals

TF1 exposes USB power status and system thermal readings through standard Linux sysfs interfaces.

### USB power

```text
/sys/class/power_supply/axp2202-usb/online
/sys/class/power_supply/axp2202-usb/present
/sys/class/power_supply/axp2202-usb/voltage_now
```

### Verified thermal zones

```text
thermal_zone0: cpu_thermal_zone
thermal_zone1: gpu_thermal_zone
thermal_zone2: ve_thermal_zone
thermal_zone3: ddr_thermal_zone
thermal_zone4: axp2202-battery
```

Thermal-zone values are reported in millidegrees Celsius. Divide by `1,000` for degrees Celsius.

Applications can combine these interfaces with runtime information such as:

- Architecture, kernel, Python and C library information.
- SDL video driver and display resolution.
- Joystick name and reported input counts.
- USB power state and current power source.
- CPU, GPU, video-engine and DDR temperatures.

Battery temperature remains on the Battery screen.

## 10. Display and screen testing

The tested framebuffer is:

```text
/dev/fb0
```

### Verified framebuffer properties

```text
Visible resolution:    640 x 480
Virtual resolution:    640 x 960
Pixel depth:           32 bits
Stride:                2,560 bytes
Rotation:              0
State:                 0
Reported modes:        1280 x 1024 at 59 Hz
                       640 x 480 at 59 Hz
```

The `2,560`-byte stride is consistent with a 640-pixel row using four bytes per pixel.

The probe exposed:

```text
No DRM connector entries
No standard /sys/class/backlight device
No X11 display
No Wayland display
```

Use the verified fullscreen SDL2 path instead of writing directly to `/dev/fb0`.

### Recommended screen-test patterns

Recommended visual test patterns include:

- Solid red, green, blue, white and black.
- Greyscale steps.
- Colour bars.
- Horizontal and vertical gradients.
- One-pixel checkerboard.
- Alternating horizontal and vertical one-pixel lines.
- Outer border, inset borders, corner markers and centre crosshair.
- Cycling black, white, red, green and blue pixel-inspection fields.

One practical control scheme is:

```text
D-pad Left/Right: previous or next pattern
A:                 toggle automatic cycling
X:                 show or hide pattern labels
B:                 return to Diagnostics
```

Do not implement brightness adjustment or resolution switching until a safe control interface has been verified.

## 11. Reference implementation

A tested reference package uses:

```text
/mnt/mmc/Roms/APPS/
├── Diagnostics.sh
└── Diagnostics/
    ├── main.py
    ├── audio_worker.py
    ├── data/
    └── logs/
```

The reference implementation demonstrates:

- Fullscreen SDL2 and Pillow rendering.
- Physical-layout button testing.
- D-pad hat testing.
- Direct `/dev/input/js0` analogue monitoring.
- Axis range tracking.
- Unknown-button discovery.
- Input-state and axis-range reporting.
- Expanded audio validation and optional result logging.
- Screen and pixel test patterns.
- Battery telemetry through the AXP2202 power-supply interface.
- Runtime, USB-power and thermal telemetry.

## 12. Installation and validation

From a staging directory containing the launcher and matching folder:

```bash
cp -a My_App.sh /mnt/mmc/Roms/APPS/
cp -a My_App /mnt/mmc/Roms/APPS/
sync
```

Validate the installation:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
python3 -m py_compile /mnt/mmc/Roms/APPS/My_App/main.py
ls -lah /mnt/mmc/Roms/APPS/My_App.sh
ls -lah /mnt/mmc/Roms/APPS/My_App/
```

## 13. VFAT storage constraints

The APPS partition is VFAT. Do not rely on:

- Unix ownership.
- Unix executable metadata.
- Symbolic links.
- Case-sensitive filenames.
- POSIX atomic filesystem behaviour.

Store writable content under `config`, `data` or `logs`. Run `sync` after installation and important writes.

If a forced reset leaves the partition dirty or read-only, unmount `/dev/mmcblk0p1` before running filesystem repair.

## 14. Local dependencies

Pure-Python dependencies belong under `modules/` and are added to `PYTHONPATH` by the launcher.

AArch64 shared libraries belong under `lib/` and are added to `LD_LIBRARY_PATH` by the launcher.

Shared libraries must be compatible with:

```text
Architecture:   aarch64
Python ABI:     Python 3.10
C library:      glibc 2.35 or older-compatible
```

Do not replace the system SDL, glibc or vendor libraries.

## 15. Compiled application checks

For a compiled AArch64 application:

```bash
file my-app
readelf -l my-app | grep interpreter
readelf -d my-app | grep NEEDED
ldd my-app
```

An AArch64 build is not automatically compatible with TF1. Cross-built applications must not require a glibc version newer than `2.35`.

## 16. Safe development workflow

1. Keep the last working package as a rollback copy.
2. Change one substantial subsystem at a time.
3. Validate shell and Python syntax before copying.
4. Copy the launcher and matching application folder into `APPS`.
5. Run `sync`.
6. Launch from the TF1 menu.
7. Inspect application-local logs.
8. Confirm a clean return to the stock menu.
9. Use isolated workers and parent watchdogs for operations that may block.
10. Keep source modules cohesive and avoid unnecessary one-function files.

Do not initially:

- Replace `dmenu.bin`.
- Edit stock launcher scripts.
- Install into `/mnt/vendor`.
- Stop the stock menu process.
- Write directly to `/dev/fb0`.
- Hardcode an evdev event number when a device can be identified by name.

## 17. Remaining unknowns

- Exact official firmware version represented by the tests.
- Custom menu icon filename and resource format.
- Functions of buttons `8`, `12` and `14`.
- SDL mapping of the physical power button.
- Whether the reset control produces a recordable event before reset.
- Physical purpose of reported axes `2` and `3`.
- HDMI audio behaviour from custom applications.
- Whether the internal physical speaker path preserves stereo separation.
- Whether global SDL audio shutdown can be made reliable without an isolated worker.

## 18. Verified application stack

```text
Top-level APPS shell launcher
  -> Python 3.10.12
  -> Pillow-generated 640 x 480 frames
  -> SDL2
  -> mali video driver
  -> accelerated opengles2 renderer
  -> ANBERNIC-keys buttons and D-pad through SDL
  -> analogue movement through /dev/input/js0 when required
  -> audiocodec internal-speaker output through an isolated worker
  -> AXP2202 battery and USB power telemetry through sysfs
  -> Linux thermal-zone telemetry through sysfs
```

Use `/mnt/mmc/Roms/APPS/My_App.sh` as the visible TF1 menu entry and `/mnt/mmc/Roms/APPS/My_App/` for code, assets, settings, data and logs.
