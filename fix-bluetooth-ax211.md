# Fix: Intel AX211 Bluetooth — Hardware error 0x0c crashes

Affects: Bluetooth peripherals (especially the PUSAT B.PRO MINI keyboard) disconnecting
repeatedly because the AX211 BT firmware crashes with `Hardware error 0x0c` followed by
`Retrieving Intel exception info failed (-16)`.

**Status (2026-05-15): FIXED for chip crashes by downgrading firmware to build 3243.**
Older firmware build **3243 / timestamp 2024.18** (the one that originally shipped with
Ubuntu 24.04 + Tuxedo PPA) runs **stable on this chip — no Hardware error 0x0c**.
Newer upstream builds **3831 (2025.13)** and **3882 (2025.20)** both crash this specific
silicon. The 2026-05-13 conclusion that "firmware is ruled out" was wrong; it only tested
the two newer builds against each other. Build 3243 was never tested until 2026-05-15.

**Separately, the audio glitches** (the symptom that drove the 2026-05-15 investigation)
turned out to be **Liberty 4 Pro multipoint mode** — the headset juggling laptop+phone,
not Linux. Fix is in the Soundcore app, not Linux. See "Audio glitches" section below.

> **Current status (2026-07-12):** live firmware is **build 3243** (verified:
> `Firmware timestamp 2024.18 buildtype 1 build 3243`, sha256 `fceb7e375d020357…`),
> `linux-firmware` is on `apt-mark hold`, and the system has since moved to
> Ubuntu 26.04 / kernel 7.0 — 3243 is still the only stable build; upstream HEAD
> is still 3882 with no fix in sight. ⚠️ The `~/upgrade-recovery/` backup copies
> mentioned below were **deleted 2026-05-22** — the live
> `/lib/firmware/intel/ibt-0180-0041.sfi` is now the **only** copy of 3243 on disk.
> **Back it up before touching the hold or the firmware directory.** The
> experiment of running build 3604 (2026-05-17, mentioned below) is over; it
> crashed within ~4 minutes idle and the system went back to 3243.

---

## Arch Linux port (2026-08-13)

System moved from Ubuntu 24.04/26.04 to **Arch Linux** (kernel `7.1.8-arch1-3`),
desktop is now Hyprland (Caelestia shell) instead of GNOME. Confirmed **same
physical hardware**: `iwlwifi` identifies it as `Intel(R) Wi-Fi 6E AX211`, and
the Bluetooth half enumerates at `usb 3-10` on `pci0000:00/0000:00:14.0/usb3`
— matching this doc's topology notes exactly.

**Fresh install shipped the known-bad build.** `pacman`'s `linux-firmware-intel`
(`20260622-1`) puts down `Firmware timestamp 2025.20 buildtype 1 build 3882`,
SHA1 `0x937bca4a` — the same crashy build documented above, confirmed via
`journalctl -k -b`. No `Hardware error 0x0c` yet in the first boot (bluetooth
was only just enabled), but expected given the documented pattern — and this
kernel (7.1) is in the "worse" bracket the original notes warned about
(6.17/7.0 crash every 1–3 min vs. 6.8's ~13 min).

**No apt-mark-hold equivalent needed.** Arch ships the firmware *compressed*
(`/usr/lib/firmware/intel/ibt-0180-0041.sfi.zst`, owned by `linux-firmware-intel`),
and the kernel firmware loader prefers an uncompressed `.sfi` with the same
base name if one exists alongside it. So the fix is just to drop the known-good
uncompressed blob in next to it — pacman doesn't own that filename, so
`pacman -Syu` can never overwrite it. No hold, no pacman hook needed.

**Recovered build 3243** from the Arch Linux Archive (checked package-by-package
against the sha256 table above): it first appears in
`linux-firmware-20240703.e94a2a3b-1` (verified sha256 `fceb7e375d0203...`,
exact match). Cached locally at `firmware-cache/ibt-0180-0041.sfi.stable-3243`
(gitignored — binary blob, not published to the repo). If that cache is ever
lost, re-derive it:
```bash
curl -O https://archive.archlinux.org/packages/l/linux-firmware/linux-firmware-20240703.e94a2a3b-1-any.pkg.tar.zst
tar --zstd -xf linux-firmware-20240703.e94a2a3b-1-any.pkg.tar.zst \
    usr/lib/firmware/intel/ibt-0180-0041.sfi.zst -O | zstd -d -o ibt-0180-0041.sfi.stable-3243
sha256sum ibt-0180-0041.sfi.stable-3243   # expect fceb7e375d020357b3ca10dbad2fb5be6ec8a4d6bbb07feb35dc3869dfa440e9
```

**Apply on this machine:**
```bash
sudo cp /home/osm/Documents/GitHub/monster-notebook-linux/firmware-cache/ibt-0180-0041.sfi.stable-3243 \
        /usr/lib/firmware/intel/ibt-0180-0041.sfi
sudo systemctl stop bluetooth
sudo modprobe -r btusb btintel
sudo modprobe btusb
sudo systemctl start bluetooth
journalctl -k -b | grep -E "Firmware (timestamp|SHA1)" | tail -2
# expect: Firmware timestamp 2024.18 buildtype 1 build 3243 / SHA1: 0xa8bb3f39
```

**bluez tuning not yet ported.** `/etc/bluetooth/main.conf` and `input.conf`
are stock Arch defaults (everything commented out) — the Ubuntu-era
`ReconnectUUIDs` (HID profile), `ReconnectAttempts`/`Intervals`, `IdleTimeout`,
and SAP-plugin-disable drop-in from the "Configuration Currently Applied"
section below have **not** been re-applied here yet. Worth porting if a BT
keyboard/mouse shows the same reconnect flakiness; the USB-autosuspend rule
too, once the affected USB bus ID is confirmed on this box (still `3-10` per
above, so likely reusable as-is). The `logind` suspend-storm drop-in is
**not** ported — that was a fix for a different symptom (`tracker-miner-fs-3`
D-state hangs blocking suspend) that hasn't been observed on this install;
apply only if the same crash-storm-on-suspend pattern actually shows up here.

## Symptom

```
Bluetooth: hciN: Hardware error 0x0c
Bluetooth: hciN: Retrieving Intel exception info failed (-16)
```

Cadence varies. On kernel 6.17 HWE: every 1–3 minutes. On kernel 6.8.0-111 (current):
usually ~13 minutes between crashes, but storms of one crash every ~30 seconds happen
when the chip enters a degraded state. After a crash, BT peripherals disconnect; bluetoothd
recovers the link within a few seconds, but the keyboard pairing is interrupted enough to
lose keystrokes / require physical wake.

## Hardware Context

- **Chip:** Intel AX211 combo (WiFi + Bluetooth), USB vendor:product `8087:0033`
- **USB topology:** AX211 BT half is `usb 3-10` on the internal USB 2.0 hub
- **Driver stack:** `btusb` (USB transport) + `btintel` (Intel-specific quirks)
- **Firmware file:** `/lib/firmware/intel/ibt-0180-0041.sfi`
- **DDC file:** `/lib/firmware/intel/ibt-0180-0041.ddc` — symlink to `ibt-0040-0041.ddc`
- **Affected peripheral:** PUSAT B.PRO MINI Bluetooth keyboard (BT 5.1, no USB/dongle fallback)

## Root cause: firmware build choice (CORRECTED 2026-05-15)

Earlier conclusion ("firmware ruled out") was wrong — it only tested two newer
revisions against each other. Build **3243** was the missing test, and it runs
clean on this hardware.

| Build | Timestamp | SHA1 | sha256 (first 16) | Status on this chip |
|---|---|---|---|---|
| **3243** | **2024.18** | **0xa8bb3f39** | **`fceb7e375d020357`** | ✅ **Stable — no Hardware error 0x0c** |
| 3604 | 2024.48 | 0xc115e35a | `acc84f47e763c740` | ❌ Hardware error 0x0c within ~4 min idle (tested 2026-05-17) |
| 3831 | 2025.13 | 0x47cf9d0e | `e3c16c3c4024a84e` | ❌ Crashes within 15 s of load |
| 3882 | 2025.20 | 0x937bca4a | `205f22d76408283b` | ❌ Crashes every ~13 min on 6.8, every 1–3 min on 6.17 / 7.0; **upstream HEAD as of 2025-10-10** |

**Currently active:** build **3243** (the 3604 experiment of 2026-05-17 lasted about
four minutes before the first `Hardware error 0x0c`; reverted same day).

Firmware inventory on disk (as of 2026-05-17 — **the `~/upgrade-recovery/` entries
below no longer exist**, deleted 2026-05-22; kept here so the sha256 table above
stays meaningful):

- `/lib/firmware/intel/ibt-0180-0041.sfi` — **live (build 3243, stable, uncompressed; kernel firmware loader prefers uncompressed over `.sfi.zst`)**
- `/lib/firmware/intel/ibt-0180-0041.sfi.zst` — what `linux-firmware` 26.04 ships (build 3882, *not* in use because uncompressed exists)
- `~/upgrade-recovery/ibt-0180-0041.sfi.stable-3243` — explicit named copy of 3243 (sha `fceb7e3…`)
- `~/upgrade-recovery/ibt-0180-0041.sfi.bak` — same as 3243
- `~/upgrade-recovery/ibt-0180-0041.sfi.build-3604` — Jan 2025 upstream commit `7ccc69cfa4`, downloaded 2026-05-17 from `endlessm/linux-firmware` (cross-verified against `pop-os/linux-firmware`), also crashes
- `~/upgrade-recovery/ibt-0180-0041.sfi.upstream-3882` — saved upstream HEAD (crashy)
- `~/upgrade-recovery/ibt-0180-0041.sfi.bak2` — same as 3882

**`linux-firmware` package on apt-mark hold** so apt-upgrades don't reinstall the package
and overwrite the live `.sfi`. Since the live file is now the only copy of 3243, the
safe procedure if you ever need to unhold is:
```bash
# BEFORE anything else — save the stable blob somewhere an upgrade can't touch
sudo cp /lib/firmware/intel/ibt-0180-0041.sfi /root/ibt-0180-0041.sfi.stable-3243
sudo apt-mark unhold linux-firmware
# ... after any linux-firmware upgrade, put 3243 back:
sudo cp /root/ibt-0180-0041.sfi.stable-3243 /lib/firmware/intel/ibt-0180-0041.sfi
sudo modprobe -r btusb btintel && sudo modprobe btusb
sudo apt-mark hold linux-firmware
```

To switch BACK to upstream 3882 (e.g. to test if a future firmware version is finally
stable, or for a different chip):
```bash
sudo cp /lib/firmware/intel/ibt-0180-0041.sfi.upstream-3882 /lib/firmware/intel/ibt-0180-0041.sfi
sudo systemctl stop bluetooth && sudo modprobe -r btusb btintel && sudo modprobe btusb && sudo systemctl start bluetooth
```

### Confirming the live firmware build
```bash
sudo dmesg | grep -E "Firmware (timestamp|Version)" | tail -3
# Should show:  Firmware timestamp 2024.18 buildtype 1 build 3243
```

## Audio glitches — UNRESOLVED (as of 2026-05-15)

Symptom: brief audio drops / drips / quality loss on the Liberty 4 Pro headset, even
after the chip stopped crashing on firmware 3243.

What's been ruled out so far:
- **Chip crashes**: firmware 3243 produces zero `Hardware error 0x0c` for the duration
  of audio testing.
- **Range / signal strength**: bluetoothctl reports RSSI **-7 dBm** (extremely strong,
  near-touching) and link quality 103. Not an RF distance problem.
- **Multipoint mode on the headset**: user confirmed Liberty 4 Pro is paired only to
  the PC, no phone or second host.
- **Two-device BR/EDR concurrency**: glitches persisted with the keyboard disconnected
  (only the headset on the chip).
- **System-wide audio path**: internal speakers played clean — BT path is specifically
  the problem.
- **Pipewire underrun headroom**: quantum is 4096 (~85 ms) and didn't fix it.

What's puzzling:
- Liberty 4 Pro is LDAC-capable but only advertises **SBC / SBC-XQ** in A2DP
  negotiation. wireplumber offered LDAC/aptX_HD/aptX/AAC during testing — bluez
  registered the endpoints, headset refused them. No multipoint, no obvious mode lock;
  unclear why LDAC isn't negotiated.

Plausible remaining causes (not investigated this session):
- Anker firmware limiting codec advertisement based on some other state (e.g. Game Mode,
  ANC setting, etc.)
- A2DP packet loss at RF level despite the -7 dBm signal (interference inside the
  chip's link manager that doesn't surface as Hardware error 0x0c)
- A wireplumber / pipewire bug in 1.0.5 / 0.4.17 on Ubuntu 24.04 specifically with this
  combo
- The headset itself — try pairing to a different Linux machine to isolate.

To continue when picking this back up: `btmon` trace during playback (`sudo btmon -t -w
trace.btsnoop &` then play music for a minute; analyze in Wireshark with the bluetooth
dissector for retransmits and SBC frame drops).

### 2. WiFi/BT 2.4 GHz coexistence (effectively ruled out, 2026-05-13)

The classic AX211 failure mode is BT + 2.4 GHz WiFi fighting for the chip's shared RF
frontend. **This system's WiFi is on 5 GHz** (channel 52, 5260 MHz), so the in-chip
coexistence path is not active. Verified with `iw dev`.

### 3. USB autosuspend (already mitigated, 2026-05-12)

- udev rule `/etc/udev/rules.d/99-bluetooth-autosuspend.rules` forces `power/control=on`
  for AX211.
- Kernel cmdline includes `btusb.enable_autosuspend=0`.
- Confirmed: `cat /sys/bus/usb/devices/3-10/power/control` returns `on`.

### 4. Suspend-related crash storms (mitigated 2026-05-13)

A separate but cascading issue — lid-close suspend failed because `tracker-miner-fs-3` got
stuck in D-state on fuse I/O, and logind retried suspend 6× in 2 min. Each failed
suspend brutalised AX211 firmware. Fixed with a logind drop-in
(`/etc/systemd/logind.conf.d/01-no-auto-suspend.conf`) that ignores lid/idle/suspend-key
events, so only user-initiated suspend can happen.

## Configuration Currently Applied

### `/etc/bluetooth/main.conf` (bluetoothd)

Verbatim of relevant settings:

```ini
[General]
FastConnectable = true
AutoEnable = true
ReconnectUUIDs = 00001124-0000-1000-8000-00805f9b34fb  # HID profile (was missing — fixed 2026-05-12)
ReconnectAttempts = 7
ReconnectIntervals = 1, 2, 4, 8, 16, 32, 64
```

### `/etc/bluetooth/input.conf` (HID profile)

```ini
[General]
IdleTimeout = 15        # was 0 (never disconnect on idle), bumped to 15 min 2026-05-13
UserspaceHID = true
ClassicBondedOnly = false
```

Note: `IdleTimeout=0` actually means *never* disconnect on idle. Bumping it to 15 is
technically a regression — but the user's actual disconnects are firmware crashes, not
idle-driven, so the practical difference is zero.

### `/etc/systemd/system/bluetooth.service.d/disable-sap.conf`

`bluetoothd` runs with `--noplugin=sap` (the SAP plugin was crashing on startup).

### `~/.config/wireplumber/bluetooth.lua.d/51-bluez-override.lua` (A2DP-only + codec offer list, 2026-05-15)

Strips HSP/HFP entirely so wireplumber never tries to claim the
"Hands-Free Voice gateway" profile when a headset connects. Without
this, every headset connect produced this cascade in the journal:

```
bluetoothd: Unable to get io data for Hands-Free Voice gateway: getpeername: Transport endpoint is not connected (107)
dbus-daemon: Rejected send message ... bluetoothd ... wireplumber
bluetoothd: a2dp_select_capabilities() Unable to select SEP
bluetoothd: a2dp-sink profile connect failed: Device or resource busy
gnome-shell: Failed to connect device "soundcore Liberty 4 Pro": br-connection-page-timeout
```

The cause is a wireplumber/bluez race when negotiating two profiles
simultaneously — bluez exposes both A2DP and HFP, wireplumber claims
both, then bluetoothd refuses the HFP claim, and the A2DP transport
ends up "busy" because the link is half-torn-down. A2DP-only avoids
the race entirely.

The current file ALSO sets a codec preference list so wireplumber
offers LDAC/aptX_HD/aptX/AAC/SBC-XQ/SBC during AVDTP negotiation —
the device picks whichever it actually supports. Liberty 4 Pro in
multipoint mode only accepts SBC; switching it to single-device mode
in the Soundcore app should expose LDAC. The `bluez_monitor.rules`
intentionally does **not** force `device.profile`, so `pactl
set-card-profile … a2dp-sink-sbc_xq` can manually flip codec if
desired (note: some headsets disconnect during such a switch
mid-stream — pause audio first).

```lua
bluez_monitor.properties["bluez5.roles"]              = "[ a2dp_sink a2dp_source ]"
bluez_monitor.properties["bluez5.autoswitch-profile"] = false
bluez_monitor.properties["bluez5.hfphsp-backend"]     = "none"
bluez_monitor.properties["bluez5.codecs"]             = "[ ldac aptx_hd aptx aac sbc_xq sbc ]"
bluez_monitor.properties["bluez5.a2dp.ldac.quality"]  = "auto"

bluez_monitor.rules = {
  { matches = { { { "device.name", "matches", "bluez_card.*" } } },
    apply_properties = {
      ["bluez5.auto-connect"] = "[ a2dp_sink ]",
    } },
}
```

**Trade-off:** headset microphone is unusable (HSP/HFP is the only BT
mic protocol). Pure listening only. Backup of pre-fix file at
`51-bluez-override.lua.bak.<timestamp>`.

After editing: `systemctl --user restart wireplumber` then disconnect
+ reconnect the headset.

This does **NOT** fix the chip-level `Hardware error 0x0c` crashes —
those are still the AX211 hardware issue and require an AX210 swap or
USB BT dongle. But it eliminates the userspace error storm that was
amplifying every disconnect into a full audio collapse.

### `~/.config/pipewire/pipewire.conf.d/99-bt-audio-stability.conf` (audio glitch mitigation, 2026-05-15)

Raises the PipeWire clock quantum from the default 1024 frames
(~21 ms @ 48 kHz) to **2048 frames (~43 ms)**, with a 1024–8192 range.
This is the most impactful fix for AX211 BT audio glitches **without
swapping hardware**: a short Hardware-error-0x0c blip blanks audio in
under one default-quantum, but two quanta gives the recovery path
time to refill before the underrun is audible. Tested 2026-05-15 —
SBC stream on Liberty 4 Pro stays clean through brief chip hiccups.

```ini
context.properties = {
    default.clock.quantum       = 2048
    default.clock.min-quantum   = 1024
    default.clock.max-quantum   = 8192
}
```

**Trade-off:** ~80 ms added output latency on the BT path
(imperceptible for music/video, not suitable for live monitoring or
twitch gaming).

Reverse: `rm ~/.config/pipewire/pipewire.conf.d/99-bt-audio-stability.conf && systemctl --user restart pipewire`.

### LTS-only kernel constraint (2026-05-15)

User explicitly requested staying on **upstream LTS kernels only**.
Currently on Ubuntu 24.04's `linux-image-generic` (6.8.x series),
the GA kernel for Noble; supported through 2029. **HWE kernels (6.11
/ 6.14 / 6.17) are off the table** even if they ship newer btintel /
btusb patches. Practical consequence: the chip-level Hardware-error
0x0c crashes cannot be mitigated by kernel switch; userspace
buffer/codec tuning is the maximum achievable on this system. The
external USB BT dongle remains the only definitive fix.

### Firmware: definitively at upstream-latest (2026-05-15)

Local `/lib/firmware/intel/ibt-0180-0041.sfi` byte-matches upstream
linux-firmware (`sha256: 205f22d76408283b…`, build 3882, timestamp
2025.20). Nothing newer exists upstream as of session date. There
is no firmware update to chase.

## Diagnostic Commands

```bash
# Firmware crashes in current boot
journalctl -k -b | grep "Hardware error\|hci"

# Live monitor
journalctl -k -f | grep -E "Hardware error|hci|Bluetooth: "

# What firmware is loaded right now (build + sha1 + timestamp)
sudo dmesg | grep -E "hci[0-9]+: Firmware (Version|timestamp|SHA1)" | tail -10

# AX211 USB power state (should be "on")
cat /sys/bus/usb/devices/3-10/power/control

# Reload btusb (re-loads firmware without reboot)
sudo systemctl stop bluetooth && sudo modprobe -r btusb && sudo modprobe btusb && sudo systemctl start bluetooth

# Hot-recover BT after a crash storm
sudo rfkill block bluetooth && sudo rfkill unblock bluetooth

# Confirm WiFi band (5 GHz = good for AX211 coexistence)
iw dev | grep -E "channel|freq"
```

## What's Left to Try

In rough order of effort:

### (A) Disable BT MSFT extension / specific features

`dmesg` shows `Failed to read MSFT supported features (-19)` and `Bad flag given (0x1) vs
supported (0x0)` on every reload. These are non-fatal but suggest the host driver is
negotiating features the firmware rejects. Possible mitigations:

- Investigate `btusb` module parameters: `modinfo btusb`
- Some kernels expose `enable_msft` or similar; if so, force it off.

### (B) Newer LTS kernel between 6.8 and 6.17

The user switched off 6.17 HWE specifically because AX211 BT was *worse* there. But there
may be a kernel in between (e.g. 6.12 LTS) with btusb/btintel fixes that don't carry the
6.17 regression. Lower priority — the user is sceptical and the win is uncertain.

### (C) External 2.4 GHz interference audit

5 GHz WiFi rules out in-chip coexistence. But ambient 2.4 GHz noise can still corrupt BT:

- Any 2.4 GHz wireless mouse/keyboard receiver dongles plugged in nearby — none currently
  (the SHARKOON mouse on bus 003 is wired/charging; its "2.4GHz Wireless" name is just the
  product string in firmware).
- USB 3.0 device next to the laptop (well-documented Intel paper: USB 3.0 emits noise
  around 2.4–2.5 GHz). Not currently active — no USB 3.0+ device enumerated.
- Microwave running during keyboard sessions.

### (D) USB Bluetooth adapter — **recommended fix** (2026-05-13)

The decision after A–C: external USB BT adapter (~$5–15). Sidesteps the AX211 BT entirely
while keeping AX211 WiFi (which is fine on 5 GHz). Reversible — unplug and the AX211 BT is
back. Solves all BT devices at once, not just the keyboard. The keyboard itself is fine —
no Linux-specific issues for the PUSAT B.PRO MINI.

#### Recommended adapter chipsets

- **Realtek RTL8761B / RTL8761BU** (BT 5.x, broad Linux support since kernel 5.8+, used by
  most $5–10 nano dongles on Amazon / AliExpress)
- **CSR8510** (cheap, BT 4.0 only — fine for keyboards/mice, no BLE Audio)
- Avoid: anything sold as "Broadcom BCM20702" — driver bitrot, requires hex firmware
  blobs scraped from Windows installers.

#### Pre-staged disable rule (already on disk)

`/etc/udev/rules.d/99-disable-ax211-bt.rules.disabled`

When inactive (current state, `.disabled` extension), udev ignores it. When activated, it
sets `ATTR{authorized}=0` on the AX211 BT USB device (8087:0033) at every enumeration,
making it invisible to btusb/bluez. The AX211 WiFi half is on PCIe (not USB) and is
unaffected.

#### Activation procedure (when the adapter arrives)

1. Plug in the new USB BT adapter. Confirm it enumerates:
   ```bash
   lsusb | grep -iE "bluetooth|realtek|csr|cambridge"
   dmesg | tail -30   # look for "Bluetooth: hciN: ..." with the new product/firmware line
   hciconfig          # should now show 2 controllers: hci0 (AX211) and hci1 (new)
   ```
2. Activate the AX211 disable rule:
   ```bash
   sudo mv /etc/udev/rules.d/99-disable-ax211-bt.rules.disabled \
           /etc/udev/rules.d/99-disable-ax211-bt.rules
   sudo udevadm control --reload
   sudo udevadm trigger --action=add --attr-match=idVendor=8087
   ```
   The AX211 BT controller (hci0) should disappear immediately. Verify:
   ```bash
   hciconfig         # only the new adapter should show
   cat /sys/bus/usb/devices/3-10/authorized   # should be 0
   ```
3. Re-pair the keyboard to the new adapter:
   ```bash
   bluetoothctl
   > power on
   > scan on
   # wait for the keyboard MAC to appear, then:
   > pair <MAC>
   > trust <MAC>
   > connect <MAC>
   ```
4. Confirm no `Hardware error 0x0c` crashes for ~30 minutes:
   ```bash
   journalctl -k -f | grep -E "Hardware error|hci"
   ```

#### Rollback (re-enable AX211 BT)

```bash
sudo rm /etc/udev/rules.d/99-disable-ax211-bt.rules
sudo udevadm control --reload
sudo udevadm trigger --action=add --attr-match=idVendor=8087
# Or just reboot — udev will not deauthorize the device without the rule.
```

#### Alternative routes (worse value, documented for completeness)

- **Replace the M.2 WiFi/BT card** with a different model (e.g. AX210 — older revision,
  fewer reported coex issues; or a different vendor). Requires opening the laptop.
- **Replace the keyboard with one that has a 2.4 GHz USB receiver** — uses the receiver
  instead of BT, bypasses AX211 BT but ties you to one specific keyboard.

Neither is recommended unless the USB BT adapter route somehow doesn't work.

## Rollback / Recovery

If anything in this directory's instructions makes things worse:

1. **Firmware:** see the firmware section above for the restore procedure (back up the
   live 3243 blob first — it's the only copy).
2. **bluez config:** `/etc/bluetooth/main.conf` and `input.conf` are conffiles —
   `sudo apt install --reinstall bluez` restores stock defaults.
3. **Suspend policy:** remove `/etc/systemd/logind.conf.d/01-no-auto-suspend.conf` and
   restart `systemd-logind` to restore default lid/idle suspend behavior.

## References

- Linux kernel `btusb` / `btintel` source: `drivers/bluetooth/btusb.c`, `btintel.c`
- linux-firmware tree: https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/
- Keyboard product page: https://www.monsternotebook.com.tr/klavye/pusat-business-pro-mini-bluetooth-kablosuz-klavye-siyah/
  (Bluetooth 5.1, 10 m range, 3-device pairing, no USB receiver option)
