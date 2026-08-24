# RG40XX V Stock Firmware Application Development Guide

## Verified TF1 environment

```text
Architecture: aarch64
Ubuntu base: 22.04
Python: 3.10.12
glibc: 2.35
Kernel: 4.9.170
Display: 640 x 480 at 60 Hz
SDL video: mali
SDL renderer: opengles2
Joystick: ANBERNIC-keys
```

## Application installation

TF1 discovers a top-level shell launcher in `/mnt/mmc/Roms/APPS/`. Use a matching application folder:

```text
/mnt/mmc/Roms/APPS/
├── My_App.sh
└── My_App/
    ├── main.py
    ├── audio_worker.py
    ├── data/
    └── logs/
```

The launcher should set application-local HOME and XDG paths, change to the application directory, run Python, log output, call `sync`, and return the Python exit status. Do not install packages or modify firmware from the launcher.

## Verified inputs

```text
D-pad: hat 0, up 1, right 2, down 4, left 8
A: 0
B: 1
Y: 2
X: 3
L1: 4
R1: 5
Select: 6
Start: 7
Stick press: 9
L2: 10
R2: 11
Menu: 13
Volume Down: 15
Volume Up: 16
```

Buttons 8, 12 and 14 remain unassigned. Analogue movement is available from `/dev/input/js0`; Linux joystick packets use `<IhBB>` and axis events have base type `2`. The primary axes are 0 horizontal and 1 vertical.

## Verified audio

The internal SDL output device is `audiocodec`. The verified format is 48,000 Hz, signed 16-bit little-endian stereo with 1,024 requested samples. Use an isolated worker, queue PCM, wait for the queue to empty, pause, clear, close, and exit with `os._exit()`. Do not call global SDL shutdown in the worker.

Diagnostics includes channel, stereo, frequency-range, output-level and low-frequency resonance tests.

## Battery monitoring

Battery telemetry is read-only at:

```text
/sys/class/power_supply/axp2202-battery
```

The Battery screen must remain battery-specific. It may display:

```text
capacity
status
health
voltage_now
temp
capacity_level
present
```

Conversions:

```text
voltage_now: divide microvolts by 1,000,000
battery temp: divide tenths of a degree Celsius by 10
```

## System information

Non-battery telemetry belongs on System Information, not Battery.

USB power:

```text
/sys/class/power_supply/axp2202-usb/online
/sys/class/power_supply/axp2202-usb/present
```

System thermal zones:

```text
thermal_zone0: CPU
thermal_zone1: GPU
thermal_zone2: video engine
thermal_zone3: DDR
```

Thermal-zone values are millidegrees Celsius and are divided by 1,000. The redesigned System Information screen combines runtime details, SDL/input details, USB power state, current power source and CPU, GPU, video-engine and DDR thermals. Battery temperature remains on Battery.

## Storage and safety

The APPS partition is VFAT. Do not rely on Unix ownership, executable metadata, symbolic links, case-sensitive names or POSIX atomic behaviour. Store writable files under `data`, `config` or `logs`, and run `sync` after installation and important writes.

Keep source modules cohesive. Add a new source file only for a substantial separate subsystem, such as the isolated audio worker.
