# DIY Wireless Hi-Fi

A DIY project for turning ordinary passive speakers into **battery-powered wireless mono speakers**.

Each speaker is a self-contained unit consisting of:

- a tool battery for portable power
- a TPA3116D2 amplifier
- a Bluetooth audio receiver
- a buck converter to provide 5 V to the Bluetooth board
- a passive speaker

The two speakers can be connected to a Raspberry Pi and used for simultaneous playback. The Raspberry Pi acts as the central audio hub, while the individual speaker units remain simple and independent.

![Project hardware](https://github.com/user-attachments/assets/9f4c1b2a-bba8-4541-a774-d98e661bf29e)

---

# Hardware

## Main components

- Raspberry Pi 4B + microSD card
- 2× DollaTek DC 12V–24V TPA3116D2 100W Mono Audio Amplifier boards
- 2× Heemol VHM-314 Bluetooth 5.0 audio receiver boards
- 2× LM2596 DC-DC step-down regulator modules
- 2× tool batteries  
  - I used generic unbranded "Makita-style" 18–21 V batteries, 4 Ah
  - 2× matching DC adapters are used to connect the batteries
- 1× 3.5 mm stereo mini-jack cable, cut in half
- 2× 10 kΩ 0.25 W resistors
- Wire

The two speaker units are electrically independent. Each one has its own battery, amplifier, Bluetooth receiver and buck converter.

---

# Battery-Powered Bluetooth Mono Speaker

Each speaker converts a normal passive speaker into a portable Bluetooth speaker.

The VHM-314 Bluetooth receiver provides a **stereo** audio output, while the TPA3116D2 amplifier is being used as a **mono** amplifier. The left and right channels therefore need to be combined into one mono signal.

Two 10 kΩ resistors are used for this purpose. The resistors prevent the left and right outputs of the Bluetooth board from being directly connected to each other.

## Signal path

```text
VHM-314 Bluetooth board
(3.5 mm stereo output)
        │
        ├── L ──[10 kΩ]──┐
        │                │
        │                ├──► Amp "+" (IN)
        │                │
        ├── R ──[10 kΩ]──┘
        │
        └── GND ─────────────► Amp "−" (IN)
```

### Why the resistors are needed

The two resistors form a simple passive stereo-to-mono mixer.

Do **not** simply connect L and R together. The resistor on each channel keeps the two outputs isolated from one another while allowing their signals to be combined at the amplifier input.

---

# Power path

```text
18–21 V "Makita-style" tool battery
        │
        ├────────────────────────────► TPA3116D2 amplifier
        │                              VIN+ / VIN−
        │
        └──► LM2596 buck converter
                    │
                    └──► 5.0 V ──► VHM-314 Bluetooth board
```

The TPA3116D2 amplifier can accept the battery voltage directly because its specified supply range is 12–24 V.

The Bluetooth receiver requires a lower voltage, so the battery voltage is reduced to **5 V** using an LM2596 buck converter.

**Important:** set the LM2596 output to 5.0 V *before* connecting the Bluetooth board.

---

# Wiring table

The following connections are for **one speaker**. Build the same circuit a second time for the other speaker.

| From | To | Notes |
|---|---|---|
| Battery + / − | LM2596 IN+ / IN− | Raw 18–21 V battery voltage |
| Battery + / − | Amp VIN+ / VIN− | Direct connection; amplifier accepts 12–24 V |
| LM2596 OUT+ / OUT− | BT board 5V / GND | Adjust LM2596 to 5.0 V first |
| BT board L out | 10 kΩ resistor #1 | Other resistor leg goes to amplifier "+" input |
| BT board R out | 10 kΩ resistor #2 | Other resistor leg goes to the same amplifier "+" input |
| BT board GND | Amp "−" input | Direct connection |
| Amp speaker out + / − | Speaker | Normal two-wire speaker connection |

![Wiring](https://github.com/user-attachments/assets/b55af238-f67d-45f3-b928-9518c6c83329)

---

# Installation

## Raspberry Pi

The Raspberry Pi is used as the central audio hub. It handles Bluetooth connections and audio routing between the two wireless speakers.

### 1. Install Ubuntu Server

Flash **Ubuntu 26.04 LTS Server** to a microSD card using Raspberry Pi Imager.

During installation, enable:

- SSH
- Wi-Fi

Boot the Raspberry Pi and find its local IP address.

Then connect over SSH:

```bash
ssh username@<raspberry-pi-ip>
```

For example:

```bash
ssh pi@192.168.1.102
```

### 2. Update the system

```bash
sudo apt update
sudo apt upgrade -y
```

---

# Audio setup

This project uses **PipeWire** for audio routing.

PipeWire sits between applications and the physical/Bluetooth audio devices. It is useful here because the Raspberry Pi needs to manage multiple independent Bluetooth audio outputs.

## Check the current audio server

```bash
pactl info | grep "Server Name"
```

## Check whether PipeWire is installed

```bash
dpkg -l | grep pipewire
```

## Check the running audio services

```bash
systemctl --user list-units | grep -i 'pulse\|pipewire\|wireplumber'
```

## Install PipeWire

```bash
sudo apt-get update
sudo apt-get install -y     pipewire     pipewire-audio-client-libraries     pipewire-pulse     pipewire-jack     pipewire-alsa     libspa-0.2-bluetooth     wireplumber
```

Also make sure the Bluetooth integration package is installed:

```bash
sudo apt-get install -y libspa-0.2-bluetooth pipewire-audio
```

## Enable PipeWire

```bash
systemctl --user enable pipewire pipewire-pulse wireplumber
systemctl --user start pipewire pipewire-pulse wireplumber
```

Verify that PipeWire is now providing the audio server:

```bash
pactl info | grep "Server Name"
```

You should see something similar to:

```text
Server Name: PulseAudio (on PipeWire ...)
```

## Check PipeWire

```bash
pw-cli info
```

List PipeWire audio nodes:

```bash
pw-cli list-objects | grep -i "PipeWire:Interface:Node"
```

Or use WirePlumber's more user-friendly interface:

```bash
wpctl status
```

To see the audio devices:

```bash
wpctl status | grep -A 20 "Audio"
```

# Bluetooth setup

Bluetooth is handled by **BlueZ**, the standard Linux Bluetooth protocol stack. PipeWire/WirePlumber then handles the audio side of the Bluetooth connection.

## Install Bluetooth packages

```bash
sudo apt install bluetooth bluez bluez-tools -y
```

Install `rfkill` as well:

```bash
sudo apt install rfkill -y
```

## Check the Bluetooth installation

Check the BlueZ version:

```bash
bluetoothctl --version
```

Check whether a Bluetooth adapter is present:

```bash
lsusb | grep -i bluetooth
lspci | grep -i bluetooth
```

Check whether the kernel recognizes the adapter:

```bash
hciconfig -a
```

Or use the newer BlueZ interface:

```bash
bluetoothctl show
```

Check loaded Bluetooth kernel modules:

```bash
lsmod | grep bluetooth
```

View the adapter version:

```bash
hciconfig hci0 version
```

---

# Headless Ubuntu fixes

Because the Raspberry Pi is running Ubuntu Server without a desktop environment, some Bluetooth/PipeWire components may need additional configuration.

Make sure the Bluetooth PipeWire integration package is installed:

```bash
sudo apt install libspa-0.2-bluetooth
```

You can confirm that it is actually installed rather than only having the PipeWire/WirePlumber core packages.

## Enable user lingering

This allows the user's systemd user services to continue running without an active graphical login session.

Replace `<your-user>` with the Linux user that will run the audio system:

```bash
sudo loginctl enable-linger <your-user>
```

Then, while logged in as that user:

```bash
systemctl --user enable --now pipewire pipewire-pulse wireplumber
```

## WirePlumber configuration

Edit:

```text
/usr/share/wireplumber/wireplumber.conf
```

Add/modify:

```text
monitor.bluez.seat-monitoring = disabled

wireplumber.profiles = {
  main = {
    monitor.bluez.seat-monitoring = disabled
  }
}
```

This is intended to prevent WirePlumber from depending on desktop-session seat monitoring, which can otherwise interfere with Bluetooth audio on a headless server.

---

# Pairing the speakers

`bluetoothctl` is the main command-line tool used to manage Bluetooth devices.

Start it with:

```bash
bluetoothctl
```

Inside the interactive shell:

```text
[bluetooth]# show
[bluetooth]# power on
[bluetooth]# agent on
[bluetooth]# default-agent
[bluetooth]# discoverable on
[bluetooth]# discoverable-timeout 120
[bluetooth]# pairable on
```

Exit with:

```text
[bluetooth]# quit
```

To connect a speaker:

```bash
bluetoothctl
```

Then:

```text
[bluetooth]# power on
[bluetooth]# scan on
[bluetooth]# pair XX:XX:XX:XX:XX:XX
[bluetooth]# trust XX:XX:XX:XX:XX:XX
[bluetooth]# connect XX:XX:XX:XX:XX:XX
```

Repeat the pairing and connection process for the second speaker.

---

# Combining the two Bluetooth speakers

Once both speakers are connected, check the available sinks:

```bash
pactl list short sinks
```

You should see Bluetooth sinks similar to:

```text
bluez_output.52_58_0D_19_0A_4B.1
bluez_output.63_5E_53_8E_2B_06.1
```

Create a combined sink:

```bash
pactl load-module module-combine-sink     sink_name=both_speakers     slaves=bluez_output.52_58_0D_19_0A_4B.1,bluez_output.63_5E_53_8E_2B_06.1
```

Set it as the default output:

```bash
pactl set-default-sink both_speakers
```

At this point, normal PulseAudio-compatible applications can send audio to `both_speakers`, causing the same audio stream to be sent to both Bluetooth speakers.

---

# ALSA / PipeWire configuration

If an application tries to use ALSA directly, it can be useful to route the ALSA default device through PipeWire.

Create or edit:

```text
~/.asoundrc
```

Or configure it system-wide in:

```text
/etc/asound.conf
```

Use:

```text
pcm.!default {
    type pipewire
}

ctl.!default {
    type pipewire
}
```

This allows ALSA applications to use PipeWire as their audio backend.

---

## Test audio playback

Using the PulseAudio-compatible interface:

```bash
paplay /usr/share/sounds/alsa/Front_Center.wav
```

Using PipeWire directly:

```bash
pw-play /usr/share/sounds/alsa/Front_Center.wav
```

The second command is useful when testing PipeWire itself because it talks directly to the PipeWire audio system.

More information:

https://oneuptime.com/blog/post/2026-03-02-how-to-set-up-pipewire-as-audio-server-on-ubuntu/view

---

# Project architecture

The basic architecture is:

```text
                 ┌──────────────────────┐
                 │    Raspberry Pi 4B   │
                 │                      │
                 │  Ubuntu Server       │
                 │  BlueZ               │
                 │  PipeWire            │
                 │  WirePlumber         │
                 └──────────┬───────────┘
                            │
                 Bluetooth │ Bluetooth
                    ┌───────┴───────┐
                    │               │
                    ▼               ▼
             ┌─────────────┐ ┌─────────────┐
             │  Speaker 1  │ │  Speaker 2  │
             │             │ │             │
             │ BT receiver │ │ BT receiver │
             │     ↓       │ │     ↓       │
             │ TPA3116D2   │ │ TPA3116D2   │
             │     ↓       │ │     ↓       │
             │  Speaker    │ │  Speaker    │
             └─────────────┘ └─────────────┘
                    │               │
                 Battery         Battery
```

The Raspberry Pi is therefore the central controller. The speaker units do not need to run Linux or perform any audio processing themselves; they only need to receive Bluetooth audio and amplify it.

---

# Notes and next steps

This README currently documents the working hardware setup and the basic Ubuntu/PipeWire/Bluetooth configuration.

You can now go ahead and use this in combination with uxplay (linux airplay server) or cliamp (terminal winamp style media player) to enjoy some music with your wireless bluetooth system!

The next logical steps for the project are:

- automate Bluetooth speaker connection
- automatically create the combined audio sink
- investigate synchronization between the Bluetooth speakers
- measure the latency of each speaker
- add per-speaker delay compensation
- automate calibration from the Raspberry Pi
- eventually turn the setup into a proper multi-speaker wireless surround system
