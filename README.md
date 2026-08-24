# RG40XX V Stock Firmware Application Development Guide

## 1. Scope and verified environment

This guide documents application packaging and runtime behaviour directly tested on an Anbernic RG40XX V using the tested stock firmware environment, referred to here as **TF1**.

```text
Architecture:      aarch64
Operating system:  Ubuntu 22.04 base
Python:            3.10.12
glibc:             2.35
Kernel:            Linux 4.9.170
Display:           640 x 480 at 60 Hz
SDL video driver:  mali
SDL renderer:      opengles2
SDL joystick:      ANBERNIC-keys
```

Other firmware releases may use different launcher paths, libraries, mappings, display initialisation or audio handling.

## 2. TF1 application structure

The tested single-card APPS path is:

```text
/mnt/mmc/Roms/APPS
```

A folder containing only a launcher did not appear in the stock menu. A shell script placed directly in `APPS` did appear. Use a top-level launcher and a matching application folder:

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

Only include directories the application actually uses. A separate file is justified when it represents a substantial subsystem. The audio worker is separate because the verified audio shutdown path requires an isolated process.

## 3. Complete launcher

Save as `/mnt/mmc/Roms/APPS/My_App.sh`:

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

Validate and normalise the launcher:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
sed -i 's/\r$//' /mnt/mmc/Roms/APPS/My_App.sh
```

Do not run package installation, upgrades or firmware modifications from the launcher.

## 4. SDL application baseline

The tested graphical path is Python 3, Pillow-generated frames and SDL2 display/input through `ctypes`. Use:

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

Load SDL2 from the system library and fall back to the verified AArch64 location:

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

Open joystick index `0` and verify that the device name is `ANBERNIC-keys` on the tested environment.

## 5. Verified physical input mapping

The current mapping was verified through the Diagnostics application and direct testing on the console.

```text
D-pad Up:       hat 0, value 1
D-pad Right:    hat 0, value 2
D-pad Down:     hat 0, value 4
D-pad Left:     hat 0, value 8

A:              button 0
B:              button 1
Y:              button 2
X:              button 3
L1:             button 4
R1:             button 5
SELECT:         button 6
START:          button 7
Stick press:    button 9
L2:             button 10
R2:             button 11
Menu (M):       button 13
Volume Down:    button 15
Volume Up:      button 16
```

Button numbers `8`, `12` and `14` remain unassigned. Do not bind application actions to them until a physical or firmware function is verified.

The console also has a physical power button and a reset control. Their SDL button mapping has not been verified. The reset control may interrupt normal execution before an application can record an event, so it must not be assumed to behave like an SDL joystick button.

### Hat parsing

```python
hat_index = raw[12]
hat_value = raw[13]
```

Hat values may be combined for diagonals:

```text
0  centred
1  up
2  right
3  up + right
4  down
6  down + right
8  left
9  up + left
12 down + left
```

## 6. Analogue stick input

The physical stick click is `button 9`.

The Linux input inventory for the tested unit exposes `ANBERNIC-keys` through `js0` and `event1`, and advertises absolute axes. SDL button and hat events work, but analogue movement was not reliably observed through the initial SDL event parser. The Diagnostics application therefore reads movement non-blockingly from `/dev/input/js0` while continuing to use SDL for display, buttons and hats.

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

Production code must buffer partial reads, process every complete eight-byte packet, close the descriptor during shutdown and avoid blocking the SDL render loop.

The expected primary mapping remains:

```text
Axis 0: horizontal, left negative, right positive
Axis 1: vertical, up negative, down positive
```

Discover and report all available axes rather than assuming that only axes `0` and `1` exist. Axes `2` and `3` have been reported but remain physically unassigned. A practical initial dead zone is `8000`, but applications should expose calibration when precision matters.

## 7. Verified internal-speaker audio path

The internal SDL audio device is enumerated as:

```text
audiocodec
```

Verified audio parameters:

```text
Sample rate:  48,000 Hz
Format:       0x8010, signed 16-bit little-endian
Channels:     2
SDL samples:  1,024 frames
Buffer size:  4,096 bytes
Silence byte: 0
```

The tested 120 ms, 440 Hz tone at 2% waveform amplitude was audible through the internal speaker.

Use an isolated audio worker with this sequence:

1. Initialise SDL audio.
2. Enumerate output devices.
3. Select the name beginning with `audiocodec`.
4. Open the device with the verified desired format.
5. Queue PCM audio.
6. Unpause playback.
7. Wait until the queued byte count reaches zero or a parent-side timeout expires.
8. Pause and clear the queue.
9. Close the device.
10. Exit the worker with `os._exit()`.

Do not call global `SDL_Quit()` inside the isolated audio worker. That call blocked in testing after the audio device had closed. The graphical parent can still perform normal SDL video and input cleanup.

Do not use blocking `aplay` from the graphical application. The tested approach became unresponsive and required a forced power-off.

## 8. Diagnostics reference application

The companion `Diagnostics` package demonstrates the tested application format:

```text
/mnt/mmc/Roms/APPS/
├── Diagnostics.sh
└── Diagnostics/
    ├── main.py
    ├── audio_worker.py
    ├── data/
    └── logs/
```

It includes:

- Fullscreen SDL2/Pillow interface
- Physical-layout button display
- D-pad hat display
- Direct `/dev/input/js0` analogue monitoring
- Axis current/minimum/maximum tracking
- Unknown-button discovery
- Input report export
- Verified 440 Hz and 880 Hz audio tests
- Left, right and both-channel audio sequence
- Read-only system/runtime information

## 9. Installation and validation

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

## 10. Storage rules

The APPS partition is VFAT. Do not rely on:

- Unix ownership or executable metadata
- Symbolic links
- Case-sensitive filenames
- POSIX atomic filesystem behaviour

Store writable content under `config`, `data` or `logs`. Run `sync` after installation and important writes.

If a forced reset leaves the FAT partition dirty or read-only, unmount `/dev/mmcblk0p1` before running filesystem repair.

## 11. Local dependencies

Pure-Python dependencies belong under `modules/` and are exposed through `PYTHONPATH`. AArch64 shared libraries belong under `lib/` and are exposed through `LD_LIBRARY_PATH`.

Shared libraries must be compatible with:

```text
Architecture: aarch64
Python ABI:   Python 3.10
C library:    glibc 2.35 or older-compatible
```

Do not replace system SDL, glibc or vendor libraries.

## 12. Compiled application checks

For a compiled AArch64 application:

```bash
file my-app
readelf -l my-app | grep interpreter
readelf -d my-app | grep NEEDED
ldd my-app
```

AArch64 alone does not guarantee compatibility. Cross-built applications must not require a glibc newer than 2.35.

## 13. Safe development workflow

- Keep the last working package as a rollback copy.
- Change one substantial subsystem at a time.
- Validate shell and Python syntax before copying.
- Copy the launcher and matching application folder into `APPS`.
- Run `sync`.
- Launch from the TF1 menu.
- Inspect application-local logs.
- Confirm clean return to the stock menu.
- Use isolated workers and parent watchdogs for operations that may block.
- Keep source modules cohesive and avoid unnecessary one-function files.

Do not initially replace `dmenu.bin`, edit stock launcher scripts, install into `/mnt/vendor`, stop the stock menu process, write directly to `/dev/fb0`, or hardcode an evdev event number when a device can be identified by name.

## 14. Remaining unknowns

- Exact official firmware version represented by the tests
- Custom menu icon filename and resource format
- Functions of buttons `8`, `12` and `14`
- SDL mapping of the physical power button
- Whether the reset control produces any recordable input event before resetting
- Physical purpose of reported axes `2` and `3`
- HDMI audio behaviour from custom APPS
- Whether the internal physical speaker path preserves stereo separation
- Whether global SDL audio shutdown can be made reliable without an isolated worker

## 15. Verified application stack

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
```

Use `/mnt/mmc/Roms/APPS/My_App.sh` as the visible TF1 entry and `/mnt/mmc/Roms/APPS/My_App/` for code, assets, settings, data and logs.
