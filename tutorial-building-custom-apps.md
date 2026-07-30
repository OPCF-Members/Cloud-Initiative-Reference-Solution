# Building Custom Apps for the Reference Solution

The reference solution is not a closed appliance: everything it collects and
controls is reachable through open, standards-based interfaces. This tutorial
shows how to build your own applications on top of it using the **OPC UA Web API**
exposed by UA Cloud Action.

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
