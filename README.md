# Cloud-Initiative-Reference-Solution

OPC Foundation Cloud Initiative Open-Source Reference Solution

## Reference Edge Hardware

The reference solution is validated on a compact, fanless industrial PC built
around the Raspberry Pi Compute Module 5 (CM5). This platform provides an
industrial-grade, DIN-rail-mountable edge gateway suitable for running the
OPC UA / cloud reference workloads.

### Bill of Materials (Purchasing)

Purchase the following components from Waveshare. The CM5 module and the
NVMe SSD are **not** included with the enclosure and must be ordered separately.
A USB-to-M.2 adapter is required to image the SSD from a separate PC before it is
installed into the enclosure.

| # | Component | Description | Product Page |
|---|-----------|-------------|--------------|
| 1 | **IPCBox-CM5-A** | Industrial computer / enclosure kit for the Raspberry Pi Compute Module 5 (aluminum-alloy passive-cooling case, carrier board, dual Gigabit Ethernet, USB, dual HDMI, M.2 M-Key NVMe slot, wide-voltage DC input, RTC). | <https://www.waveshare.com/ipcbox-cm5-a.htm> |
| 2 | **Raspberry Pi Compute Module 5** | The system-on-module (BCM2712 quad-core Cortex-A76). Select a variant **without eMMC** (not needed) and RAM (e.g. 8 GB RAM) and optionally wireless to match your needs. | <https://www.waveshare.com/compute-module-5.htm> |
| 3 | **SK NVMe 2242 128G SSD (M.2)** | 128 GB M.2 2242 NVMe SSD used for the operating system and application data storage. | <https://www.waveshare.com/sk-nvme-2242-128g-ssd-m.2.htm> |
| 4 | **USB-to-M.2 (NVMe) adapter / enclosure** | A USB 3.x adapter that accepts an **M-Key M.2 NVMe** SSD (2242 compatible). Used to connect the SSD to a separate PC so it can be imaged with Raspberry Pi Imager before final assembly. | <https://www.waveshare.com/usb-to-sata.htm> |

> **Ordering notes**
> - You will also need (not sold with the kit): a compatible **DC power supply**
>   within the enclosure's rated input range (7V to 36V), an Ethernet cable (if used), and a
>   separate PC (Windows, macOS, or Linux) to run Raspberry Pi Imager.

### Software Installation (Imaging the SSD)

The operating system is written to the NVMe SSD from a **separate PC** using the
USB-to-M.2 adapter and Raspberry Pi Imager. Do this before assembling the unit.

1. **Insert the SSD into the USB-to-M.2 adapter.** Seat the SK NVMe 2242 128G SSD
   in the adapter's M-Key slot and secure it, then plug the adapter into a USB 3.x
   port on your PC. The drive should enumerate as a USB mass-storage device.
2. **Install Raspberry Pi Imager** on the PC from
   <https://www.raspberrypi.com/software/> and launch it.
3. **Select the target device:** click **CHOOSE DEVICE** and pick
   **Raspberry Pi Compute Module 5**.
4. **Select the OS:** click **CHOOSE OS** and pick
   **Raspberry Pi OS Lite (64-bit)** (recommended for a headless edge gateway) or
   the full **Raspberry Pi OS (64-bit)**.
5. **Select the storage:** click **CHOOSE STORAGE** and pick the SK NVMe SSD
   presented through the USB-to-M.2 adapter.
   > ⚠️ Double-check you are selecting the SSD and not another drive on your PC —
   > imaging is destructive and erases the selected device.
6. **Pre-configure OS settings:** click **NEXT → EDIT SETTINGS** and set the
   **hostname**, **username/password**, **Wi‑Fi** (if used), **locale/timezone**,
   and enable **SSH** on the Services tab. This allows a fully headless first boot.
7. **Write and verify.** Click **SAVE → YES → WRITE** and wait for Imager to write
   and verify the image.
8. **Eject** the adapter, remove the SSD, and proceed to the
   [Hardware Installation](#hardware-installation) to assemble the device.

### Hardware Installation

> **Safety:** Power off and disconnect the DC input before opening the case.
> Work on an anti-static surface and handle the CM5 and SSD by their edges.

> **Image the SSD first.** Complete the [Software Installation](#software-installation-imaging-the-ssd)
> steps below to write Raspberry Pi OS onto the NVMe SSD using the USB-to-M.2
> adapter **before** installing the SSD into the enclosure. 

1. **Open the enclosure.** Remove the screws securing the IPCBox-CM5-A cover to
   expose the CM5 carrier board and the M.2 slot.
2. **Install the (already-imaged) NVMe SSD.** Insert the SK NVMe 2242 128G SSD
   into the M.2 M-Key slot at roughly a 30° angle, press it down flat, and secure
   it with the provided M.2 standoff screw.
3. **Install the Compute Module 5.** Align the two high-density connectors on the
   CM5 with the mating connectors on the carrier board and press firmly and
   evenly until the module seats fully. Fasten the CM5 mounting screws.
4. **Apply the thermal interface.** Ensure the thermal pad between the CM5 SoC
   and the aluminum case/heat spreader is in place so the enclosure can act as a
   passive heatsink.
5. **Reassemble the enclosure** and reinstall the cover screws.
6. **Connect peripherals.** Attach the Ethernet cable, and (optionally, for
   first-time console access) an HDMI monitor and USB keyboard.
7. **Connect power.** Wire the DC input within the enclosure's rated voltage
   range (7V to 36V, so a 24V industrial power supply is ideal) to the power terminal / jack and switch it on. The power/status LED
   should illuminate and the device will boot from the NVMe SSD.

#### First-Boot Configuration

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

## Deploying the IoT Stack

The reference workload is defined in [`iot-stack.yaml`](./iot-stack.yaml) and runs
on a lightweight single-node Kubernetes cluster (**K3s**) on the CM5.

### What the Stack Contains

`iot-stack.yaml` deploys the following components, forming an end-to-end pipeline
from industrial protocols to a time-series database:

| Component | Image | Role | Ports |
|-----------|-------|------|-------|
| **ua-edgetranslator** | `ghcr.io/opcfoundation/ua-edgetranslator:main` | OPC Foundation **UA Edge Translator** — connects to southbound assets and translates protocols (LoRaWAN, OCPP, etc.) into an OPC UA information model. Exposes a web UI for configuration. | 4840 (OPC UA server), 5000/5001 (LoRaWAN), 19520/19521 (OCPP), **8080 (web UI)** |
| **ua-cloudpublisher** | `ghcr.io/barnstee/ua-cloudpublisher:main` | **UA Cloud Publisher** — subscribes to OPC UA data (from the edge translator) and publishes it as **OPC UA PubSub** JSON messages to the MQTT broker. Exposes a web UI for configuration. | **8081 (web UI)** |
| **mosquitto** | `eclipse-mosquitto:2.0.18` | **Eclipse Mosquitto** MQTT broker that carries the OPC UA PubSub `data/#` and `metadata` messages between the publisher and Telegraf. Configured via `mosquitto-conf` (anonymous access, listener on 1883). | 1883 (MQTT) |
| **telegraf** | `telegraf:1.37-alpine` | **Telegraf** agent that consumes the MQTT PubSub messages, parses them with the `json_v2` parser (defined in the `telegraf-conf` ConfigMap), and writes them into InfluxDB. Measurements: `opcua_pubsub` (data) and `opcua_metadata` (metadata). | — |
| **influxdb** | `influxdb:2.7` | **InfluxDB 2.7** time-series database that stores the ingested telemetry. Initialized with org `iot`, bucket `mqtt`, and admin user `myUsername`. Exposes a web UI (Data Explorer / dashboards). | **8086 (web UI/API)** |
| **influxdb-auth** (Secret) | — | Holds the `INFLUX_TOKEN` used by InfluxDB (admin token) and Telegraf (write token). Provided at deploy time via the `${INFLUX_TOKEN}` variable. | — |
| **telegraf-conf** / **mosquitto-conf** (ConfigMaps) | — | Configuration for Telegraf (MQTT inputs + InfluxDB output) and Mosquitto respectively. | — |

Data flow:

```
Assets → ua-edgetranslator → ua-cloudpublisher → Mosquitto (MQTT) → Telegraf → InfluxDB
         (OPC UA / protocol  (OPC UA PubSub                         (json_v2   (time-series
          translation)        JSON over MQTT)                        parsing)   storage + UI)
```

> **Security note:** the manifest ships with placeholder credentials
> (`myUsername` / `myPassword`) and anonymous MQTT access for demo purposes.
> Change these before any production or exposed deployment.

### Install K3s on the Pi

Once the device has booted and been updated, install K3s (SSH into the device
first):

```bash
# Install a single-node K3s cluster (server + agent on the same node)
curl -sfL https://get.k3s.io | sh -

# Verify the node is Ready (may take ~30s)
sudo k3s kubectl get nodes
```

To use the standard `kubectl` command and the `KUBECONFIG` without `sudo`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$(id -u):$(id -g)" ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

kubectl get nodes
```

> K3s ships with the **Traefik** ingress controller and a built-in
> **ServiceLB (klipper-lb)** load balancer, so the `type: LoadBalancer`
> Services in the manifest are reachable directly on the node's IP address.

### Apply the Stack Manifest

1. Copy `iot-stack.yaml` onto the device (e.g. with `git clone`, `scp`, or by
   pasting it into a file opened via a text editor like nano).
2. Provide the InfluxDB token. The manifest references `${INFLUX_TOKEN}`, so
   generate one and substitute it at apply time:

   ```bash
   # Generate a random token (or supply your own)
   export INFLUX_TOKEN="$(openssl rand -hex 32)"

   # Substitute the variable and apply
   envsubst < iot-stack.yaml | kubectl apply -f -
   ```

   > `envsubst` is part of the `gettext` package (`sudo apt install -y gettext-base`).
   > Keep the generated `INFLUX_TOKEN` — you'll need it to authenticate Telegraf
   > and to log into InfluxDB via the API.

3. Watch the workloads come up:

   ```bash
   kubectl get pods -w
   kubectl get svc
   ```

   All pods should reach `Running`/`Ready`, and each `LoadBalancer` Service should
   receive an `EXTERNAL-IP` (the node's IP).

   If the external IP address for some Kubernetes services shows as <pending>, use the following command to assign the external IP address of the traefik service: sudo kubectl patch service <theService> -p '{"spec": {"type": "LoadBalancer", "externalIPs":["<the traefik external IP address>"]}}'.

### Where Telemetry Data Is Persisted

The ingested OPC UA telemetry is stored by **InfluxDB** in the `mqtt` bucket. That
database is backed by a Kubernetes `hostPath` volume, so the data lives directly
on the Pi's NVMe SSD at:

```
/influxdb2
```

Because this is a host directory (not ephemeral pod storage), the telemetry
**survives pod restarts, redeploys, and reboots**. A few related persistence
paths are also mapped as `hostPath` volumes on the Pi:

| Path on the Pi | Component | Contents |
|----------------|-----------|----------|
| `/influxdb2` | InfluxDB | Time-series telemetry, buckets, and InfluxDB config (the primary telemetry store). |
| `/translator/settings`, `/translator/nodesets`, `/translator/pki`, `/translator/logs` | UA Edge Translator | Configuration, OPC UA nodesets, certificates, and logs. |
| `/publisher/settings`, `/publisher/pki`, `/publisher/logs` | UA Cloud Publisher | Configuration, certificates, and logs. |

> **Note:** the Mosquitto broker uses an `emptyDir` volume, so its queued
> messages are **not** persisted across pod restarts — durable telemetry lives in
> InfluxDB (`/influxdb2`). Back up `/influxdb2` to preserve historical data, and
> keep the `INFLUX_TOKEN` needed to read it.

## Accessing the Web UIs

Replace `<device-ip>` with the CM5's IP address (from `ip addr` or
`kubectl get svc`). All services are exposed as `LoadBalancer` types on the node.

| Service | URL | Notes |
|---------|-----|-------|
| **UA Edge Translator** | `http://<device-ip>:8080` | Configure southbound asset connections and the OPC UA information model. Log in with `myUsername` / `myPassword` (from the manifest env vars). |
| **UA Cloud Publisher** | `http://<device-ip>:8081` | Configure which OPC UA nodes to publish and the MQTT broker target (`mosquitto.default.svc.cluster.local:1883`). |
| **InfluxDB** | `http://<device-ip>:8086` | Time-series UI, Data Explorer, and dashboards. Log in with `myUsername` / `myPassword` (org `iot`, bucket `mqtt`). |

To keep both UIs reachable on the single node, the manifest publishes the Cloud Publisher Service on host port
 **8081** (mapped to the container's 8080) while the Edge Translator stays on **8080**. No extra steps are needed — just browse to `:8080` and `:8081` respectively.

## Onboarding Devices

Devices are brought into the pipeline in two stages:

1. **UA Edge Translator** exposes device data as an **OPC UA server** (endpoint
   `opc.tcp://<device-ip>:4840`). Native OPC UA devices can be published directly,
   while **non-OPC UA devices** are first *mapped* into the translator's OPC UA
   information model.
2. **UA Cloud Publisher** connects to an OPC UA server (typically the Edge
   Translator, or any other OPC UA device), browses its address space, and
   selects which nodes to publish to MQTT — from where the data flows through
   Telegraf into InfluxDB.

> Before publishing any nodes, configure the Cloud Publisher's **broker
> connection** so it knows where to send the OPC UA PubSub data and metadata —
> see the next section.

### Configuring the Broker Connection and Metadata (UA Cloud Publisher)

The Cloud Publisher must be pointed at the in-cluster **Mosquitto** broker and,
to populate the `opcua_metadata` measurement in InfluxDB, be told to **send OPC
UA metadata**. All of this is done on the **Configuration** page.

1. Open the **UA Cloud Publisher** UI at `http://<device-ip>:8081` and go to
   **Configuration**.
2. Under **Broker Connection**, set the fields to match the stack's Mosquitto
   Service (the broker listens on `1883` with anonymous access and **no TLS**):

   | Field | Value | Notes |
   |-------|-------|-------|
   | **Publisher Name** | `UACloudPublisher` (or any unique name) | Becomes the `PublisherId` tag in InfluxDB. |
   | **Broker URL** | `mosquitto.default.svc.cluster.local` | In-cluster DNS name of the Mosquitto Service. Use the node IP if reaching it externally. |
   | **Broker Port** | `1883` | Plain MQTT port (the manifest exposes 1883). |
   | **Broker Username** | `myUsername` | Mosquitto is configured with `allow_anonymous true`, but the UI requires non-empty credentials — any value works. |
   | **Broker Password** | `myPassword` | Same as above. |
   | **Broker Message Topic** | `data` | Must match Telegraf's `data/#` subscription. |

3. **Uncheck** **Use TLS with Broker** (Mosquitto in this stack is plaintext on
   1883 — leaving TLS on will fail the connection).
4. **Uncheck** **Use Kafka** (this stack uses MQTT, not Kafka).
5. Leave **Create Broker SAS Token**, **Use OPC UA certificate for
   authentication**, and **Use custom certificate for authentication** unchecked
   (not needed for anonymous Mosquitto).

To enable metadata (schema) messages that land in the `opcua_metadata`
measurement:

6. Check **Send OPC UA Metadata Messages**.
7. Set **Broker Metadata Topic** to `metadata` — this must match Telegraf's
   `metadata` topic subscription.
8. Leave **Use Alternative Broker For OPC UA Metadata Messages** unchecked so
   metadata goes to the same Mosquitto broker as the data.
9. Click **Apply** to save. The settings are persisted to
    `/publisher/settings/settings.json` on the Pi.

> **Result:** data messages are published to `data` (consumed by Telegraf's
> `data/#` input → `opcua_pubsub`) and metadata messages to `metadata` (consumed
> by Telegraf's `metadata` input → `opcua_metadata`), both landing in the
> InfluxDB `mqtt` bucket.

### Onboarding an OPC UA Device (via UA Cloud Publisher)

Use this path when the device already speaks OPC UA (including data that the Edge
Translator has already exposed).

1. Open the **UA Cloud Publisher** UI at `http://<device-ip>:8081`.
2. Go to **OPC UA Connect** and enter the device's OPC UA endpoint URL, e.g.:
   - The Edge Translator: `opc.tcp://<device-ip>`
   - A standalone OPC UA device: `opc.tcp://<device-address>:<port>`
3. Set the security policy and, if required, the credentials (`OPCUA_USERNAME` /
   `OPCUA_PASSWORD`, default `myUsername` / `myPassword`) and click **Connect**.
   > On first connect the client and server exchange certificates. If the UA Cloud Publisher
   > connection is rejected, trust its certificate in the OPC UA device and retry.
4. **Browse** the device's address space and select the variable nodes you want
   to publish and click the 'publish' button.
5. The Publisher immediately begins sending OPC UA PubSub JSON messages to Mosquitto (`data/#`), and the metadata
   describing each dataset to the `metadata` topic. Confirm data is flowing by checking the InfluxDB `mqtt` bucket (see
   [Querying Data in the InfluxDB Dashboard](#querying-data-in-the-influxdb-dashboard)).

### Onboarding a Non-OPC UA Device (map it in UA Edge Translator first)

The UA Edge Translator uses **W3C Web of Things (WoT) Thing Descriptions (TDs)**
to model a non-OPC UA asset (e.g. Modbus TCP, LoRaWAN, OCPP, or an HTTP/REST
device) and expose its data points as OPC UA nodes. Once mapped, the device is
published exactly like a native OPC UA device.

1. Open the **UA Edge Translator** UI at `http://<device-ip>:8080`.
2. **Add / define the asset** by providing a **WoT Thing Description** for it.
   The Thing Description declares:
   - the **protocol binding** (e.g. `modbus+tcp://<device-ip>:502`, an OCPP/
     LoRaWAN endpoint, or an HTTP base URL),
   - the device's **properties/telemetry** (each becomes an OPC UA variable), and
   - the **datatype, access, and addressing** (e.g. Modbus register/coil, unit id)
     for each property. If your device vendor didn't supply a Thing Description, see [Generating WoT Thing Descriptions from PLC Engineering Tools](https://github.com/OPCFoundation/UA-EdgeTranslator#generating-wot-thing-descriptions-from-plc-engineering-tools). and [Generating a WoT Thing Description for a Foxed-Function Asset](https://github.com/OPCFoundation/UA-EdgeTranslator#generating-a-thing-description-for-a-fixed-function-asset) on how to generate a Thing Description.
   You can upload an existing TD file. Uploaded TDs are persisted on the Pi under `/translator/settings`.
   The Edge Translator connects to the asset over its native protocol and instantiates the mapped data points as OPC
   UA nodes in its address space (served at `opc.tcp://<device-ip>:4840`).
4. **Publish the mapped nodes** by following the
   [Onboarding an OPC UA Device](#onboarding-an-opc-ua-device-via-ua-cloud-publisher)
   steps above, pointing the Cloud Publisher at `opc.tcp://<device-ip>` and
   selecting the newly mapped variables.

> **Summary:** OPC UA devices → publish directly in the Cloud Publisher.
> Non-OPC UA devices → first map them to OPC UA via a WoT Thing Description in the
> Edge Translator, then publish those nodes in the Cloud Publisher. In both cases
> the resulting telemetry lands in InfluxDB via Mosquitto and Telegraf.

### Querying Data in the InfluxDB Dashboard

InfluxDB 2.x includes a built-in UI with a **Data Explorer** and **Dashboards**
that query data using the **Flux** language.

1. Browse to `http://<device-ip>:8086` and sign in (`myUsername` / `myPassword`).
2. Go to **Data Explorer** (the graph icon) to build and preview queries, or
   **Dashboards → Create Dashboard → Add Cell** to pin a query to a dashboard.
3. Data written by Telegraf lands in the **`mqtt`** bucket under the measurements
   **`opcua_pubsub`** (live values) and **`opcua_metadata`** (schema/metadata).

**Example Flux queries** (paste into a Data Explorer / dashboard cell in
**Script Editor** mode):

- Show all published values over the last 15 minutes:

  ```flux
  from(bucket: "mqtt")
    |> range(start: -15m)
    |> filter(fn: (r) => r._measurement == "opcua_pubsub")
  ```

- Filter to a specific publisher and dataset writer:

  ```flux
  from(bucket: "mqtt")
    |> range(start: -1h)
    |> filter(fn: (r) => r._measurement == "opcua_pubsub")
    |> filter(fn: (r) => r.publisher == "MyPublisher")
    |> filter(fn: (r) => r.datasetWriterId == "1")
  ```

- Chart a single numeric field, e.g. a payload value, aggregated per minute:

  ```flux
  from(bucket: "mqtt")
    |> range(start: -6h)
    |> filter(fn: (r) => r._measurement == "opcua_pubsub")
    |> filter(fn: (r) => r._field == "Payload_Temperature_Value")
    |> aggregateWindow(every: 1m, fn: mean, createEmpty: false)
  ```

- Inspect the metadata (schema) messages:

  ```flux
  from(bucket: "mqtt")
    |> range(start: -24h)
    |> filter(fn: (r) => r._measurement == "opcua_metadata")
  ```

> **Tip:** Use the Data Explorer's visual **Query Builder** to discover the exact
> `_field` names available (they mirror the OPC UA PubSub payload keys, e.g.
> `Payload_<NodeName>_Value`), then switch to the **Script Editor** to refine the
> Flux and save the cell to a dashboard. Set the cell's refresh interval and time
> range at the top of the dashboard for live monitoring.

The device is now running the full OPC Foundation Cloud Initiative reference
software stack, from protocol translation at the edge through to a queryable
time-series dashboard.
