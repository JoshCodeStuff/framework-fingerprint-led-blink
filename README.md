# framework-fingerprint-led-blink

Blink the **power-button LED** of a Framework laptop while the fingerprint
reader is waiting for your finger.

<p align="center">
  <img src="docs/demo.gif" alt="Framework Laptop 16 power button blinking while waiting for a fingerprint" width="360">
  <br><sub>Framework Laptop 16 waiting for a fingerprint during <code>sudo</code> (2× speed)</sub>
</p>

On the Framework Laptop 16 the fingerprint reader lives inside the power
button, but nothing tells you when it is actually expecting a touch (sudo,
lock screen, KWallet, polkit…). Neither fprintd, the BIOS, nor the desktop
offer a setting for this, so this small service adds the visual cue.

## How it works

- A root systemd service runs `dbus-monitor` on the system bus and watches the
  fprintd interface `net.reactivated.Fprint.Device`.
- On `VerifyStart` / `EnrollStart` it takes over the LED
  `/sys/class/leds/chromeos:white:power` (exposed by the `leds_cros_ec`
  kernel driver) and blinks it: 0.25 s on / 0.25 s off.
- On `VerifyStop` / `EnrollStop` / `Release`, or a final `VerifyStatus`
  (`done = true`), it stops and hands the LED back to the embedded controller
  (`chromeos-auto` trigger).
- Safety nets: blinking never lasts more than 60 s, and the LED is restored
  when the service stops (including `ExecStopPost`).

Nothing in fprintd, libfprint, PAM or the sensor firmware is modified.

## Install (Fedora)

```bash
git clone https://github.com/alejandroqh/framework-fingerprint-led-blink.git
cd framework-fingerprint-led-blink
./install.sh
```

The installer asks for sudo, installs missing dependencies (`dbus-tools`,
`python3`, `fprintd`), copies the files, restores SELinux labels and enables
the service.

Test it:

```bash
sudo -k; sudo true     # watch the power button before touching it
```

Uninstall:

```bash
./install.sh --uninstall
```

### Installed files

| File | Purpose |
|---|---|
| `/usr/local/sbin/framework-fingerprint-led-blink` | the watcher (Python, stdlib only) |
| `/etc/systemd/system/framework-fingerprint-led-blink.service` | systemd unit |

### Tuning

Edit the constants at the top of `framework-fingerprint-led-blink` (`ON_S`, `OFF_S`,
`MAX_BLINK_S`, `LED`) and re-run `./install.sh`.

## Tested on

Tested and installed on 2026-11-08.

### Software

| Component | Version |
|---|---|
| Fedora | **45** (Forty Five) |
| Kernel | 7.2.15-300.fc45.x86_64 |
| fprintd / fprintd-pam | 1.94.5-6.fc45 |
| libfprint | 1.94.100-1.fc45 |
| PAM | 1.7.2-4.fc45 |
| systemd | 262.1-1.fc45 |
| dbus-broker | 37-9.fc45 |
| dbus-tools (`dbus-monitor`) | 1.16.2-6.fc45 |
| Python | 3.15.0-1.fc45 |
| Desktop | KDE Plasma 6.8.2 (Wayland) |
| SELinux | enforcing |

### Hardware

| Component | Details |
|---|---|
| Laptop | Framework Laptop 16 (AMD Ryzen AI 300 Series), board rev. A9 |
| CPU / GPU | AMD Ryzen AI 9 HX 370 w/ Radeon 890M |
| BIOS | 04.01 (2026-06-12) |
| Fingerprint sensor | Goodix Match-on-Chip, USB `27C6:609C`, firmware `01000334` |
| LED | `chromeos:white:power` (power button, via `leds_cros_ec`) |

It should also work on other Framework models (e.g. Laptop 13) that expose the
same `chromeos:white:power` LED and use fprintd, but that is untested.
Reports welcome.

## Troubleshooting

```bash
systemctl status framework-fingerprint-led-blink
journalctl -u framework-fingerprint-led-blink -b
```

If the LED ever gets stuck, return it to the EC manually:

```bash
echo chromeos-auto | sudo tee /sys/class/leds/chromeos:white:power/trigger
```

## License

Apache License 2.0, see [LICENSE](LICENSE).
