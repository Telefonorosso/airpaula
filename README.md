# Emu68 Paula Bluetooth

Stream real Amiga Paula audio over Bluetooth Classic A2DP (SBC) using a PiStorm and the Raspberry Pi's onboard Bluetooth controller. The system runs in bare-metal Emu68 without Linux, USB audio hardware, or an AmigaOS Bluetooth service. PaulaMixer provides native AmigaOS mixer controls.

**Hardware-tested configuration:** Amiga 600, PiStorm Classic, Raspberry Pi 3A+, AmigaOS 3.1, and MiamiDX/WiFiPi.device. Other combinations have not been validated.

## Requirements

- A **matching prebuilt** POC11-B FIX1 Emu68 kernel and `BTDiag` (diagnostic ABI 13).
- The working PaulaMixer Commodity executable and its `.info` icon, if mixer controls are desired.
- Bluetooth Classic A2DP headphones or speaker. MiamiDX and WiFiPi.device are optional for Wi-Fi networking.

**Note:** The source milestone ZIP does **not** contain the tested prebuilt kernel or AmigaOS executables. Obtain the matching binaries separately; this document does not cover compilation.

## Installation and first pairing

1. Back up the current Emu68 boot image and the Raspberry Pi boot partition's `config.txt`. Install the POC11-B FIX1 Emu68 boot image in place of the existing one. Copy `BTDiag` to `C:` (or another directory in your AmigaDOS command path).
2. For initial discovery, use these lines in the boot partition's `config.txt` (retaining any other required settings):

   ```ini
   dtoverlay=emu68
   dtparam=args=paula_bt
   ```

3. Perform a **cold boot**. Put the audio device in discoverable/pairing mode, then run:

   ```text
   BTDiag --scan
   ```

4. Copy the device's Bluetooth Classic address from the scan and update `config.txt`:

   ```ini
   dtoverlay=emu68
   dtparam=args=paula_bt bt_peer=88:E9:FE:8E:BC:7F
   ```

   The address above is an example; use your own device's address. Perform another cold boot with the device in pairing mode. Wait for A2DP audio to start.
5. Once pairing succeeds, run:

   ```text
   BTDiag --key
   ```

   Copy the **complete `dtparam=args=...` line** printed by BTDiag into `config.txt`, replacing the previous `dtparam=args` line. It includes `bt_bond=TYPE:LINK_KEY` and preserves the pairing across reboots. The kernel obtains its own Bluetooth address from the Raspberry Pi Device Tree automatically; `bt_bdaddr=` is normally unnecessary.

   **Keep the Link Key private**: it is a Bluetooth bonding credential. Do not publish it in logs, screenshots, or example configurations.

## Everyday use

- Power on the paired speaker/headphones and boot the Amiga. Bluetooth reconnects automatically, including after a chord reset or speaker power cycle.
- Run `BTDiag` to inspect Bluetooth state. For working playback, look for `Reconnect: streaming`, `stream_open=1` and `started=1`.
- `BTDiag --scan` discovers devices; `BTDiag --key` displays the saved bonding configuration; `BTDiag --recover` performs a **Bluetooth-only rebootstrap** after the WiFiPi initialization race. Do not invoke `--recover` repeatedly while Bluetooth is healthy.

## Wi-Fi with MiamiDX

On the tested Raspberry Pi 3A+, bringing WiFiPi online **after** Bluetooth can reset the shared wireless chip and interrupt audio. The hardware-validated workaround is to run `BTDiag --recover` **once, after WiFiPi has initialized**. This restores A2DP without resetting the Raspberry Pi, disconnecting Wi-Fi, or modifying WiFiPi.device.

1. Create `S:PaulaBT-Recover` containing:

   ```amigados
   C:BTDiag --recover >NIL:
   ```

2. In **MiamiDX → Events**, configure the appropriate **Start/online** event to execute `S:PaulaBT-Recover` once after the Wi-Fi interface has come online. Use the exact event timing that works in your MiamiDX setup.
3. For initial testing, remove `>NIL:` to see the recovery result. `HCI WORKING` means rebootstrap succeeded; A2DP reconnection follows asynchronously. Confirm with normal `BTDiag` and audible playback.

As an alternative **Wi-Fi-first** workaround, the standalone token `pause` in `cmdline.txt` delays Bluetooth startup by about 60 seconds, allowing WiFiPi to initialize first. Normally **do not use `pause` together with the MiamiDX recovery event**.

## PaulaMixer Commodity

Copy the matching `PaulaMixer` executable and `PaulaMixer.info` to the same directory. For automatic startup, place PaulaMixer in `SYS:WBStartup` with its `.info` icon and configure ToolTypes in Workbench **Information**:

```ini
DONOTWAIT
CX_POPUP=NO
CX_PRIORITY=0
CX_POPKEY=rawkey control alt m
MASTER=80
STEREO=100
INTERPOLATION=LINEAR
MUTE0=NO
MUTE1=NO
MUTE2=NO
MUTE3=NO
```

`CX_POPUP=NO` starts PaulaMixer silently as a Commodity; show its GUI through Commodities Exchange or the configured hotkey. `CX_POPUP=YES` opens the window at startup. `MASTER` and `STEREO` accept values from 0 to 100. `INTERPOLATION` accepts `LINEAR` or `NEAREST`. Closing the GUI leaves the Commodity running.

To apply settings once from Shell without keeping a resident Commodity:

```text
PaulaMixer ONESHOT=YES MASTER=80
```

## Boot configuration notes

- **PAL is the default.** Do not append `bt_paula_clock=pal` to the already long `dtparam=args` line.
- For NTSC, add `bt_paula_clock=ntsc` to the existing single line in `cmdline.txt`.
- Raspberry Pi `config.txt` lines are limited to **98 characters** in this setup. Keep `dtparam=args=...` on one line; do **not** add a second `dtparam=args` entry as a workaround.
- After changing boot files, `config.txt`, or `cmdline.txt`, use a **cold boot** so the Raspberry Pi firmware reads the new configuration.

