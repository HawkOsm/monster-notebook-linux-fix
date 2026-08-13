# Fix: Keyboard Backlight & Touchpad Locked

**Symptoms:** Keyboard backlight is dead or stuck off. Touchpad is disabled and cannot be toggled with `Fn`+`F1`.

**Tested on:** Monster TULPAR T6 V2.1 · Ubuntu 24.04 · Kernel 6.17+

---

## Why This Happens

### Keyboard Backlight

The `tuxedo-drivers` package controls the Clevo keyboard hardware. It contains a compatibility gate that only allows itself to load on systems where the DMI vendor string is exactly `"TUXEDO"`. Monster notebooks report `"MONSTER"` — so the driver refuses to initialize and exits silently with `No such device`.

On kernel 6.17+, an additional problem appears: the kernel ships a built-in `tuxedo_io` v0.3.9 while the DKMS package provides v0.3.6, causing a module version collision.

### Touchpad Locked

The Embedded Controller (EC) on Clevo boards has a hardware-level touchpad enable/disable flag. This flag survives reboots. When the Tuxedo driver fails to load, the EC register can get stuck at `0` (disabled) with no way to toggle it back — because the `/dev/tuxedo_io` device that controls the EC never loads.

---

## Arch Linux port (2026-08-13)

Same laptop (`TULPAR T6 V2.1` per `/sys/class/dmi/id/product_name`), moved to
**Arch Linux** (kernel `7.1.8-arch1-3`), desktop is Hyprland (Caelestia shell)
instead of GNOME. The DMI-gate bug and the fix are identical — Arch just uses
different packaging.

**No Tuxedo apt repo needed.** Skip Step 1 entirely; install from the AUR
instead (`yay` already on this system):

```bash
yay -S --needed tuxedo-drivers-dkms tuxedo-control-center-bin
```

At the time of writing this pulled `tuxedo-drivers-dkms 4.22.2-2` —
**the exact same version** (`4.22.2`) this doc's Step 3 patch was written
against, so the DMI patch (Python script or manual edit) applies verbatim,
same file path:

```
/usr/src/tuxedo-drivers-4.22.2/tuxedo_compatibility_check/tuxedo_compatibility_check.c
```

Confirmed on this install: `modprobe tuxedo_keyboard` failed with the exact
`No such device` from "Why This Happens" before patching, and loaded clean
after. `tuxedo-control-center-bin` (not the source-build `tuxedo-control-center`)
was used to skip compiling Electron — installs `tccd.service`
pre-enabled, no separate systemd setup needed.

Steps 4–7 (rebuild DKMS, `/etc/modules-load.d/tuxedo.conf`, load modules now,
set backlight brightness) are unchanged and apply as-is — just use the
corrected Step 6 loop below, not the original one-liner.

**Touchpad was never stuck here** — unlike the scenario this doc was
originally written for, `/dev/tuxedo_io`'s EC touchpad-enabled flag read `0`
(disabled) on first check, but the touchpad (`FTCS1000:01 2808:0222`, a
generic `i2c_hid` device, not gated through `tuxedo_io` on this board) worked
throughout regardless. So Steps 8–9 (EC unlock script, persistent boot
service) were **not applied** — nothing was stuck to unstick. If the
touchpad ever does go dead on this install, they're the fallback; see also
the live Fn-key toggle in "Hyprland: Wiring the Fn Hotkeys" below, which is a
different mechanism (toggles the same EC flag on demand from a keypress,
rather than force-enabling it once at boot).

Step 10 (initramfs + reboot) wasn't needed — `yay`'s pacman hook already
rebuilds the initcpio automatically on DKMS install, and all modules were
loaded live via `modprobe` in the same session.

## Step 1 — Add the Tuxedo Repository

Skip this step if you already have the Tuxedo repo configured.

```bash
curl -s https://deb.tuxedocomputers.com/ubuntu/dists/noble/Release.gpg \
  | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/tuxedo.gpg > /dev/null

echo "deb https://deb.tuxedocomputers.com/ubuntu noble main" \
  | sudo tee /etc/apt/sources.list.d/tuxedo.list

sudo apt-get update
```

## Step 2 — Install tuxedo-drivers-dkms

```bash
sudo apt-get install -y tuxedo-drivers-dkms
```

The source will be placed at:

```
/usr/src/tuxedo-drivers-4.22.2/tuxedo_compatibility_check/tuxedo_compatibility_check.c
```

## Step 3 — Patch the DMI Compatibility Check

The driver rejects non-TUXEDO vendors. Open the file above and find this block:

```c
	{
		.matches = {
			DMI_MATCH(DMI_CHASSIS_VENDOR, "TUXEDO"),
		},
	},
	{ }
};
```

Replace it with:

```c
	{
		.matches = {
			DMI_MATCH(DMI_CHASSIS_VENDOR, "TUXEDO"),
		},
	},
	{
		.matches = {
			DMI_MATCH(DMI_SYS_VENDOR, "MONSTER"),
		},
	},
	{ }
};
```

Or apply the patch with Python (no editor needed):

```bash
sudo python3 - /usr/src/tuxedo-drivers-4.22.2/tuxedo_compatibility_check/tuxedo_compatibility_check.c << 'EOF'
import sys
path = sys.argv[1]
with open(path) as f:
    content = f.read()

old = ('\t{\n\t\t.matches = {\n\t\t\tDMI_MATCH(DMI_CHASSIS_VENDOR, "TUXEDO"),\n'
       '\t\t},\n\t},\n\t{ }\n};')
new = ('\t{\n\t\t.matches = {\n\t\t\tDMI_MATCH(DMI_CHASSIS_VENDOR, "TUXEDO"),\n'
       '\t\t},\n\t},\n\t{\n\t\t.matches = {\n\t\t\tDMI_MATCH(DMI_SYS_VENDOR, "MONSTER"),\n'
       '\t\t},\n\t},\n\t{ }\n};')

if old not in content:
    print("ERROR: Could not find insertion point. Source may differ from expected version.")
    sys.exit(1)

with open(path, 'w') as f:
    f.write(content.replace(old, new, 1))
print("Patch applied.")
EOF
```

## Step 4 — Rebuild the DKMS Module

```bash
KERNEL=$(uname -r)
sudo dkms unbuild tuxedo-drivers/4.22.2 -k "$KERNEL"
sudo dkms build   tuxedo-drivers/4.22.2 -k "$KERNEL"
sudo dkms install tuxedo-drivers/4.22.2 -k "$KERNEL" --force
```

## Step 5 — Configure Modules to Load at Boot

```bash
sudo tee /etc/modules-load.d/tuxedo.conf << 'EOF'
tuxedo_compatibility_check
tuxedo_keyboard
clevo_acpi
clevo_wmi
tuxedo_io
ite_829x
EOF
```

## Step 6 — Load the Modules Now

```bash
sudo modprobe -r tuxedo_keyboard tuxedo_compatibility_check 2>/dev/null || true
sudo modprobe tuxedo_keyboard
for m in clevo_acpi clevo_wmi tuxedo_io ite_829x; do
  sudo modprobe "$m" 2>/dev/null || true
done
```

> **Correction (2026-08-13):** the original one-liner here —
> `sudo modprobe clevo_acpi clevo_wmi tuxedo_io ite_829x` — silently only loads
> `clevo_acpi`. Unlike `modprobe -r`, plain `modprobe` only accepts **one**
> module name; anything after it is parsed as module *parameters*, not
> additional modules to load. It fails quietly here because of the trailing
> `2>/dev/null || true`. Confirmed via `lsmod` on the Arch install below —
> `clevo_wmi`, `tuxedo_io`, and `ite_829x` never actually loaded until each
> was `modprobe`d individually.

## Step 7 — Set Keyboard Backlight to Full Brightness

```bash
cat /sys/class/leds/rgb:kbd_backlight/max_brightness \
  | sudo tee /sys/class/leds/rgb:kbd_backlight/brightness
```

If the sysfs path is missing, it will appear after reboot once the modules load correctly.

## Step 8 — Unlock the Touchpad via EC

```bash
sudo python3 << 'EOF'
import fcntl, ctypes, struct, time, sys

def _IOC(d, t, n, s): return (d << 30) | (s << 16) | (t << 8) | n
ptr = ctypes.sizeof(ctypes.c_void_p)
R_TP = _IOC(2, 0xED, 0x15, ptr)
W_TP = _IOC(1, 0xEE, 0x14, ptr)

for _ in range(5):
    try:
        fd = open("/dev/tuxedo_io", "rb+", buffering=0)
        buf = ctypes.create_string_buffer(8)
        fcntl.ioctl(fd, R_TP, buf)
        state = struct.unpack("i", buf[:4])[0]
        if state == 0:
            print(f"  EC touchpad state was: {state} (disabled) — enabling...")
            wb = ctypes.create_string_buffer(struct.pack("i", 1) + b'\x00'*4)
            fcntl.ioctl(fd, W_TP, wb)
            fcntl.ioctl(fd, R_TP, buf)
            print(f"  EC touchpad state now: {struct.unpack('i', buf[:4])[0]} (1=enabled)")
        else:
            print(f"  Touchpad EC state: {state} (already enabled)")
        fd.close()
        sys.exit(0)
    except Exception as e:
        time.sleep(1)
print("  Could not reach tuxedo_io — touchpad state unchanged.")
EOF
```

## Step 9 — Install a Persistent Touchpad-Enable Service

This service re-enables the touchpad automatically on every boot.

**Create the helper script:**

```bash
sudo tee /usr/local/bin/tuxedo-touchpad-enable.py << 'EOF'
#!/usr/bin/env python3
import fcntl, ctypes, struct, time, sys

def _IOC(d, t, n, s): return (d << 30) | (s << 16) | (t << 8) | n
W_TP = _IOC(1, 0xEE, 0x14, ctypes.sizeof(ctypes.c_void_p))

for _ in range(5):
    try:
        fd = open("/dev/tuxedo_io", "rb+", buffering=0)
        fcntl.ioctl(fd, W_TP, ctypes.create_string_buffer(struct.pack("i", 1) + b'\x00'*4))
        fd.close()
        sys.exit(0)
    except:
        time.sleep(1)
sys.exit(1)
EOF
sudo chmod +x /usr/local/bin/tuxedo-touchpad-enable.py
```

**Create the systemd service:**

```bash
sudo tee /etc/systemd/system/tuxedo-touchpad-enable.service << 'EOF'
[Unit]
Description=Enable Clevo/MONSTER touchpad via EC
After=systemd-modules-load.service
Requires=systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/usr/bin/python3 /usr/local/bin/tuxedo-touchpad-enable.py
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable tuxedo-touchpad-enable.service
```

## Step 10 — Update initramfs and Reboot

```bash
sudo update-initramfs -u -k "$(uname -r)"
sudo reboot
```

---

## Verify After Reboot

```bash
# Keyboard backlight sysfs should exist
cat /sys/class/leds/rgb:kbd_backlight/brightness

# Tuxedo modules should be loaded
lsmod | grep tuxedo

# Touchpad service should be active
systemctl status tuxedo-touchpad-enable.service
```

---

## Where the Backlight Control Lives in Tuxedo Control Center

The slider, RGB color picker and per-zone selector are **not** on the TCC Dashboard. They live under:

> **TCC sidebar → Tools → Keyboard Backlight**

Confirm tccd detected the keyboard:

```bash
journalctl -u tccd | grep -i KeyboardBacklight
# Expect:
#   KeyboardBacklightListener: Detected RGB zone keyboard backlight
#   KeyboardBacklightListener: initUPower: Using /org/freedesktop/UPower/KbdBacklight
```

TCC talks to the LED through the **UPower DBus interface**, not sysfs directly. Quick sanity check:

```bash
gdbus call --system --dest org.freedesktop.UPower \
  --object-path /org/freedesktop/UPower/KbdBacklight \
  --method org.freedesktop.UPower.KbdBacklight.GetMaxBrightness   # → (255,)
gdbus call --system --dest org.freedesktop.UPower \
  --object-path /org/freedesktop/UPower/KbdBacklight \
  --method org.freedesktop.UPower.KbdBacklight.GetBrightness      # → (current,)
```

### "Control is enabled but the LED is off at boot"

TCC persists the last-applied backlight state in `/etc/tcc/settings` under `keyboardBacklightStates`. If `brightness: 0` is saved there, the LED comes up dark on every boot. Move the slider in TCC once and it sticks — the daemon writes back to `settings` automatically. To bootstrap from the CLI:

```bash
echo 200 | sudo tee /sys/class/leds/rgb:kbd_backlight/brightness
# Then in TCC: Tools → Keyboard Backlight → nudge the slider so tccd saves it.
```

`keyboardBacklightControlEnabled` lives in **Global Settings** (sidebar) — leave it on.

---

## Manual Keyboard Backlight Controls

```bash
# Check current brightness (0–255)
cat /sys/class/leds/rgb:kbd_backlight/brightness

# Set brightness (replace 200 with any value 0–255)
echo 200 | sudo tee /sys/class/leds/rgb:kbd_backlight/brightness

# Turn off
echo 0 | sudo tee /sys/class/leds/rgb:kbd_backlight/brightness
```

Keyboard shortcuts (hold `Fn`):

| Keys | Action |
|------|--------|
| `Fn` + `/` | Toggle backlight on/off |
| `Fn` + `*` | Cycle colors |
| `Fn` + `+` | Brightness up |
| `Fn` + `-` | Brightness down |

---

## Hyprland: Wiring the Fn Hotkeys (2026-08-13)

The "Manual Keyboard Backlight Controls" table above (`Fn`+`/`, `Fn`+`*`, etc.)
describes GNOME's behavior, where these keys just work once the driver loads
— GNOME's own settings daemon listens for the standard hotkey input events
and acts on them. **Hyprland does none of that automatically.** The driver
still emits correct, standard Linux input events — confirmed by decoding
`/proc/bus/input/devices`' `KEY=` bitmap for the `TUXEDO Keyboard` device
against `/usr/include/linux/input-event-codes.h` — but with zero compositor
config, they're delivered and then go nowhere.

On this board (Tulpar T6 V2.1), the driver source
(`clevo_keyboard.h`) maps them to:

| Physical key | Emitted event | Note |
|---|---|---|
| Touchpad toggle | `KEY_F21` | Driver comment: *"the weirdly named touchpad toggle key that is implemented as KEY_F21 everywhere (instead of KEY_TOUCHPAD_TOGGLE or on/off)"* |
| Kbd brightness up | `KEY_KBDILLUMUP` | |
| Kbd brightness down | `KEY_KBDILLUMDOWN` | |
| Kbd backlight toggle | `KEY_KBDILLUMTOGGLE` | |

None of these change any hardware state by themselves — `sparse_keymap`
drivers only report the event; something in userspace has to act on it.
`tccd` does **not** listen to this device directly (checked: no open fd on
its `event*` node), it only manages brightness via UPower D-Bus once told to.
So the fix is compositor-level binds. Added to
`~/.config/caelestia/hypr-user.lua` (the shell's designated user-override
file, loaded last by `hyprland.lua`):

```lua
local KBD_LED = "/sys/class/leds/rgb:kbd_backlight/brightness"
local KBD_LED_MAX = 255
local KBD_LED_STEP = 32

-- touchpad toggle: flips the same /dev/tuxedo_io EC flag Step 8 reads,
-- but live, on every keypress, instead of once at boot
hl.bind("F21", hl.dsp.exec_cmd(
    "bash -c 'S=$(sudo python3 /usr/local/bin/tuxedo-touchpad-toggle.py); " ..
    "notify-send -u low -i input-touchpad \"Touchpad\" \"$([ \"$S\" = 1 ] && echo Enabled || echo Disabled)\"'"
))

hl.bind("XF86KbdBrightnessUp", hl.dsp.exec_cmd(
    "bash -c 'v=$(cat " .. KBD_LED .. "); n=$((v+" .. KBD_LED_STEP .. ")); " ..
    "[ $n -gt " .. KBD_LED_MAX .. " ] && n=" .. KBD_LED_MAX .. "; " ..
    "echo $n | sudo tee " .. KBD_LED .. " >/dev/null'"
))

hl.bind("XF86KbdBrightnessDown", hl.dsp.exec_cmd(
    "bash -c 'v=$(cat " .. KBD_LED .. "); n=$((v-" .. KBD_LED_STEP .. ")); " ..
    "[ $n -lt 0 ] && n=0; echo $n | sudo tee " .. KBD_LED .. " >/dev/null'"
))

hl.bind("XF86KbdLightOnOff", hl.dsp.exec_cmd(
    "bash -c 'v=$(cat " .. KBD_LED .. "); " ..
    "if [ \"$v\" -gt 0 ]; then echo 0 | sudo tee " .. KBD_LED .. " >/dev/null; " ..
    "else echo " .. KBD_LED_MAX .. " | sudo tee " .. KBD_LED .. " >/dev/null; fi'"
))
```

`F21` and the `XF86Kbd*` names are Hyprland's standard XKB-derived keysym
names for those exact evdev codes — no modifier prefix, since `Fn` is
consumed by the keyboard's own firmware and never reaches the OS as a
separate modifiable event; the keyboard just sends the remapped key directly.

The touchpad toggle calls a small helper (needs root for the `/dev/tuxedo_io`
ioctl, hence `sudo` — passwordless sudo is configured on this machine):

```python
#!/usr/bin/env python3
# /usr/local/bin/tuxedo-touchpad-toggle.py
import fcntl, ctypes, struct, sys

def _IOC(d, t, n, s): return (d << 30) | (s << 16) | (t << 8) | n
ptr = ctypes.sizeof(ctypes.c_void_p)
R_TP = _IOC(2, 0xED, 0x15, ptr)
W_TP = _IOC(1, 0xEE, 0x14, ptr)

try:
    fd = open("/dev/tuxedo_io", "rb+", buffering=0)
    buf = ctypes.create_string_buffer(8)
    fcntl.ioctl(fd, R_TP, buf)
    state = struct.unpack("i", buf[:4])[0]
    new_state = 0 if state else 1
    wb = ctypes.create_string_buffer(struct.pack("i", new_state) + b'\x00' * 4)
    fcntl.ioctl(fd, W_TP, wb)
    fd.close()
    print(new_state)
except Exception as e:
    print(f"touchpad toggle failed: {e}", file=sys.stderr)
    sys.exit(1)
```

Install with `sudo install -m 0755 tuxedo-touchpad-toggle.py /usr/local/bin/`.

After editing `hypr-user.lua`: `hyprctl reload`, then confirm with
`hyprctl binds | grep -A6 "key: F21"` (or `XF86KbdBrightnessUp`, etc.) —
should show a live `bind` entry with `modmask: 0`. Physical Fn-key presses
aren't verifiable from a shell session; test by hand after reload.

---

## Troubleshooting

**`modprobe: ERROR: could not insert 'tuxedo_keyboard': No such device`**

The compatibility patch wasn't applied or the DKMS rebuild used a cached build. Force a clean rebuild:

```bash
sudo dkms unbuild tuxedo-drivers/4.22.2 -k $(uname -r)
sudo dkms build   tuxedo-drivers/4.22.2 -k $(uname -r)
sudo dkms install tuxedo-drivers/4.22.2 -k $(uname -r) --force
```

**`modprobe: ERROR: could not insert 'tuxedo_keyboard': Operation not permitted`**

Kernel lockdown may be active. Check:

```bash
cat /sys/kernel/security/lockdown
# Should read [none]
```

**Touchpad still locked after reboot**

Check the service status and re-run manually:

```bash
systemctl status tuxedo-touchpad-enable.service
sudo python3 /usr/local/bin/tuxedo-touchpad-enable.py
```
