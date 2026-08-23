# RG40XX V Stock Firmware Application Development Guide

## 1. Verified environment

This guide documents the application format and runtime behaviour directly tested on an Anbernic RG40XX V running the tested stock firmware.

```text
Architecture: aarch64
Operating-system base: Ubuntu 22.04
Python: 3.10.12
glibc: 2.35
Kernel: Linux 4.9.170
Display: 640 x 480 at 60 Hz
SDL video driver: mali
SDL renderer: opengles2
SDL joystick: ANBERNIC-keys
```

The findings apply to the tested stock firmware. Other releases may use different launcher paths, bundled libraries, input mappings, display initialisation or audio handling.

## 2. Required installed structure

The tested single-card APPS path is:

```text
/mnt/mmc/Roms/APPS
```

A folder containing only `launch.sh` did not appear in the stock menu. A shell script placed directly in `/mnt/mmc/Roms/APPS` did appear.

Use this pattern:

```text
/mnt/mmc/Roms/APPS/
├── My_App.sh
└── My_App/
    ├── main.py
    ├── audio_worker.py
    ├── app/
    │   ├── __init__.py
    │   ├── ui.py
    │   ├── input.py
    │   └── settings.py
    ├── assets/
    │   ├── images/
    │   ├── fonts/
    │   └── sounds/
    ├── modules/
    ├── lib/
    ├── config/
    ├── data/
    └── logs/
```

Only include directories the application actually needs.

Terminology used in this guide:

- **application**: the running program.
- **launcher**: the top-level `My_App.sh` menu entry.
- **application folder**: the matching `My_App/` directory.
- **application package**: the launcher plus the application folder.
- **SDL joystick**: SDL's representation of the console's built-in controls.

### Directory purposes

- `My_App.sh`: visible stock-menu entry and launcher.
- `main.py`: Python entry point.
- `audio_worker.py`: optional separate audio process, referred to in this guide as the audio worker.
- `app/`: application modules.
- `assets/`: read-only shipped images, fonts and sounds.
- `modules/`: app-local pure-Python dependencies.
- `lib/`: optional AArch64 shared libraries.
- `config/`: persistent settings.
- `data/`: saves, cache, state and reports.
- `logs/`: runtime logs.

## 3. Complete launcher sample

Save this as:

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

Validate it with:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
sed -i 's/\r$//' /mnt/mmc/Roms/APPS/My_App.sh
```

Do not run `apt install`, `pip install`, package upgrades or firmware modifications from the launcher.

## 4. Minimal graphical and controller sample

Save this as:

```text
/mnt/mmc/Roms/APPS/My_App/main.py
```

This sample creates a full-screen Pillow-generated interface through SDL2. A, X and Y update the screen. B exits to the stock menu.

```python
#!/usr/bin/env python3

from __future__ import annotations

import ctypes
import ctypes.util
import sys
from pathlib import Path

from PIL import Image, ImageDraw, ImageFont

APP_DIR = Path(__file__).resolve().parent
DATA_DIR = APP_DIR / "data"
DATA_DIR.mkdir(exist_ok=True)
SCREEN_BMP = DATA_DIR / "screen.bmp"

WIDTH = 640
HEIGHT = 480

SDL_INIT_VIDEO = 0x00000020
SDL_INIT_JOYSTICK = 0x00000200
SDL_WINDOW_FULLSCREEN = 0x00000001
SDL_WINDOWPOS_UNDEFINED = 0x1FFF0000
SDL_RENDERER_SOFTWARE = 0x00000001
SDL_RENDERER_ACCELERATED = 0x00000002
SDL_RENDERER_PRESENTVSYNC = 0x00000004
SDL_QUIT = 0x100
SDL_JOYBUTTONDOWN = 0x603

BUTTON_A = 0
BUTTON_B = 1
BUTTON_Y = 2
BUTTON_X = 3


def decode(value):
    return value.decode("utf-8", "replace") if value else None


def load_sdl():
    candidates = [
        ctypes.util.find_library("SDL2"),
        "libSDL2-2.0.so.0",
        "/usr/lib/aarch64-linux-gnu/libSDL2-2.0.so.0",
    ]

    for candidate in filter(None, candidates):
        try:
            return ctypes.CDLL(candidate)
        except OSError:
            pass

    raise RuntimeError("SDL2 library not found")


def bind_sdl(sdl):
    sdl.SDL_Init.argtypes = [ctypes.c_uint32]
    sdl.SDL_Init.restype = ctypes.c_int
    sdl.SDL_Quit.argtypes = []
    sdl.SDL_GetError.restype = ctypes.c_char_p

    sdl.SDL_CreateWindow.argtypes = [
        ctypes.c_char_p,
        ctypes.c_int,
        ctypes.c_int,
        ctypes.c_int,
        ctypes.c_int,
        ctypes.c_uint32,
    ]
    sdl.SDL_CreateWindow.restype = ctypes.c_void_p
    sdl.SDL_DestroyWindow.argtypes = [ctypes.c_void_p]

    sdl.SDL_CreateRenderer.argtypes = [
        ctypes.c_void_p,
        ctypes.c_int,
        ctypes.c_uint32,
    ]
    sdl.SDL_CreateRenderer.restype = ctypes.c_void_p
    sdl.SDL_DestroyRenderer.argtypes = [ctypes.c_void_p]

    sdl.SDL_RWFromFile.argtypes = [ctypes.c_char_p, ctypes.c_char_p]
    sdl.SDL_RWFromFile.restype = ctypes.c_void_p
    sdl.SDL_LoadBMP_RW.argtypes = [ctypes.c_void_p, ctypes.c_int]
    sdl.SDL_LoadBMP_RW.restype = ctypes.c_void_p
    sdl.SDL_FreeSurface.argtypes = [ctypes.c_void_p]
    sdl.SDL_CreateTextureFromSurface.argtypes = [ctypes.c_void_p, ctypes.c_void_p]
    sdl.SDL_CreateTextureFromSurface.restype = ctypes.c_void_p
    sdl.SDL_DestroyTexture.argtypes = [ctypes.c_void_p]

    sdl.SDL_RenderClear.argtypes = [ctypes.c_void_p]
    sdl.SDL_RenderCopy.argtypes = [
        ctypes.c_void_p,
        ctypes.c_void_p,
        ctypes.c_void_p,
        ctypes.c_void_p,
    ]
    sdl.SDL_RenderPresent.argtypes = [ctypes.c_void_p]

    sdl.SDL_PollEvent.argtypes = [ctypes.c_void_p]
    sdl.SDL_PollEvent.restype = ctypes.c_int
    sdl.SDL_Delay.argtypes = [ctypes.c_uint32]

    sdl.SDL_NumJoysticks.restype = ctypes.c_int
    sdl.SDL_JoystickOpen.argtypes = [ctypes.c_int]
    sdl.SDL_JoystickOpen.restype = ctypes.c_void_p
    sdl.SDL_JoystickClose.argtypes = [ctypes.c_void_p]


def sdl_error(sdl):
    return decode(sdl.SDL_GetError()) or "Unknown SDL error"


def get_font(size):
    candidates = [
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
        "/usr/share/fonts/truetype/liberation2/LiberationSans-Regular.ttf",
    ]

    for candidate in candidates:
        if Path(candidate).exists():
            return ImageFont.truetype(candidate, size)

    return ImageFont.load_default()


def make_screen(last_button=None):
    image = Image.new("RGB", (WIDTH, HEIGHT), (12, 18, 30))
    draw = ImageDraw.Draw(image)
    title_font = get_font(30)
    body_font = get_font(20)

    draw.rectangle((0, 0, WIDTH, 64), fill=(28, 82, 145))
    draw.text((22, 14), "RG40XX V Sample App", font=title_font, fill="white")
    draw.text(
        (24, 108),
        "Python, Pillow, SDL2 and built-in controls exposed as an SDL joystick are working.",
        font=body_font,
        fill=(215, 225, 240),
    )
    draw.text(
        (24, 162),
        f"Last button: {last_button if last_button is not None else 'none'}",
        font=body_font,
        fill=(120, 230, 155),
    )
    draw.text(
        (24, 360),
        "Press A, X or Y to update the screen.",
        font=body_font,
        fill="white",
    )
    draw.text(
        (24, 404),
        "Press B to return to the stock menu.",
        font=body_font,
        fill=(255, 195, 105),
    )
    image.save(SCREEN_BMP, "BMP")


def create_texture(sdl, renderer):
    rwops = sdl.SDL_RWFromFile(str(SCREEN_BMP).encode(), b"rb")
    if not rwops:
        raise RuntimeError(f"SDL_RWFromFile failed: {sdl_error(sdl)}")

    surface = sdl.SDL_LoadBMP_RW(rwops, 1)
    if not surface:
        raise RuntimeError(f"SDL_LoadBMP_RW failed: {sdl_error(sdl)}")

    texture = sdl.SDL_CreateTextureFromSurface(renderer, surface)
    sdl.SDL_FreeSurface(surface)

    if not texture:
        raise RuntimeError(
            f"SDL_CreateTextureFromSurface failed: {sdl_error(sdl)}"
        )

    return texture


def render(sdl, renderer, old_texture, last_button=None):
    make_screen(last_button)

    if old_texture:
        sdl.SDL_DestroyTexture(old_texture)

    texture = create_texture(sdl, renderer)
    sdl.SDL_RenderClear(renderer)
    sdl.SDL_RenderCopy(renderer, texture, None, None)
    sdl.SDL_RenderPresent(renderer)
    return texture


def main():
    sdl = load_sdl()
    bind_sdl(sdl)

    if sdl.SDL_Init(SDL_INIT_VIDEO | SDL_INIT_JOYSTICK) != 0:
        raise RuntimeError(f"SDL_Init failed: {sdl_error(sdl)}")

    window = None
    renderer = None
    joystick = None
    texture = None

    try:
        if sdl.SDL_NumJoysticks() > 0:
            joystick = sdl.SDL_JoystickOpen(0)

        if not joystick:
            raise RuntimeError("ANBERNIC-keys joystick could not be opened")

        window = sdl.SDL_CreateWindow(
            b"RG40XX V Sample App",
            SDL_WINDOWPOS_UNDEFINED,
            SDL_WINDOWPOS_UNDEFINED,
            WIDTH,
            HEIGHT,
            SDL_WINDOW_FULLSCREEN,
        )

        if not window:
            raise RuntimeError(f"SDL_CreateWindow failed: {sdl_error(sdl)}")

        renderer = sdl.SDL_CreateRenderer(
            window,
            -1,
            SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC,
        )

        if not renderer:
            renderer = sdl.SDL_CreateRenderer(
                window,
                -1,
                SDL_RENDERER_SOFTWARE,
            )

        if not renderer:
            raise RuntimeError(f"SDL_CreateRenderer failed: {sdl_error(sdl)}")

        texture = render(sdl, renderer, texture)
        event = ctypes.create_string_buffer(64)
        running = True

        while running:
            while sdl.SDL_PollEvent(ctypes.byref(event)):
                raw = event.raw
                event_type = int.from_bytes(raw[0:4], sys.byteorder)

                if event_type == SDL_QUIT:
                    running = False
                elif event_type == SDL_JOYBUTTONDOWN:
                    button = raw[12]

                    if button == BUTTON_B:
                        running = False
                    elif button in (BUTTON_A, BUTTON_X, BUTTON_Y):
                        texture = render(sdl, renderer, texture, button)

            sdl.SDL_Delay(10)

    finally:
        if texture:
            sdl.SDL_DestroyTexture(texture)
        if renderer:
            sdl.SDL_DestroyRenderer(renderer)
        if window:
            sdl.SDL_DestroyWindow(window)
        if joystick:
            sdl.SDL_JoystickClose(joystick)

        sdl.SDL_Quit()

    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## 5. Verified controller mapping

```text
D-pad Up:    hat 0, value 1
D-pad Right: hat 0, value 2
D-pad Down:  hat 0, value 4
D-pad Left:  hat 0, value 8

A:      button 0
B:      button 1
Y:      button 2
X:      button 3
L1:     button 4
R1:     button 5
SELECT: button 6
START:  button 7
M:      button 8
Stick:  button 9
L2:     button 10
R2:     button 11

Stick horizontal: axis 0
  Left: negative
  Right: positive

Stick vertical: axis 1
  Up: negative
  Down: positive
```

Use this raw hat parsing:

```python
hat_index = raw[12]
hat_value = raw[13]
```

A practical initial analogue dead zone is:

```python
DEAD_ZONE = 8000
```

Button 13 appeared alongside one M-button event and should remain unassigned. Axes 2 and 3 were reported but not associated with another visible stick.

## 6. Optional internal-speaker audio worker sample

Save this as:

```text
/mnt/mmc/Roms/APPS/My_App/audio_worker.py
```

This sample uses the verified audio format and cleanup path. The isolated worker queues one 120 ms, 440 Hz tone at 2% waveform amplitude.

```python
#!/usr/bin/env python3

from __future__ import annotations

import ctypes
import ctypes.util
import math
import os
import struct
import time

SDL_INIT_AUDIO = 0x00000010
AUDIO_S16LSB = 0x8010
SDL_AUDIO_ALLOW_ANY_CHANGE = 0x000F

SAMPLE_RATE = 48000
CHANNELS = 2
FREQUENCY = 440.0
DURATION_SECONDS = 0.12
AMPLITUDE = 0.02


class SDL_AudioSpec(ctypes.Structure):
    _fields_ = [
        ("freq", ctypes.c_int),
        ("format", ctypes.c_uint16),
        ("channels", ctypes.c_uint8),
        ("silence", ctypes.c_uint8),
        ("samples", ctypes.c_uint16),
        ("padding", ctypes.c_uint16),
        ("size", ctypes.c_uint32),
        ("callback", ctypes.c_void_p),
        ("userdata", ctypes.c_void_p),
    ]


def load_sdl():
    name = ctypes.util.find_library("SDL2") or "libSDL2-2.0.so.0"
    return ctypes.CDLL(name)


def bind_sdl(sdl):
    sdl.SDL_Init.argtypes = [ctypes.c_uint32]
    sdl.SDL_Init.restype = ctypes.c_int
    sdl.SDL_GetNumAudioDevices.argtypes = [ctypes.c_int]
    sdl.SDL_GetNumAudioDevices.restype = ctypes.c_int
    sdl.SDL_GetAudioDeviceName.argtypes = [ctypes.c_int, ctypes.c_int]
    sdl.SDL_GetAudioDeviceName.restype = ctypes.c_char_p
    sdl.SDL_OpenAudioDevice.argtypes = [
        ctypes.c_char_p,
        ctypes.c_int,
        ctypes.POINTER(SDL_AudioSpec),
        ctypes.POINTER(SDL_AudioSpec),
        ctypes.c_int,
    ]
    sdl.SDL_OpenAudioDevice.restype = ctypes.c_uint32
    sdl.SDL_QueueAudio.argtypes = [
        ctypes.c_uint32,
        ctypes.c_void_p,
        ctypes.c_uint32,
    ]
    sdl.SDL_QueueAudio.restype = ctypes.c_int
    sdl.SDL_GetQueuedAudioSize.argtypes = [ctypes.c_uint32]
    sdl.SDL_GetQueuedAudioSize.restype = ctypes.c_uint32
    sdl.SDL_PauseAudioDevice.argtypes = [ctypes.c_uint32, ctypes.c_int]
    sdl.SDL_ClearQueuedAudio.argtypes = [ctypes.c_uint32]
    sdl.SDL_CloseAudioDevice.argtypes = [ctypes.c_uint32]


def make_tone():
    frame_count = int(SAMPLE_RATE * DURATION_SECONDS)
    fade_frames = int(SAMPLE_RATE * 0.01)
    output = bytearray()

    for frame in range(frame_count):
        envelope = min(
            1.0,
            frame / max(1, fade_frames),
            (frame_count - 1 - frame) / max(1, fade_frames),
        )
        value = int(
            32767
            * AMPLITUDE
            * envelope
            * math.sin(2 * math.pi * FREQUENCY * frame / SAMPLE_RATE)
        )
        output.extend(struct.pack("<hh", value, value))

    return bytes(output)


def main():
    sdl = load_sdl()
    bind_sdl(sdl)

    if sdl.SDL_Init(SDL_INIT_AUDIO) != 0:
        os._exit(10)

    device_id = 0

    try:
        names = []
        for index in range(max(0, sdl.SDL_GetNumAudioDevices(0))):
            raw_name = sdl.SDL_GetAudioDeviceName(index, 0)
            names.append(raw_name.decode("utf-8", "replace"))

        device_name = next(
            name for name in names if name.strip().startswith("audiocodec")
        )

        desired = SDL_AudioSpec(
            SAMPLE_RATE,
            AUDIO_S16LSB,
            CHANNELS,
            0,
            1024,
            0,
            0,
            None,
            None,
        )
        obtained = SDL_AudioSpec()

        device_id = sdl.SDL_OpenAudioDevice(
            device_name.encode(),
            0,
            ctypes.byref(desired),
            ctypes.byref(obtained),
            SDL_AUDIO_ALLOW_ANY_CHANGE,
        )

        if not device_id:
            os._exit(11)

        tone = make_tone()
        tone_buffer = ctypes.create_string_buffer(tone)

        if sdl.SDL_QueueAudio(device_id, tone_buffer, len(tone)) != 0:
            os._exit(12)

        sdl.SDL_PauseAudioDevice(device_id, 0)

        deadline = time.monotonic() + 1.2
        while (
            time.monotonic() < deadline
            and sdl.SDL_GetQueuedAudioSize(device_id) > 0
        ):
            time.sleep(0.01)

        sdl.SDL_PauseAudioDevice(device_id, 1)
        sdl.SDL_ClearQueuedAudio(device_id)
        sdl.SDL_CloseAudioDevice(device_id)
        device_id = 0

        # Do not call global SDL_Quit() in this audio worker.
        os._exit(0)

    except Exception:
        if device_id:
            try:
                sdl.SDL_PauseAudioDevice(device_id, 1)
                sdl.SDL_ClearQueuedAudio(device_id)
                sdl.SDL_CloseAudioDevice(device_id)
            except Exception:
                pass

        os._exit(13)


if __name__ == "__main__":
    main()
```

Launch the audio worker without blocking the main SDL event loop:

```python
import subprocess
import sys
from pathlib import Path

APP_DIR = Path(__file__).resolve().parent

audio_process = subprocess.Popen(
    [sys.executable, str(APP_DIR / "audio_worker.py")],
    cwd=APP_DIR,
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
)
```

For a real application, prevent overlapping audio workers and add a parent-side timeout.

## 7. Verified audio behaviour

The internal device is enumerated by SDL as:

```text
audiocodec,
```

The accepted format is:

```text
Sample rate: 48,000 Hz
Format: 0x8010, signed 16-bit little-endian
Channels: 2
SDL samples: 1,024 frames
SDL buffer size: 4,096 bytes
Silence byte: 0
```

The tested 120 ms, 440 Hz, 2% amplitude tone was audible through the internal speaker.

The verified worker sequence is:

```text
Initialise SDL audio
Enumerate audiocodec
Open device
Queue PCM
Unpause
Wait for queued bytes to reach 0
Pause
Clear queue
Close device
Exit worker with os._exit(0)
```

Global `SDL_Quit()` blocked when called inside the isolated audio worker after the audio device had closed. The worker must skip that call. The graphical parent can still perform normal video and input cleanup when exiting.

Do not use `aplay` from the graphical application. The tested blocking `aplay` approach became unresponsive and required a forced power-off.

## 8. Development repository sample

On a development machine, use:

```text
my-app-project/
├── README.md
├── LICENSE
├── docs/
├── tools/
└── package/
    ├── My_App.sh
    └── My_App/
        ├── main.py
        ├── audio_worker.py
        ├── app/
        │   ├── __init__.py
        │   ├── ui.py
        │   ├── input.py
        │   └── settings.py
        ├── assets/
        │   ├── images/
        │   ├── fonts/
        │   └── sounds/
        ├── modules/
        ├── lib/
        ├── config/
        ├── data/
        └── logs/
```

Only the contents of `package/` are copied into `/mnt/mmc/Roms/APPS/`.

## 9. Installation sample

From a staging directory containing `My_App.sh` and `My_App/`:

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
- Symlinks
- Case-sensitive filenames
- POSIX atomic filesystem behaviour

Store writable content under `config`, `data` or `logs`. Run `sync` after installation and important writes.

If a forced reset leaves the FAT partition dirty or read-only, unmount `/dev/mmcblk0p1` before running `fsck.fat`.

## 11. Local dependencies

Pure-Python modules belong under `modules/` and are exposed through `PYTHONPATH`.

AArch64 shared libraries belong under `lib/` and are exposed through `LD_LIBRARY_PATH`.

AArch64 shared-library dependencies must match:

```text
Architecture: aarch64
Python ABI: Python 3.10
C library: glibc 2.35 or older-compatible
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

AArch64 alone does not guarantee compatibility. Cross-built programs must not require a glibc newer than 2.35.

## 13. Safe development workflow

1. Keep the last working package as a rollback copy.
2. Change one subsystem at a time.
3. Validate shell and Python syntax.
4. Copy the launcher and matching application folder into the APPS directory.
5. Run `sync`.
6. Launch from the stock menu.
7. Inspect app-local logs.
8. Confirm normal return to the menu.
9. Use isolated workers and parent watchdogs for risky operations.
10. Write and `fsync` small checkpoints around operations that may block.

Do not initially replace `dmenu.bin`, edit the stock launcher scripts, install into `/mnt/vendor`, stop the stock menu process, write directly to `/dev/fb0`, or hardcode paths such as `/dev/input/event1`; identify input devices by name instead.

## 14. Remaining unknowns

- Exact official firmware version represented by the tests
- Custom menu icon filename and resource format
- Purpose of button 13
- Purpose of reported axes 2 and 3
- Functions of buttons 12, 14, 15 and 16
- HDMI audio behaviour from custom APPS
- Whether the internal physical speaker path preserves stereo separation
- Whether global SDL audio shutdown can be made reliable without an isolated worker

## 15. Conclusion

The verified Python application stack is:

```text
Top-level APPS shell launcher
  -> Python 3.10.12
  -> SDL2
  -> the `mali` video driver and accelerated `opengles2` renderer
  -> ANBERNIC-keys built-in controls exposed as an SDL joystick
  -> ALSA audiocodec internal-speaker output
```

Use `/mnt/mmc/Roms/APPS/My_App.sh` as the visible entry and `/mnt/mmc/Roms/APPS/My_App/` for code, assets, settings, data and logs.
