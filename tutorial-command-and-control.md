# Command & Control with UA Cloud Commander

While the Publisher → Telegraf → InfluxDB path handles **read-only telemetry**,
**UA Cloud Commander** adds a **command & control** path so a cloud (or local)
application can remotely **read, write, call methods on, and historically read**
OPC UA nodes on the edge — all over the same Mosquitto broker.

Cloud Commander implements the OPC UA PubSub **Actions** request/response pattern
(IEC 62541-14). It acts as the **Responder**: it subscribes to the `commands/#`
topic, executes the requested OPC UA operation against an on-premises OPC UA
server (e.g. the Edge Translator at `opc.tcp://<device-ip>:4840`), and publishes
the result to the `responses` topic.

## How It Is Configured

`ua-cloudcommander` is deployed by `cloud.yaml` and connects to Mosquitto via
these environment variables (no web UI — it is configured entirely through env):

| Variable | Value | Purpose |
|----------|-------|---------|
| `BROKERNAME` | `mosquitto.cloud.svc.cluster.local` | In-cluster Mosquitto Service. |
| `BROKERPORT` | `8883` | TLS MQTT port. |
| `USE_TLS` | `true` | Mosquitto requires TLS. |
| `CLIENTNAME` | `UACloudCommander` | MQTT client id / Responder `PublisherId`. |
| `TOPIC` | `commands/#` | Topic it subscribes to for `ua-action-request` messages. |
| `RESPONSE_TOPIC` | `responses` | Default topic for `ua-action-response` messages (used when a request omits `ResponseAddress`). |
| `USERNAME` / `PASSWORD` | `${IOT_USERNAME}` / `${IOT_PASSWORD}` | Broker credentials (same as the rest of the stack). |
| `UA_USERNAME` / `UA_PASSWORD` | `${IOT_USERNAME}` / `${IOT_PASSWORD}` | Credentials used to sign in to the target OPC UA servers. |

The OPC UA client certificate and logs are persisted on the Pi under
`/commander/pki` and `/commander/logs`.

> **Self-signed TLS:** Mosquitto uses a self-signed certificate generated at pod
> startup. If Cloud Commander rejects the TLS handshake, mount/trust the broker's
> CA certificate in the Commander container (or use a trusted-CA certificate for
> Mosquitto) as recommended in the security note above.

## Sending a Command

Publish a `ua-action-request` NetworkMessage to the `commands` topic and listen
on `responses` for the reply. For example, to **read** a node (from a machine that
can reach the broker, using the stack credentials over TLS):

```bash
# Subscribe for responses in one terminal
mosquitto_sub -h <device-ip> -p 8883 --cafile ca.crt \
  -u "$IOT_USERNAME" -P "$IOT_PASSWORD" -t responses

# Publish a Read request in another terminal
mosquitto_pub -h <device-ip> -p 8883 --cafile ca.crt \
  -u "$IOT_USERNAME" -P "$IOT_PASSWORD" -t commands -m '{
    "MessageId": "32235f26-4a3a-4a56-9f1f-2b6f8a2f0a11",
    "MessageType": "ua-action-request",
    "PublisherId": "MyCloudApp",
    "Timestamp": "2022-11-28T12:01:00.0923534Z",
    "RequestorId": "MyCloudApp",
    "TimeoutHint": 15000,
    "Messages": [
      {
        "DataSetWriterId": 1,
        "ActionTargetId": 1,
        "RequestId": 1,
        "ActionState": 1,
        "Payload": {
          "Endpoint": "opc.tcp://<device-ip>:4840",
          "NodeId": "http://opcfoundation.org/UA/Station/;i=123"
        }
      }
    ]
  }'
```

The `ActionTargetId` selects the operation: **1 = Read**, **2 = HistoricalRead**,
**3 = Write**, **4 = MethodCall**. Cloud Commander replies on `responses` with a
`ua-action-response` message echoing `RequestorId` / `RequestId` and containing
the `Result` (or an `Error`). See the
[UA Cloud Commander documentation](https://github.com/OPCFoundation/UA-CloudCommander)
for the full payload schema of each operation.

## Automated Feedback Loop with UA Cloud Action

**UA Cloud Action** is the automated **Requestor** counterpart to Cloud Commander.
Instead of a human publishing a command, it **polls telemetry and reacts**: on a
15-second loop it queries a configured value and, when it crosses a threshold,
publishes a `ua-action-request` (a `MethodCall`) to the `commands` topic — which
Cloud Commander executes on the target OPC UA server. This closes an edge-local
**digital feedback loop** entirely on the Pi.

In this reference stack it is deployed (`cloud.yaml`) to read from **InfluxDB**
and drive UA Cloud Commander over Mosquitto:

| Variable | Value | Purpose |
|----------|-------|---------|
| `DATA_SOURCE` | `InfluxDB` | Use InfluxDB as the trigger source (instead of Azure Data Explorer). |
| `INFLUX_URL` | `http://influxdb.cloud.svc.cluster.local:8086` | InfluxDB endpoint. |
| `INFLUX_ORG` / `INFLUX_BUCKET` | `iot` / `mqtt` | Org and bucket to query. |
| `INFLUX_MEASUREMENT` | `opcua_pubsub` | Measurement to query. |
| `INFLUX_FIELD` | `Payload_Pressure_Value` | **Numeric field to evaluate — change to match your assets.** |
| `INFLUX_THRESHOLD` | `4000` | Trigger when the latest value exceeds this. |
| `INFLUX_RANGE` | `-1m` | Look-back window for the latest value. |
| `INFLUX_TOKEN` | *(from `influxdb-auth`)* | Query token. |
| `MESSAGING_PLATFORM` | `MQTT` | Reach Commander over MQTT (not Kafka). |
| `MQTT_TARGET` | `UACloudCommander` | Send the full OPC UA PubSub ActionRequest envelope. |
| `BROKER_NAME` / `MQTT_PORT` | `mosquitto…` / `8883` | Mosquitto broker (TLS). |
| `MQTT_USE_TLS` / `MQTT_TLS_INSECURE` | `true` / `true` | TLS with the self-signed broker cert (verification skipped, like Telegraf). |
| `BROKER_USERNAME` / `BROKER_PASSWORD` | `${IOT_USERNAME}` / `${IOT_PASSWORD}` | Broker credentials. |
| `TOPIC` / `RESPONSE_TOPIC` | `commands` / `responses` | The topics Cloud Commander subscribes/replies on. |
| `UA_SERVER_ENDPOINT`, `UA_SERVER_METHOD_ID`, `UA_SERVER_OBJECT_ID`, … | *(placeholders)* | The OPC UA method Cloud Commander invokes on trigger — set these to a real method on your target server (e.g. the Edge Translator). |

> **InfluxDB data source:** support for querying InfluxDB was added to UA Cloud
> Action for this stack (the upstream app queries Azure Data Explorer). The
> `DATA_SOURCE=InfluxDB` branch runs a Flux query for the latest `INFLUX_FIELD`
> value and compares it to `INFLUX_THRESHOLD`. Set `INFLUX_FIELD` to a numeric
> field your assets actually publish (discover exact names in the InfluxDB Data
> Explorer or Grafana), otherwise no trigger will fire.

A small status UI is available at `http://<device-ip>:8082` (log in with your
`IOT_USERNAME` / `IOT_PASSWORD`), showing connectivity to the data source, the
broker, and Cloud Commander.

---

[< Back to the main README](./README.md)
