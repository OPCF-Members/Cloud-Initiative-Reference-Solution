# Querying Data in the InfluxDB Dashboard

InfluxDB 2.x includes a built-in UI with a **Data Explorer** and **Dashboards**
that query data using the **Flux** language.

1. Browse to `http://<device-ip>:8086` and sign in with the `IOT_USERNAME` / `IOT_PASSWORD` you set.
2. Go to **Data Explorer** (the graph icon) to build and preview queries, or
   **Dashboards → Create Dashboard → Add Cell** to pin a query to a dashboard.
3. Data written by Telegraf lands in the **`mqtt`** bucket under the measurements
   **`opcua_pubsub`** (live values) and **`opcua_metadata`** (schema/metadata).

**Example Flux query** (paste into a Data Explorer / dashboard cell in
**Script Editor** mode). This joins the live `opcua_pubsub` data with the
`opcua_metadata` schema on `datasetWriterId` so each series is labelled with its
human-readable metadata name:

```flux
data =
  from(bucket: "mqtt")
    |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
    |> filter(fn: (r) =>
      r._measurement == "opcua_pubsub" and
      r._field == "Payload_VoltageL-N_Value_C"
    )
    |> keep(columns: ["_time", "_value", "datasetWriterId"])
    |> group(columns: ["datasetWriterId"])
    |> sort(columns: ["_time"])

meta =
  from(bucket: "mqtt")
    |> range(start: -30d)
    |> filter(fn: (r) =>
      r._measurement == "opcua_metadata" and
      r._field == "cfgMajor"
    )
    |> group(columns: ["datasetWriterId"])
    |> last()
    |> keep(columns: ["datasetWriterId", "metaName"])
    |> group(columns: ["datasetWriterId"])

join(tables: {d: data, m: meta}, on: ["datasetWriterId"], method: "inner")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: float(v: r._value),
      _source: r.datasetWriterId,
      _tagName: r.metaName
  }))
```

> **Tip:** Use the Data Explorer's visual **Query Builder** to discover the exact
> `_field` names available (they mirror the OPC UA PubSub payload keys, e.g.
> `Payload_<NodeName>_Value`), then switch to the **Script Editor** to refine the
> Flux and save the cell to a dashboard. Set the cell's refresh interval and time
> range at the top of the dashboard for live monitoring.

---

[< Back to the main README](./README.md)
