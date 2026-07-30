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
and drive Commander over Mosquitto:

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

## Accessing the OPC UA Web API (UA Cloud Action)

In addition to the MQTT-based feedback loop, **UA Cloud Action** exposes an
**OPC UA Web API** — a RESTful, [OpenAPI](https://swagger.io/specification/)-based
HTTP interface to the standard OPC UA services defined in
[IEC 62541-4](https://reference.opcfoundation.org/Core/Part4/v105/docs/). This
lets you build **custom applications** — dashboards, mobile apps, analytics jobs,
or backend integrations — that talk to the edge's OPC UA servers over plain
HTTP/JSON, without embedding a native OPC UA stack.

> **Implemented services:** this reference deployment implements the
> **`Read`**, **`HistoryRead`** (historical read), and **`Browse`** services of
> the OPC UA Web API. Other services (`Write`, `Call`, etc.) are **not implemented** due to security considerations — requests to them are not available. Use UA Cloud Commander (the
> MQTT command path) for `Write`/`Call` operations in the meantime.

> **Note:** the Web API is being implemented in UA Cloud Action. The endpoints and
> auth described below track the OPC UA Web API specification; confirm the exact
> routes against the running service's OpenAPI/Swagger document.

## Reaching the Web API

The API is served by the `ua-cloudaction` container and reachable through its
Service at:

```
http://<device-ip>:8082
```

- The **OpenAPI/Swagger** definition (e.g. `http://<device-ip>:8082/swagger`)
  describes every available route and schema — point your tooling at it to explore
  or generate clients. In the interactive **Swagger UI**, use the **Authorize**
  button to supply the Basic credentials before invoking the `/opcua/read` and
  `/opcua/historyread` operations.
- **Authentication is mandatory.** The UA Cloud Action web UI and Web API require
  **HTTP Basic authentication** on every request — there is no anonymous access.
  Supply the `IOT_USERNAME` / `IOT_PASSWORD` credentials (set via the manifest's
  `ADMIN_USERNAME` / `ADMIN_PASSWORD`) in the HTTP `Authorization: Basic <base64>`
  header. Unauthenticated requests receive `401 Unauthorized` with a
  `WWW-Authenticate: Basic` challenge.
  > The OPC UA Web API specification also allows bearer/JWT tokens in the
  > `Authorization` header (the reference gateway uses OAuth2 JWTs); Basic auth is
  > what this reference deployment mandates.
- **The API is rate limited** per client IP using a fixed window. The limit and
  window are configurable via the `RATE_LIMIT_PERMIT` and
  `RATE_LIMIT_WINDOW_SECONDS` environment variables (defaulting to **60 requests
  per 60 seconds** in `cloud.yaml`). Requests exceeding the limit receive
  `429 Too Many Requests`.
- Behind the OPC UA Web API, UA Cloud Action forwards the requested service to the
  target OPC UA server (e.g. the Edge Translator at
  `opc.tcp://ua-edgetranslator.edge.svc.cluster.local:4840`).

## Building Custom Apps with the Starter Kit

Use the OPC Foundation
[UA Web API Starter Kit](https://github.com/OPCFoundation/UA-WebApi-StarterKit/tree/master/UaWebApiClient)
as the starting point. Its `UaWebApiClient` folder contains ready-to-run sample
clients in several environments that call the OPC UA Web API using pre-built
**stubs** (classes/constants for the OPC UA services, `BrowseName`s, and
`NodeId`s):

| Sample client | Language / stubs |
|---------------|------------------|
| `UaWebApiClient/csharp` | C# — [DotNet stubs](https://github.com/OPCFoundation/opcua-webapi-dotnet) |
| `UaWebApiClient/nodejs` | JavaScript — [TypeScript stubs](https://github.com/OPCFoundation/opcua-webapi-typescript) |
| `UaWebApiClient/react`  | TypeScript (browser) — TypeScript stubs |
| `UaWebApiClient/python` | Python — [Python stubs](https://github.com/OPCFoundation/opcua-webapi-python) |

Typical workflow to build your own app:

1. **Clone the starter kit** and pick the sample client that matches your stack:
   ```bash
   git clone https://github.com/OPCFoundation/UA-WebApi-StarterKit.git
   cd UA-WebApi-StarterKit/UaWebApiClient
   ```
2. **Point the client at your Web API endpoint** — set its base URL to
   `http://<device-ip>:8082` (the UA Cloud Action Web API) and set the
   `Authorization: Basic <base64(user:pass)>` header (mandatory) using your
   `IOT_USERNAME` / `IOT_PASSWORD`.
3. **Use the pre-built stubs** to invoke OPC UA services with typed
   requests/responses instead of hand-crafting JSON. In this deployment the
   available operations are **`Read`**, **`HistoryRead`**, and **`Browse`**
   (reading current and historical variable values and browsing the address
   space); `Write` and `Call` are not yet exposed by the Web API.
4. **(Optional) Generate model-specific classes.** For DataTypes from a custom
   information model, convert its NodeSet to an OpenAPI schema with the
   [Opc.Ua.ModelCompiler](https://github.com/OPCFoundation/UA-ModelCompiler), then
   generate typed classes with the
   [OpenAPI Generator](https://openapi-generator.tech/) — exactly as the starter
   kit does — so your app understands the model's structured values.
5. **Iterate from a sample** — start from the closest `UaWebApiClient` sample and
   extend it into your own dashboard, service, or integration.

> Because the interface is standard OpenAPI, you can also generate a client in any
> other language supported by the OpenAPI Generator directly from the service's
> OpenAPI document, rather than using the pre-built stubs.

---

[< Back to the main README](./README.md)
