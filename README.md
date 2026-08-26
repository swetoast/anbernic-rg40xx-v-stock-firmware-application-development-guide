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
Pillow:             9.0.1
glibc:              2.35
Kernel:             Linux 4.9.170
Display surface:    640 x 480
SDL video driver:   mali
SDL renderer:       opengles2
SDL joystick:       ANBERNIC-keys
```

### Pillow compatibility

TF1 uses Pillow 9.0.1. Do not require APIs introduced by newer Pillow releases.

In particular, direct use of `Image.Resampling.LANCZOS` is not compatible with the tested environment. Use a compatibility selector:

```python
from PIL import Image

RESAMPLE_LANCZOS = getattr(
    getattr(Image, "Resampling", Image),
    "LANCZOS",
    getattr(Image, "LANCZOS", Image.BICUBIC),
)
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
    ├── logs/
    └── tests/
```

Include only the directories required by the application.

### Directory roles

- `My_App.sh`: launcher shown by the TF1 menu.
- `main.py`: application entry point.
- `audio_worker.py`: optional isolated SDL audio process.
- `app/`: substantial application modules.
- `assets/`: packaged images, sounds and other local assets.
- `modules/`: application-local pure-Python dependencies.
- `lib/`: application-local AArch64 shared libraries.
- `config/`: persistent settings.
- `data/`: persistent saves, durable state and intentional caches.
- `logs/`: bounded runtime logs.
- `tests/`: consolidated regression tests where the application includes them.

Use `/tmp` for render frames, scratch files and session-only state. Do not write high-frequency generated frames into `data/`.

Keep the source structure cohesive. Add a new file only when it represents a substantial separate subsystem. The audio worker is separate because audio operations must not be allowed to block the graphical process.

## 3. Recommended launcher

Save the launcher as:

```text
/mnt/mmc/Roms/APPS/My_App.sh
```

```bash
#!/bin/bash

APP_DIR="/mnt/mmc/Roms/APPS/My_App"
PYTHON="/usr/bin/python3"
PERSISTENT_DATA="$APP_DIR/data"
PERSISTENT_CONFIG="$APP_DIR/config"
PERSISTENT_LOGS="$APP_DIR/logs"
TEMP_ROOT="/tmp/My_App"
LOCK_DIR="/tmp/My_App.lock"
LOCK_PID="$LOCK_DIR/pid"

can_write_dir() {
    directory="$1"
    test_file="$directory/.write-test.$$"
    [ -d "$directory" ] || return 1
    : > "$test_file" 2>/dev/null || return 1
    rm -f "$test_file" 2>/dev/null
}

acquire_lock() {
    if mkdir "$LOCK_DIR" 2>/dev/null; then
        printf '%s\n' "$$" > "$LOCK_PID"
        return 0
    fi

    old_pid=""
    [ -r "$LOCK_PID" ] && old_pid="$(cat "$LOCK_PID" 2>/dev/null)"
    case "$old_pid" in
        ''|*[!0-9]*) ;;
        *)
            if kill -0 "$old_pid" 2>/dev/null; then
                return 1
            fi
            ;;
    esac

    rm -rf "$LOCK_DIR" 2>/dev/null || return 1
    mkdir "$LOCK_DIR" 2>/dev/null || return 1
    printf '%s\n' "$$" > "$LOCK_PID"
}

cleanup() {
    current_pid=""
    [ -r "$LOCK_PID" ] && current_pid="$(cat "$LOCK_PID" 2>/dev/null)"
    if [ "$current_pid" = "$$" ]; then
        rm -rf "$LOCK_DIR" 2>/dev/null || true
    fi
}

mkdir -p "$PERSISTENT_DATA" "$PERSISTENT_CONFIG" "$PERSISTENT_LOGS" 2>/dev/null || true
mkdir -p "$TEMP_ROOT/home" "$TEMP_ROOT/config" "$TEMP_ROOT/data" "$TEMP_ROOT/logs" || exit 1

acquire_lock || exit 0
trap cleanup EXIT INT TERM HUP

if can_write_dir "$PERSISTENT_DATA"; then
    export HOME="$PERSISTENT_DATA"
    export XDG_DATA_HOME="$PERSISTENT_DATA"
else
    export HOME="$TEMP_ROOT/home"
    export XDG_DATA_HOME="$TEMP_ROOT/data"
fi

if can_write_dir "$PERSISTENT_CONFIG"; then
    export XDG_CONFIG_HOME="$PERSISTENT_CONFIG"
else
    export XDG_CONFIG_HOME="$TEMP_ROOT/config"
fi

if can_write_dir "$PERSISTENT_LOGS"; then
    LOG_FILE="$PERSISTENT_LOGS/app.log"
else
    LOG_FILE="$TEMP_ROOT/logs/app.log"
fi

export SDL_NOMOUSE=1
export PYTHONDONTWRITEBYTECODE=1

if [ -d "$APP_DIR/modules" ]; then
    export PYTHONPATH="$APP_DIR/modules:${PYTHONPATH:-}"
fi

if [ -d "$APP_DIR/lib" ]; then
    export LD_LIBRARY_PATH="$APP_DIR/lib:${LD_LIBRARY_PATH:-}"
fi

cd "$APP_DIR" || exit 1
"$PYTHON" "$APP_DIR/main.py" >> "$LOG_FILE" 2>&1
exit $?
```

The lock prevents multiple fullscreen instances. The writable-directory checks allow the application to start with temporary state if the APPS partition is read-only.

Bound persistent log growth. Do not allow normal runtime logs to grow indefinitely.

Do not run `sync` after every ordinary application exit. Use it after installation or after an important persistent update when data has actually changed.

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

Render transient Pillow frames under `/tmp`, for example:

```text
/tmp/My_App-screen.bmp
```

Remove the transient frame during normal shutdown.

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
A:               button 0
B:               button 1
Y:               button 2
X:               button 3
L1:              button 4
R1:              button 5
Select:          button 6
Start:           button 7
Menu Hold:       button 8
Stick press:     button 9
L2:              button 10
R2:              button 11
Menu short:      button 13
Volume Down:     button 15
Volume Up:       button 16
```

Button `13` is the normal short Menu press. TF1 emits button `8` as a separate Menu Hold event. Applications should preserve short button `13` presses as ordinary input.

If button `8` is used for navigation, do not assume a conventional paired down/up lifetime. Remove it from the application's held-button state after handling the navigation event.

Buttons `12` and `14` remain physically unassigned. Do not bind actions to them until a physical or firmware function is verified.

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

### Test activation

If a button press starts an input test, do not count that same press as test data. Arm the test on button down and begin capture after the activation button is released.

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
- Enumerate available axes rather than assuming only axes `0` and `1` exist.

### Verified primary axes

```text
Axis 0: horizontal
  Left:  negative
  Right: positive

Axis 1: vertical
  Up:    negative
  Down:  positive
```

Axes `2` and `3` have been reported but remain physically unassigned.

A practical initial dead zone is:

```python
DEAD_ZONE = 8000
```

Applications requiring precise analogue input should support calibration instead of treating this value as universal.

For drift, range and circularity measurements, use a controlled sample interval independent of the render-loop speed. A 50 ms interval is used by the tested Diagnostics implementation.

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

The audio device accepts two-channel PCM. This does not prove the presence of two physical speakers or preserved acoustic stereo separation. The tested console has one visible front speaker grille. Left-only, right-only and combined signals remain useful for checking routing and mixing behaviour.

### Required worker sequence

Use an isolated audio worker:

1. Initialise SDL audio.
2. Enumerate output devices.
3. Select the device whose name begins with `audiocodec`.
4. Open it with the verified desired format.
5. Queue PCM audio.
6. Unpause the device.
7. Wait until the queued byte count reaches zero or a worker deadline expires.
8. Pause the device.
9. Clear the queue.
10. Close the device.
11. Terminate the worker process without calling global `SDL_Quit()`.

A worker deadline prevents a stuck queue from hanging the worker. The graphical parent should also retain a watchdog and terminate an unresponsive worker.

Do not call global `SDL_Quit()` from the isolated audio worker. That call blocked during testing after the device had closed. The graphical parent process can still perform normal SDL video and input cleanup.

Do not use blocking `aplay` from the graphical application. The tested approach became unresponsive and required a forced power-off.

### Useful audio validation patterns

Useful validation patterns include:

- Left-only signal.
- Right-only signal.
- Left, right and combined output sequence.
- Frequency-range test from 100 Hz through 16 kHz.
- Output-level progression from 1% through 25% waveform amplitude.
- Low-frequency resonance test from 60 Hz through 315 Hz.
- Optional logging of enumerated devices, obtained format and playback result.

Use neutral names such as `Output sequence` or `Channel mixing check` unless physical stereo separation is verified.

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

The current Diagnostics reference view displays CPU and GPU telemetry. VE and DDR are available to applications but are not currently shown by that reference view.

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

## 11. Verified fonts and text rendering

TF1 provides these usable fonts:

```text
/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSerif.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSerif-Bold.ttf
/usr/share/fonts/TTF/DejaVuSansMono.ttf
```

All seven loaded successfully through Pillow at sizes `10`, `12`, `14`, `16`, `22` and `36` on the tested device.

Recommended UI roles:

```text
DejaVu Sans:
  Labels, descriptions, instructions and natural-language status

DejaVu Sans Bold:
  Page titles, selected menu titles and major headings

DejaVu Sans Mono:
  Coordinates, percentages, temperatures, voltages, capacity and elapsed time

DejaVu Sans Mono Bold:
  Prominent aligned measurements
```

Do not assume Liberation Sans is installed.

The font under `/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf` is preferred over `/usr/share/fonts/TTF/DejaVuSansMono.ttf` for compact UI text on the tested environment.

Measure text using the active font rather than truncating by character count. Fit titles, status badges, card values and footer text by rendered pixel width.

## 12. Joystick RGB configuration

The tested TF1 environment stores joystick-ring configuration at:

```text
/mnt/data/dmenu/mculed_attr.ini
```

The tested file is:

```text
Size:       184 bytes
Structure:  46 little-endian unsigned 32-bit words
Integrity:  word 45
```

The following fields were identified in the tested configuration:

```text
word 27    foreground red
word 28    foreground green
word 29    foreground blue
word 30    brightness
word 42    background red
word 43    background green
word 44    background blue
```

Treat effect and enabled fields as read-only unless their behaviour is separately verified.

Before editing:

- Verify the expected file size.
- Verify the integrity field.
- Reject editing when verification fails.
- Clamp editable byte-style values to `0` through `255`.
- Preserve the original bytes.
- Write and verify the complete replacement.
- Restore the original if verification fails.

Do not write unverified data directly to a live LED device endpoint.

The physical ring may not reload the file immediately. On the tested setup, changed values became visible after Diagnostics exited and TF1 reloaded the configuration.

These offsets and integrity rules are firmware-specific and must not be assumed on other releases without verification.

## 13. Offline assets

Bundle required UI assets locally. Do not depend on network-hosted fonts, icons or images.

A compact application may use a single cohesive sprite sheet instead of many small icon files. Keep readable text labels so the interface remains usable if a decorative asset cannot be loaded.

Do not bundle fonts that already exist in the verified TF1 environment unless the application requires a specific unavailable typeface.

## 14. Verified Wi-Fi and Bluetooth

### Wi-Fi

TF1 exposes two wireless interfaces:

```text
wlan0
wlan1
```

Both interfaces use the `rtl8821cs` driver. The tested environment exposes 2.4 GHz and 5 GHz Wi-Fi support.

During verification, `wlan0` was connected in managed mode on `5220 MHz`. The reported signal was `-35 dBm`, the reported transmit link rate was `434.0 MBit/s`, and Wi-Fi power saving was enabled. These are link-state values, not measured application throughput.

The active network-management stack includes:

```text
NetworkManager
wpa_supplicant
```

Available network tools include:

```text
ip
iw
iwconfig
rfkill
wpa_cli
wpa_supplicant
nmcli
```

One saved Wi-Fi profile was present during verification. The saved Wi-Fi connection reconnects after reboot on the tested device.

The default gateway was reachable during verification. A single DNS lookup check did not return a result, so DNS reliability was not established by that check.

### Bluetooth

TF1 exposes a Realtek Bluetooth controller as:

```text
hci0
```

The controller uses UART transport and reports Bluetooth HCI 4.1. It supports central and peripheral roles.

The active Bluetooth stack includes:

```text
bluetoothd
rtk_hciattach
rtl_btlpm
```

Available Bluetooth tools include:

```text
bluetoothctl
hciconfig
hcitool
rfkill
```

The tested `bluetoothctl` version supports filtered device queries using:

```text
bluetoothctl devices Paired
bluetoothctl devices Connected
bluetoothctl devices Bonded
```

The older `paired-devices` command is not supported on the tested environment.

The following accessories were simultaneously reported by BlueZ as paired, bonded and connected:

- Google Pixel Buds Pro.
- Nintendo Pro Controller.

The Nintendo Pro Controller worked correctly for normal controller input on the tested TF1 environment.

The controller can remain powered with connected accessories while not discoverable. During the connected-device verification it was powered, pairable, not discoverable and not discovering.

TF1 exposes a BlueALSA playback definition through ALSA:

```text
bluealsa
    Bluetooth Audio
```

The verified ALSA inventory also exposed the internal `audiocodec` device and the HDMI audio device.

## 15. Reference implementation

A tested reference package uses:

```text
/mnt/mmc/Roms/APPS/
├── Diagnostics.sh
└── Diagnostics/
    ├── main.py
    ├── audio_worker.py
    ├── assets/
    │   └── diagnostics-icons.png
    ├── data/
    ├── logs/
    └── tests/
        └── test_diagnostics.py
```

The reference implementation demonstrates:

- Fullscreen SDL2 and Pillow rendering.
- Physical-layout button testing.
- Separate short Menu and Menu Hold handling.
- D-pad hat testing.
- Direct `/dev/input/js0` analogue monitoring.
- Controlled analogue sampling.
- Axis range tracking.
- Unknown-button discovery.
- Expanded audio validation through an isolated worker.
- Screen and pixel test patterns.
- Battery telemetry through the AXP2202 power-supply interface.
- Runtime, USB-power and thermal telemetry.
- Local offline icon assets with a text-only fallback.
- Regression tests consolidated in one test file.

Do not retain generated frame files under `data/`. The active render path should remain under `/tmp`.

## 16. Installation and validation

From a staging directory containing the launcher and matching folder:

```bash
cp -a My_App.sh /mnt/mmc/Roms/APPS/
cp -a My_App /mnt/mmc/Roms/APPS/
sync
```

Validate the installation:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
PYTHONDONTWRITEBYTECODE=1 python3 -m py_compile /mnt/mmc/Roms/APPS/My_App/main.py
ls -lah /mnt/mmc/Roms/APPS/My_App.sh
ls -lah /mnt/mmc/Roms/APPS/My_App/
```

Use `sync` after installation. Do not force it after every routine application exit.

## 17. VFAT storage constraints

The APPS partition is VFAT.

Do not rely on:

- Unix ownership or permissions.
- Executable metadata.
- Symbolic links.
- Case-sensitive filenames.
- Fully POSIX-compliant replacement semantics.

Use `/tmp` for render frames, scratch files and session-only state.

Use `config/`, `data/` and `logs/` only for content that must persist.

Avoid unnecessary writes.

## 18. Local dependencies

Pure-Python dependencies belong under `modules/` and are added to `PYTHONPATH` by the launcher.

AArch64 shared libraries belong under `lib/` and are added to `LD_LIBRARY_PATH` by the launcher.

Shared libraries must be compatible with:

```text
Architecture:   aarch64
Python ABI:     Python 3.10
C library:      glibc 2.35 or older-compatible
```

Do not replace the system SDL, glibc or vendor libraries.

## 19. Compiled application checks

For a compiled AArch64 application:

```bash
file my-app
readelf -l my-app | grep interpreter
readelf -d my-app | grep NEEDED
ldd my-app
```

An AArch64 build is not automatically compatible with TF1. Cross-built applications must not require a glibc version newer than `2.35`.

## 20. Safe development workflow

1. Keep the last working package as a rollback copy.
2. Change one substantial subsystem at a time.
3. Validate shell and Python syntax before copying.
4. Copy the launcher and matching application folder into `APPS`.
5. Run `sync` after installation.
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

## 21. Remaining unknowns

- Exact official firmware version represented by the tests.
- Custom menu icon filename and resource format.
- Physical functions of buttons `12` and `14`.
- SDL mapping of the physical power button.
- Whether the reset control produces a recordable event before reset.
- Physical purpose of reported axes `2` and `3`.
- HDMI audio behaviour from custom applications.
- Whether the internal physical speaker path preserves stereo separation.
- Whether global SDL audio shutdown can be made reliable without an isolated worker.
- Whether joystick-ring configuration offsets differ between TF1 releases.

## 22. Verified application stack

```text
Top-level APPS shell launcher
  -> Python 3.10.12
  -> Pillow 9.0.1
  -> Pillow-generated 640 x 480 frames under /tmp
  -> SDL2
  -> mali video driver
  -> accelerated opengles2 renderer with software fallback
  -> ANBERNIC-keys buttons and D-pad through SDL
  -> short Menu on button 13
  -> Menu Hold on button 8
  -> analogue movement through /dev/input/js0 when required
  -> audiocodec internal-speaker output through an isolated worker
  -> AXP2202 battery and USB power telemetry through sysfs
  -> Linux thermal-zone telemetry through sysfs
  -> DejaVu Sans text and DejaVu Sans Mono measurements
  -> local offline UI assets
  -> rtl8821cs 2.4 GHz and 5 GHz Wi-Fi through NetworkManager and wpa_supplicant
  -> Realtek Bluetooth 4.1 through BlueZ and the UART H5 transport
  -> BlueALSA Bluetooth audio definition
```

Use `/mnt/mmc/Roms/APPS/My_App.sh` as the visible TF1 menu entry and `/mnt/mmc/Roms/APPS/My_App/` for code, assets, settings, persistent data and logs.
