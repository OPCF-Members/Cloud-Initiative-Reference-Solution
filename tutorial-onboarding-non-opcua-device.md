# Onboarding a Non-OPC UA Device

The UA Edge Translator uses **W3C Web of Things (WoT) Thing Descriptions (TDs)**
to model a non-OPC UA asset (e.g. Modbus TCP, LoRaWAN, OCPP, or an HTTP/REST
device) and expose its data points as OPC UA nodes. Once mapped, the device is
published exactly like a native OPC UA device.

1. Open the **UA Cloud Publisher** UI at `http://<device-ip>:8081` (log in with
   the `IOT_USERNAME` / `IOT_PASSWORD` you set, exposed via the manifest
   `PUBLISHER_USERNAME` / `PUBLISHER_PASSWORD` env vars).
2. Go to **UA Edge Translator** and provide a **WoT Thing Description** for your asset.
   The Thing Description declares:
   - the **protocol binding** (e.g. `modbus+tcp://<device-ip>:502`, an OCPP/
     LoRaWAN endpoint, or an HTTP base URL),
   - the device's **properties/telemetry** (each becomes an OPC UA variable), and
   - the **datatype, access, and addressing** (e.g. Modbus register/coil, unit id)
     for each property. 
   
   If your device vendor didn't supply a Thing Description, see [Generating WoT Thing Descriptions from PLC Engineering Tools](https://github.com/OPCFoundation/UA-EdgeTranslator#generating-wot-thing-descriptions-from-plc-engineering-tools). and [Generating a WoT Thing Description for a Foxed-Function Asset](https://github.com/OPCFoundation/UA-EdgeTranslator#generating-a-thing-description-for-a-fixed-function-asset) on how to generate a Thing Description.
   
   Uploaded TDs are persisted on the Pi under `/translator/settings`.
   The Edge Translator connects to the asset over its native protocol and instantiates the mapped data points as OPC
   UA nodes in its address space (served at `opc.tcp://<device-ip>:4840`).

   All asset tags specified in the Thing Description as properties are automatically published.

---

[< Back to the main README](./README.md)
