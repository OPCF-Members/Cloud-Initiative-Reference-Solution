# Onboarding an OPC UA Device

Use this path when the device already speaks OPC UA (including data that the Edge
Translator has already exposed).

1. Open the **UA Cloud Publisher** UI at `http://<device-ip>:8081` (log in with
   the `IOT_USERNAME` / `IOT_PASSWORD` you set, exposed via the manifest
   `PUBLISHER_USERNAME` / `PUBLISHER_PASSWORD` env vars).
2. Go to **OPC UA Connect** and enter the device's OPC UA endpoint URL, e.g.:
   - The Edge Translator: `opc.tcp://<device-ip>`
   - A standalone OPC UA device: `opc.tcp://<device-address>:<port>`
3. Set the security policy and, if required, the credentials (`OPCUA_USERNAME` /
   `OPCUA_PASSWORD`, i.e. the `IOT_USERNAME` / `IOT_PASSWORD` you set) and click **Connect**.
   > On first connect the client and server exchange certificates. If the UA Cloud Publisher
   > connection is rejected, trust its certificate in the OPC UA device and retry.
   > If the device supports the OPC UA Server Push Configuration model, the
   > Publisher can provision the trust relationship for you automatically — see
   > [Automatic Certificate Provisioning (GDS Server Push)](./README.md#automatic-certificate-provisioning-gds-server-push).
4. **Browse** the device's address space and select the variable nodes you want
   to publish and click the 'publish' button.
5. The Publisher immediately begins sending OPC UA PubSub JSON messages to Mosquitto (`data/#`), and the metadata
   describing each dataset to the `metadata` topic. Confirm data is flowing by checking the InfluxDB `mqtt` bucket (see
   [Querying Data in the InfluxDB Dashboard](./tutorial-influxdb-queries.md)).

---

[< Back to the main README](./README.md)
