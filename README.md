# Enabling AX210 WiFi on Compulab IOT-GATE-IMX8 under balenaOS

## The problem

The board ships with an Intel Wi-Fi 6E AX210 (PCI ID `8086:2725`), but balenaOS for
this device type does not bundle the AX210 firmware, and the full firmware path hits a
driver/interrupt bug on the iMX8 PCIe controller.

Two distinct issues:

1. **Missing firmware.** The image ships only older `iwlwifi` blobs (cc-a0, 9260, 8265).
   There is no `iwlwifi-ty-a0-gf-a0-*.ucode`, so the driver logs
   `no suitable firmware found!`.
2. **PNVM-load timeout.** If you supply the upstream PNVM file
   (`iwlwifi-ty-a0-gf-a0.pnvm`), the firmware loads but init then fails with
   `Timeout waiting for PNVM load!` and `Failed to run INIT ucode: -110`. This is an
   iMX8 PCIe interrupt-delivery issue.

## The fix

Side-load **only the ucode** into balenaOS's persistent `extra-firmware` volume, and
**deliberately omit the PNVM**. Without the PNVM the driver falls back to hardware
defaults and brings up `wlan0` for 2.4GHz and 5GHz. (You lose 6GHz / Wi-Fi 6E, which is
irrelevant for a gateway.)

The ucode keeps its honest upstream name (`-59`). The driver searches filenames from
`-66` down to `-39` and loads the first match, so the real `-59` file is found on the
way down. No renaming needed.

---

## Requirements

| Requirement | Minimum         |
| ----------- | --------------- |
| balenaOS    | v6.10.7 or higher |
| Supervisor  | v17.5.2 or higher |

Confirm on a running device with `cat /etc/os-release`. Older balenaOS does not have the
extra-firmware feature and the label/volume will silently do nothing.

You also need, on a **Linux** machine (the preload step uses Docker and the device-type
storage driver; Docker Desktop on Mac/Windows trips on this):

- balena CLI (`balena login` done)
- Docker running locally
- A balenaCloud fleet for the iot-gate-imx8

---

## Part 1: Build the firmware service and create a fleet release

Create this project structure:

```
my-fleet/
├── docker-compose.yml
└── wifi-firmware/
    ├── Dockerfile
    └── run.sh
```

### docker-compose.yml

```yaml
version: '2.4'

services:
  wifi-firmware:
    build: ./wifi-firmware
    labels:
      io.balena.features.extra-firmware: '1'
      io.balena.features.supervisor-api: '1'
    restart: always
```

> Do **not** use the prebuilt `bh.cr/balena_os/linux-firmware-aarch64` block. It mirrors
> upstream linux-firmware, which includes the AX210 PNVM, and that file reintroduces the
> timeout bug. You need a custom service that installs the ucode only.

### wifi-firmware/Dockerfile

```dockerfile
FROM alpine
RUN apk add --no-cache curl
# bake the ucode in at build time so first boot needs no network
RUN curl -L -o /iwlwifi-ty-a0-gf-a0-59.ucode \
    "https://raw.githubusercontent.com/navronics/linux-firmware/main/iwlwifi-ty-a0-gf-a0-59.ucode"
COPY run.sh /run.sh
RUN chmod +x /run.sh
CMD ["/run.sh"]
```

### wifi-firmware/run.sh

```sh
#!/bin/sh
set -e

# Install the ucode into the extra-firmware volume.
# Do NOT copy any .pnvm file - it triggers the iMX8 PNVM-load timeout.
cp -n /iwlwifi-ty-a0-gf-a0-59.ucode /extra-firmware/

# The iwlwifi driver probes early in boot, before this container runs, so on the
# very first boot it won't find the firmware yet. Reboot once so the driver
# re-probes and picks up the file. The guard file (on the persistent volume)
# ensures we only reboot once and never loop.
if [ ! -f /extra-firmware/.fw-installed ]; then
  touch /extra-firmware/.fw-installed
  curl -X POST \
    -H "Content-Type:application/json" \
    -H "Authorization: Bearer $BALENA_SUPERVISOR_API_KEY" \
    "$BALENA_SUPERVISOR_ADDRESS/v1/reboot"
fi

sleep infinity
```

### Push it

```bash
cd my-fleet
balena login
balena push <your-fleet-name>
```

Confirm in the dashboard that the release built and is active.

---

## Part 2: Download and configure the OS image (WiFi credentials baked in)

Because these devices may have no ethernet, the WiFi credentials must be embedded so the
device joins your network on first boot.

```bash
balena os download iot-gate-imx8 -o balena.img --version default

balena os configure balena.img --fleet <your-fleet-name> \
  --config-network=wifi \
  --config-wifi-ssid "YOUR_SSID" \
  --config-wifi-key "YOUR_PASSWORD"
```

If those flag names error on your CLI version, run `balena os configure --help` or just
run it interactively; it will prompt for network type and WiFi credentials.

---

## Part 3: Preload the fleet release into the image

This bakes your firmware service onto the image so it runs offline on first boot.

```bash
balena preload balena.img --fleet <your-fleet-name> --commit current
```

`--commit current` embeds the fleet's latest release.

---

## Part 4: Flash and deploy

1. Write `balena.img` to a USB drive with Etcher (or `balena local flash`).
2. Boot the IOT-GATE-IMX8 from the USB drive.
3. It flashes itself to internal eMMC, then powers down.
4. Remove the USB drive and boot from internal storage.

On that first offline boot:

- the firmware service copies the ucode into `/extra-firmware`,
- it triggers a single reboot via the Supervisor API,
- after reboot the `iwlwifi` driver finds the firmware and `wlan0` comes up,
- NetworkManager joins your WiFi with the baked-in credentials,
- the device provisions to balenaCloud over WiFi.

Ethernet is never required.

---

## Verifying on a device

```bash
# interface should exist and have an IP
ifconfig wlan0

# driver log, filtered
dmesg | grep -i iwlwifi | grep -iv "Direct firmware load"

# manual connect if needed
nmcli dev wifi connect "YOUR_SSID" password "YOUR_PASSWORD"
```

A healthy log shows `loaded firmware version 59.601f3a66.0` and the card initialising,
with **no** `Timeout waiting for PNVM load!` line.

---

## Notes and caveats

- **Regulatory domain.** Without the PNVM the country setting may default conservatively.
  Check `iw reg get`; if channels are missing, set `"country": "AU"` (or your locale) in
  `config.json`.
- **6GHz.** Omitting the PNVM disables Wi-Fi 6E. 2.4GHz and 5GHz are unaffected.
- **Per-network credentials.** Devices going to different WiFi networks need Part 2
  re-run per credential set, but the same preloaded release (Parts 1 and 3) is reused.
- **Offline reflash wipes volumes.** If a device later goes through balena's offline
  update/reflash, preloaded named volumes are wiped and the firmware service reinstalls
  on next boot. Normal first-boot provisioning is unaffected.
- **This is still a workaround.** Filing an issue with balena (bundle the AX210 ucode
  natively; carry the PNVM interrupt patch) is what gets this fixed in the OS image so
  the side-load is no longer needed.

---

## One-off manual fix (single device, already running)

If you just want one device working now without the fleet/preload flow:

```bash
DEST=/var/lib/docker/volumes/extra-firmware/_data
curl -L -o $DEST/iwlwifi-ty-a0-gf-a0-59.ucode \
  "https://raw.githubusercontent.com/Netronome/linux-firmware/master/iwlwifi-ty-a0-gf-a0-59.ucode"
file $DEST/iwlwifi-ty-a0-gf-a0-59.ucode          # must say "data"
rm -f $DEST/iwlwifi-ty-a0-gf-a0.pnvm*            # ensure no PNVM present
reboot
```

After reboot, `ifconfig wlan0` should show the interface.
