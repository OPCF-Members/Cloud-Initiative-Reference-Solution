# Reference Edge Hardware (Recommendation)

> This is a **recommendation**, not a requirement. The
> [Cloud-Initiative-Reference-Solution](./README.md) runs on any 64-bit Linux
> host that can run K3s. The hardware below is the platform the reference
> solution has been validated on.

The reference solution is validated on a compact, fanless industrial PC built
around the Raspberry Pi Compute Module 5 (CM5). This platform provides an
industrial-grade, DIN-rail-mountable edge gateway suitable for running the
OPC Foundation Cloud Initiative open-source reference workloads.

<img src="CM5.png" alt="Waveshare IPCBox-CM5" width="25%" />

## Table of Contents

- [Bill of Materials (Purchasing)](#bill-of-materials-purchasing)
- [Software Installation (Imaging the SSD)](#software-installation-imaging-the-ssd)
- [Hardware Installation](#hardware-installation)
  - [First-Boot Configuration](#first-boot-configuration)

## Bill of Materials (Purchasing)

Purchase the following components from Waveshare as a complete kit:

| # | Component | Description | Product Page |
|---|-----------|-------------|--------------|
| 1 | **IPCBox-CM5-A** | Industrial computer / enclosure kit for the Raspberry Pi Compute Module 5 (aluminum-alloy passive-cooling case, carrier board, dual Gigabit Ethernet, USB, dual HDMI, M.2 M-Key NVMe slot, wide-voltage DC input, RTC). | <https://www.waveshare.com/ipcbox-cm5-a.htm> |
| 2 | **Raspberry Pi Compute Module 5** | The system-on-module (BCM2712 quad-core Cortex-A76). Select a variant **without eMMC** (not needed), a minimum of 4GB RAM (8GB recommended for larger production use) and optionally WiFi (see note below about the antenna!) to match your needs. | <https://www.waveshare.com/compute-module-5.htm> |
| 3 | **SK NVMe 2242 128G SSD (M.2)** | 128 GB M.2 2242 NVMe SSD used for the operating system and application data storage. | <https://www.waveshare.com/sk-nvme-2242-128g-ssd-m.2.htm> |
| 4 | **USB-to-M.2 (NVMe) adapter / enclosure** | A USB 3.x adapter that accepts an **M-Key M.2 NVMe** SSD (2242 compatible). Used to connect the SSD to a separate PC so it can be imaged with Raspberry Pi Imager before final assembly. | <https://www.waveshare.com/usb-to-sata.htm> |

## Software Installation (Imaging the SSD)

The operating system is written to the NVMe SSD from a **separate PC** using the
USB-to-M.2 adapter and Raspberry Pi Imager. Do this before assembling the unit.

1. **Insert the SSD into the USB-to-M.2 adapter.** Seat the SK NVMe 2242 128G SSD
   in the adapter's M-Key slot and secure it, then plug the adapter into a USB 3.x
   port on your PC. The drive should enumerate as a USB mass-storage device.
2. **Install Raspberry Pi Imager** on the PC from <https://www.raspberrypi.com/software/> and launch it.
3. **Select Device:** Pick **Raspberry Pi 5**.
4. **Select OS:** Click **Raspberry Pi OS (other)** and pick **Raspberry Pi OS Lite (64-bit)**.
5. **Select Storage:** Pick the SK NVMe SSD presented through the USB-to-M.2 adapter.
   > ⚠️ Double-check you are selecting the SSD and not another drive on your PC —
   > imaging is destructive and erases the selected device.
6. **Customization:** Set the **hostname**, **username/password**, **Wi‑Fi** (if used), **locale/timezone**,
   and enable **SSH** on the Services tab. This allows a fully headless first boot.
7. **Writing:** Click **WRITE** and wait for Imager to write and verify the image.
8. **Finish:** Close the imager, eject the adapter, remove the SSD, and proceed to the
   [Hardware Installation](#hardware-installation) to assemble the device.

## Hardware Installation

> **Image the SSD first.** Complete the [Software Installation](#software-installation-imaging-the-ssd)
> steps above to write Raspberry Pi OS onto the NVMe SSD using the USB-to-M.2
> adapter **before** installing the SSD into the enclosure.

Follow the instructions in the [Assembly Guide](https://docs.waveshare.com/IPCBOX-CM5-A/Assembly-Guide).

> ⚠️ If you bought the CM5 with WiFi, don't forget to [purchase](https://www.amazon.com/Compatible-Antenna-Raspberry-Supports-Frequency/dp/B0D4MKJHXJ) and plug the antenna cable into the CM5 board before assembly!
>
>The CM5's wireless module has **no usable on-board antenna** — unlike a Raspberry
>Pi 5, it relies on an external antenna connected to the U.FL/MHF4 connector on
>the CM5 module. Fitting it is a two-part job: the cable must be physically connected
>*and* the firmware must be told to use that connector, otherwise Wi-Fi will
>either fail to associate or perform very poorly.
>
>1. **Connect the antenna** to the U.FL connector on the CM5 module before
>   closing the enclosure. It is easy to miss, and it cannot be reached
>   afterwards without disassembly.
>2. **Select the external antenna** in firmware by adding this line to
>   `/boot/firmware/config.txt` (`/boot/config.txt` on OS releases older than
>   Bookworm):
>   ```
>   dtparam=ant2
>   ```
>   Then reboot. Without it the module stays on the internal antenna path and the
>   external one is effectively ignored.
>
> **Verify this against your specific module.** `dtparam=ant2` is the documented
> external-antenna selector for the Compute Module family, but antenna wiring
> varies between CM variants and carrier boards. Confirm against the
> [Compute Module antenna documentation](https://www.raspberrypi.com/documentation/computers/compute-module.html#antenna)
> and your carrier board's guide. The practical test is the signal level before
> and after — see step 5 below.

### First-Boot Configuration

1. Power on the assembled device and log in (via the HDMI/keyboard console or over
   SSH using the hostname/credentials configured during imaging):
   ```bash
   ssh <username>@<hostname>.local
   ```
2. Update the system:
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   ```
3. Verify the NVMe SSD is the active root device and the CM5 memory is detected:
   ```bash
   lsblk
   free -h
   ```
4. Confirm network connectivity on the built-in Gigabit Ethernet:
   ```bash
   ip addr
   ping -c3 opcfoundation.org
   ```
   Check that the wired interface — not `wlan0` — carries the address you intend
   to use. `NO-CARRIER` on `eth0` means the cable or switch port is not live:
   ```bash
   ip link show eth0
   ```
5. **Only if you are using Wi-Fi**, verify the antenna is working and turn off
   power saving. Check the signal level and confirm the external antenna is
   actually in use:
   ```bash
   iw wlan0 link
   ```
   Expect better than **−65 dBm**. A weak reading (−75 dBm or worse) while close
   to the access point usually means the antenna is not connected or
   `dtparam=ant2` was not applied.

   Wi-Fi power saving is enabled by default and parks the radio between frames,
   which stalls the long-lived MQTT and OPC UA sessions this stack depends on.
   Turn it off:
   ```bash
   sudo iw wlan0 set power_save off
   iw wlan0 get power_save          # expect: Power save: off
   ```
   That setting does **not** survive a reboot. Make it permanent with a small
   systemd unit (`printf` avoids a heredoc, which would carry this list's
   indentation into the unit file and make it invalid):
   ```bash
   printf '%s\n' \
     '[Unit]' \
     'Description=Disable Wi-Fi power saving on wlan0' \
     'After=network-online.target' \
     'Wants=network-online.target' \
     '' \
     '[Service]' \
     'Type=oneshot' \
     'ExecStart=/usr/sbin/iw wlan0 set power_save off' \
     'RemainAfterExit=yes' \
     '' \
     '[Install]' \
     'WantedBy=multi-user.target' \
     | sudo tee /etc/systemd/system/wlan0-powersave-off.service >/dev/null
   sudo systemctl enable --now wlan0-powersave-off
   ```

> ⚠️ **Use wired Ethernet for production rollouts.** The CM5's Wi-Fi is fine for
> initial bring-up, but this stack is a poor fit for a wireless link in
> production:
>
> - **MQTT and OPC UA are long-lived sessions.** Unlike HTTP request/response
>   traffic, they hold connections open for hours. A brief wireless dropout tears
>   down publisher sessions and OPC UA subscriptions, which then have to
>   reconnect and re-establish security channels.
> - **Throughput is continuous, not bursty.** UA Cloud Publisher emits a PubSub
>   message every publishing cycle, and Telegraf writes every one of them to
>   InfluxDB. Sustained loss shows up as gaps in the historian, which silently
>   distorts the OEE calculations.
> - **Interference is normal on a plant floor.** Machinery, other radios and
>   metal enclosures all degrade 2.4/5 GHz reliability in exactly the
>   environments this device is meant for. The IPC-BOX metal enclosure itself
>   attenuates the signal.
> - **Diagnosis is much harder.** Wireless faults are intermittent and
>   asymmetric, and they surface as puzzling application-level symptoms —
>   stalled sessions, partial writes, timeouts — rather than an obvious link
>   failure.
>
> If Wi-Fi is unavoidable, follow step 5 above — external antenna fitted and
> selected, signal better than −65 dBm, power saving permanently off — and
> prefer 5 GHz. Treat it as a temporary arrangement rather than a deployment.

Once the device is up to date and reachable, continue with
[Deploying the Software Stack](./README.md#deploying-the-software-stack) in the
main README.
