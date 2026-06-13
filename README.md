# Salida

1. A linuxfrom scratch project (Using a Go bin as a userspace)
2. mp3 player
3. Linux phone (Mobile VOIP only to simplify everything?) Unlikely to get to this stage but whatever

## Hardware

**SoC:** Allwinner T113-S3
- Dual-core Cortex-A7 @ 1.2GHz
- 128MB DDR3 (integrated in package)
- HiFi4 DSP for audio offload
- Integrated audio codec (ADC/DAC, I2S, DMIC)
- USB OTG
- QFN package — no BGA
**Modem:** Quectel EC200U (LTE Cat 1, USB)
- Data only — voice and SMS handled over IP
- AT command interface
- Presents as standard network interface once connected
**Other:** LiPo + AXP209 PMIC, SPI NOR flash (boot), microSD (storage), USB-C, 3.5mm audio jack, MEMS microphone, tactile buttons.

## Software

Mainline Linux kernel with a custom board device tree. No init system, no package manager, no userspace beyond a single statically compiled Go binary. No cgo

The Go binary runs as PID 1. It mounts `/proc` and `/sys`, brings up the network interface, manages the modem via AT commands over serial, and handles all audio I/O through direct ALSA ioctls on `/dev/snd/`. No CGo — `CGO_ENABLED=0`, fully static.

```
Mainline Linux kernel + custom DTB
└── Go binary (PID 1, statically linked)
    ├── MP3 decode (go-mp3)
    ├── Audio I/O (/dev/snd/ via ioctl)
    ├── Input handling (/dev/input/event*)
    ├── Modem management (AT commands over /dev/ttyUSB*)
    ├── VoIP (Opus encode/decode, SIP or WebRTC via pion)
    ├── Messaging (Matrix or custom TCP/TLS)
    └── Networking (Go net package, DHCP, DNS)
```

## Memory

128MB is the primary constraint. Go's GC allocates before collecting, so peak usage during VoIP or network bursts can spike well above steady state. If the process is OOM-killed and it's PID 1, the kernel panics.

Two mitigations are planned: a thin C init as PID 1 that restarts the Go binary on crash, and `GOMEMLIMIT=96MiB` to keep the GC aggressive. The T113-S3 hardware watchdog will reboot the device if the process hangs.

## Boot Chain

```
Boot ROM → SPL → U-Boot → kernel + DTB → Go binary
```

OS lives on SPI NOR flash. User data on microSD. FEL mode (USB) available for recovery.

## Building
 
```sh
# Kernel (cross-compile for ARMv7)
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- t113s3_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)

# Go binary
CGO_ENABLED=0 GOARCH=arm GOARM=7 GOOS=linux go build -o handheld .
```

## Daydreaming
- A headphone jack. (Why don't phones have these anymore?!?)
- Centrialized logging
- Home screen inspired by T-UI (android launcher https://f-droid.org/packages/ohi.andre.consolelauncher/)
- Scripting language like GDScript for programming anything with access to APIs for volume, gps, flashlight, ect.
- Intuitive sandboxing. Easily see app permissions and change them perminately.
- GUI/TUI framework (Idk what to do for this)
- A browser (I have no idea how this would work with so little ram. NetSurf?)
