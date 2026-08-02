# Querying Data in the InfluxDB Dashboard

InfluxDB 2.x includes a built-in UI with a **Data Explorer** and **Dashboards**
that query data using the **Flux** language.

1. Browse to `http://<device-ip>:8086` and sign in with the `IOT_USERNAME` / `IOT_PASSWORD` you set.
2. Go to **Data Explorer** (the graph icon) to build and preview queries, or
   **Dashboards → Create Dashboard → Add Cell** to pin a query to a dashboard.
3. Data written by Telegraf lands in the **`mqtt`** bucket under the measurements
   **`opcua_pubsub`** (live values) and **`opcua_metadata`** (schema/metadata).
4. Switch the view to `Table` to be able to quickly interpret query results.

**Start simple and build up.** The join below is the end goal, but if it returns
nothing, work through these steps first — each one narrows down where the data
stops. *Filtering by station* and the performance notes then follow.

### Step 1: Is anything in the bucket at all?

```flux
from(bucket: "mqtt")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "opcua_pubsub")
  |> limit(n: 10)
```

No rows means the problem is upstream of InfluxDB — check
`kubectl logs -n cloud deployment/telegraf` and confirm the Publisher is
connected to the broker.

### Step 2: Which field names actually exist?

This is the step that most often explains an empty result — field names are
derived from your OPC UA node names, so they differ per deployment:

```flux
from(bucket: "mqtt")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "opcua_pubsub")
  |> keep(columns: ["_field"])
  |> group()
  |> distinct(column: "_field")
```

For the simulated Munich line you should see names such as
`Payload_NumberOfManufacturedProducts_Value`, `Payload_NumberOfDiscardedProducts_Value`
and `Payload_FaultyTime_Value`; the Modbus asset adds `Payload_Temperature_Value`,
`Payload_Pressure_Value`, `Payload_MotorSpeed_Value` and so on. Pick one from
**your** list for the next step.

### Step 3: Query that one field

```flux
from(bucket: "mqtt")
  |> range(start: -1h)
  |> filter(fn: (r) =>
    r._measurement == "opcua_pubsub" and
    r._field == "Payload_Pressure_Value"      // ← replace with a name from Step 2
  )
```

If Step 2 listed the field but this returns nothing, widen the time range —
`v.timeRangeStart` follows the dashboard's selector, which may be narrower than
the data you have.

### Step 4: Join with the metadata for human-readable labels

Once Step 3 returns rows, add the metadata join. It labels each series with its
`metaName` by matching on `datasetWriterId`:

```flux
data =
  from(bucket: "mqtt")
    |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
    |> filter(fn: (r) =>
      r._measurement == "opcua_pubsub" and
      r._field == "Payload_Pressure_Value"    // ← replace with a name from Step 2
    )
    |> keep(columns: ["_time", "_value", "datasetWriterId"])
    |> group(columns: ["datasetWriterId"])
    |> sort(columns: ["_time"])

meta =
  from(bucket: "mqtt")
    |> range(start: -2d)                       // ← see the note on lookback below
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

> **If the join returns nothing but Step 3 worked**, the two halves have no
> `datasetWriterId` in common. Metadata is only republished every
> `MetadataSendInterval` seconds (300 by default), so shortly after an InfluxDB restart the
> `opcua_metadata` measurement may not yet contain an entry for the writer you are
> querying. Either wait for the next metadata publish or widen the `meta` range.
> You can check both sides with:
>
> ```flux
> from(bucket: "mqtt")
>   |> range(start: -2d)
>   |> filter(fn: (r) => r._measurement == "opcua_metadata")
>   |> keep(columns: ["datasetWriterId", "metaName"])
>   |> group()
>   |> distinct(column: "datasetWriterId")
> ```

> **Choosing the metadata lookback.** The `meta` range is a resilience dial, not a
> performance one: it is effectively *how long the Publisher may stop publishing
> metadata before this query returns nothing*. `-2d` matches the provisioned
> Grafana dashboards. Shortening it to something like `-1h` looks harmless and is
> the usual cause of a join that worked yesterday and returns nothing today.

### Filtering by station

`datasetWriterId` is an opaque numeric hash, so to select a specific station you
look its writers up in `opcua_metadata` first — the `metaName` tag holds
`<ApplicationUri>;<NodeId>`, and the application URI contains the station name:

```flux
import "strings"
import "regexp"

station   = "assembly"
stationRx = regexp.compile(v: station)

writers =
  from(bucket: "mqtt")
    |> range(start: -2d)
    |> filter(fn: (r) => r._measurement == "opcua_metadata")
    |> filter(fn: (r) => r.metaName =~ stationRx)
    |> keep(columns: ["datasetWriterId"])
    |> group()
    |> distinct(column: "datasetWriterId")
    |> findColumn(fn: (key) => true, column: "_value")

writersRx = regexp.compile(v: "^(" + strings.joinStr(arr: writers, v: "|") + ")$")

from(bucket: "mqtt")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "opcua_pubsub" and r._field == "Payload_Pressure_Value")
  |> filter(fn: (r) => r.datasetWriterId =~ writersRx)
```

> **Use a regex to filter tags, not `contains()` or `strings.containsStr()`.**
> InfluxDB pushes a regex predicate on a *tag* down into the storage engine, so
> only matching series are read. `contains()` and `strings.containsStr()` are
> evaluated row-by-row in Flux and force a full scan. On the reference hardware
> this distinction was worth tens of seconds per panel. The same pattern is used
> by every provisioned dashboard query — see
> [Calculating OEE](./tutorial-oee.md#query-performance-notes).

> **Tip:** Use the Data Explorer's visual **Query Builder** to discover the exact
> `_field` names available (they mirror the OPC UA PubSub payload keys, e.g.
> `Payload_<NodeName>_Value`), then switch to the **Script Editor** to refine the
> Flux and save the cell to a dashboard. Set the cell's refresh interval and time
> range at the top of the dashboard for live monitoring.

---

[< Back to the main README](./README.md)
