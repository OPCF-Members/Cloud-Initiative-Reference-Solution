# Calculating OEE (Overall Equipment Effectiveness)

**Overall Equipment Effectiveness (OEE)** is the standard KPI for manufacturing
productivity. This tutorial computes it **per station** and **for the whole
production line** from the OPC UA telemetry already flowing into InfluxDB, and
charts the result in Grafana.

It follows the calculation used by the Digital Twin Consortium
[Manufacturing Ontologies](https://github.com/digitaltwinconsortium/ManufacturingOntologies)
reference solution (which implements it in Kusto/KQL against Azure Data Explorer),
re-expressed here as **Flux** queries against InfluxDB.

## Table of Contents

- [The Calculation](#the-calculation)
- [Where the Data Comes From](#where-the-data-comes-from)
- [Step 1: Discover Your Field Names](#step-1-discover-your-field-names)
- [Step 2: OEE for a Single Station](#step-2-oee-for-a-single-station)
- [Step 3: OEE for All Stations](#step-3-oee-for-all-stations)
- [Step 4: OEE for the Whole Line](#step-4-oee-for-the-whole-line)
- [Step 5: Visualise It in Grafana](#step-5-visualise-it-in-grafana)
- [Step 6: OEE Over Time (Trend)](#step-6-oee-over-time-trend)
- [Interpreting the Result](#interpreting-the-result)
- [Adapting This to Real Machines](#adapting-this-to-real-machines)

## The Calculation

OEE is the product of three factors, each a ratio between 0 and 1:

```
OEE = Availability x Performance x Quality
```

| Factor | Formula | Meaning |
|---|---|---|
| **Availability** | `(idealRunningTime - faultyTime) / idealRunningTime` | Share of the period the station was *not* faulted. |
| **Performance** | `idealCycleTime x (produced + scrapped) / (idealRunningTime - faultyTime)` | How close the station ran to its theoretical cycle time. |
| **Quality** | `produced / (produced + scrapped)` | Share of total pieces that were good. |

Where, over the selected time window:

- `idealRunningTime` — length of the window, in **milliseconds**
- `faultyTime` — time the station reported a fault, in **milliseconds**
- `produced` — good pieces made (`NumberOfManufacturedProducts` end − start)
- `scrapped` — bad pieces made (`NumberOfDiscardedProducts` end − start)
- `idealCycleTime` — the station's designed cycle time, in **milliseconds**
  (the Munich simulation runs a **6 s** cycle, so `6000`)

For the **whole line**, Manufacturing Ontologies takes the **minimum** of the
station OEEs — a line can only be as effective as its worst station (the
bottleneck). The MES is excluded because it produces no pieces.

## Where the Data Comes From

Each simulated station publishes these OPC UA variables (from the `Station`
information model), which the
[Simulated Production Line](./README.md#simulated-production-line) already
publishes into the `mqtt` bucket:

| OPC UA variable | NodeId | Used for |
|---|---|---|
| `NumberOfManufacturedProducts` | `i=385` | `produced` |
| `NumberOfDiscardedProducts` | `i=391` | `scrapped` |
| `FaultyTime` | `i=399` | `faultyTime` |
| `OverallRunningTime` | `i=398` | (reference) |
| `Status` | `i=400` | (reference) |
| `EnergyConsumption` | `i=406` | (reference) |

Telegraf writes them to the **`opcua_pubsub`** measurement as fields named
`Payload_<VariableName>_Value`.

### Identifying which station a value came from

UA Cloud Publisher sets `DataSetWriterId` to an **opaque numeric hash**, so you
cannot read the station name off the telemetry directly. The station identity
lives in the **`opcua_metadata`** measurement, whose `metaName` tag holds
`<ApplicationUri>;<NodeId>` — and the application URI contains the station name
(e.g. `assembly`) and the line (e.g. `munich`).

So the queries below first look up the `datasetWriterId` values belonging to a
station, then filter the telemetry by them. This mirrors how the Manufacturing
Ontologies KQL joins `opcua_metadata_lkv` with `opcua_telemetry` on `Subject`.

## Step 1: Discover Your Field Names

Before computing anything, confirm the exact field and tag names in **your**
deployment. Run this in the InfluxDB **Data Explorer** (Script Editor) or Grafana
**Explore**:

```flux
// Which fields exist on the telemetry measurement?
import "influxdata/influxdb/schema"

schema.measurementFieldKeys(bucket: "mqtt", measurement: "opcua_pubsub")
```

```flux
// What do the metadata names look like? (station identity lives here)
from(bucket: "mqtt")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "opcua_metadata")
  |> keep(columns: ["metaName", "datasetWriterId"])
  |> group()
  |> distinct(column: "metaName")
```

You should see fields such as `Payload_NumberOfManufacturedProducts_Value` and
metadata names containing `assembly`, `test`, `packaging`, and `mes`. If your
names differ, adjust the constants at the top of the queries below.

## Step 2: OEE for a Single Station

This returns **Availability, Performance, Quality and OEE** (as percentages) for
one station over the dashboard's selected time range.

```flux
import "strings"
import "array"

// ---- parameters ------------------------------------------------
station          = "assembly"   // "assembly" | "test" | "packaging"
idealCycleTimeMs = 6000.0       // Munich line: 6 s cycle time
bucket           = "mqtt"
// ----------------------------------------------------------------

start = v.timeRangeStart
stop  = v.timeRangeStop

// window length in milliseconds (int(v:) yields nanoseconds)
idealRunningTimeMs = float(v: int(v: stop) - int(v: start)) / 1000000.0

// DataSetWriterIds that belong to this station (from the metadata stream)
writers =
	from(bucket: bucket)
		|> range(start: -30d)
		|> filter(fn: (r) => r._measurement == "opcua_metadata")
		|> filter(fn: (r) => strings.containsStr(v: r.metaName, substr: station))
		|> keep(columns: ["datasetWriterId"])
		|> group()
		|> distinct(column: "datasetWriterId")
		|> findColumn(fn: (key) => true, column: "_value")

// all samples of one field for this station, as floats, in one table
series = (field) =>
	from(bucket: bucket)
		|> range(start: start, stop: stop)
		|> filter(fn: (r) => r._measurement == "opcua_pubsub" and r._field == field)
		|> filter(fn: (r) => contains(value: r.datasetWriterId, set: writers))
		|> toFloat()
		|> group()

// reduce a table to a single number
scalar = (tables=<-, fn) => (tables |> fn() |> findRecord(fn: (key) => true, idx: 0))._value

// counters are cumulative -> take (max - min) across the window
producedField = "Payload_NumberOfManufacturedProducts_Value"
scrappedField = "Payload_NumberOfDiscardedProducts_Value"

produced = scalar(tables: series(field: producedField), fn: max)
		 - scalar(tables: series(field: producedField), fn: min)
scrapped = scalar(tables: series(field: scrappedField), fn: max)
		 - scalar(tables: series(field: scrappedField), fn: min)

// FaultyTime spikes once per fault with that fault's duration -> sum it (see note below)
faultyMs = scalar(tables: series(field: "Payload_FaultyTime_Value"), fn: sum)

runTimeMs   = idealRunningTimeMs - faultyMs
totalPieces = produced + scrapped

availability = if idealRunningTimeMs > 0.0 then runTimeMs / idealRunningTimeMs else 0.0
performance  = if runTimeMs > 0.0 and totalPieces > 0.0 then
				   idealCycleTimeMs * totalPieces / runTimeMs
			   else 0.0
quality      = if totalPieces > 0.0 then produced / totalPieces else 0.0

array.from(
	rows: [
		{
			_time: stop,
			station: station,
			Availability: availability * 100.0,
			Performance: performance * 100.0,
			Quality: quality * 100.0,
			OEE: availability * performance * quality * 100.0,
		},
	],
)
```

> **Why `sum()` is correct for `FaultyTime`.** In the station simulation
> ([`StationNodeManager.cs`](https://github.com/digitaltwinconsortium/ManufacturingOntologies/blob/main/Tools/FactorySimulation/Station/StationNodeManager.cs)),
> `FaultyTime` is **not** a cumulative counter. A `Stopwatch` is started when the
> station faults, stopped when the fault is cleared, and then:
>
> ```csharp
> if (!m_faultClock.IsRunning)
> {
>     m_faultyTime = (ulong)m_faultClock.ElapsedMilliseconds;  // duration of that fault
>     if (m_faultClock.ElapsedMilliseconds != 0)
>     {
>         m_faultClock.Reset();                                // -> next update reports 0
>     }
> }
> ```
>
> So each fault produces a **single spike** carrying that episode's duration in
> milliseconds, after which the value returns to `0`. Summing those spikes over
> the window therefore yields total downtime — which is exactly what the
> Manufacturing Ontologies KQL does. Do **not** use `max - min` here.

> **What actually drives Availability in the simulation.** Faults occur randomly
> (`stationFailure = NormalDistribution(...) > 3.0`, i.e. a rare >3σ event per
> cycle). When a station enters `Fault`, the **MES** — not an external
> application — clears it, after a fixed delay that simulates manual intervention
> ([`Program.cs`](https://github.com/digitaltwinconsortium/ManufacturingOntologies/blob/main/Tools/FactorySimulation/Station/Program.cs)):
>
> ```csharp
> case StationStatus.Fault:
>     // station is at fault state, wait some time to simulate manual intervention before reseting
>     await Task.Delay(c_waitTime);            // c_waitTime = 60 * 1000 -> 60 s
>     await session.CallAsync(RootMethodNode, ResetMethodNode);
> ```
>
> So each fault contributes roughly **60 seconds** of downtime, and Availability is
> governed by how *often* stations fault rather than how long each fault lasts.

> **The pressure loop does not affect OEE here.** UA Cloud Action calls
> `OpenPressureReleaseValve` (`i=435`), which resets the station's slowly-rising
> `Pressure` back to its default. Pressure is independent of the random fault
> generator in this simulation, so that
> [feedback loop](./tutorial-command-and-control.md#automated-feedback-loop-with-ua-cloud-action)
> demonstrates closed-loop control but does **not** move the OEE numbers.

> **Empty windows:** `findRecord` errors if the query returns no rows. If you
> select a time range before the simulation started, widen the range.

## Step 3: OEE for All Stations

Wrap the calculation in a function and evaluate it for each station. This gives
one row per station — ideal for a bar gauge or table panel.

```flux
import "strings"
import "array"

idealCycleTimeMs = 6000.0
bucket           = "mqtt"

start = v.timeRangeStart
stop  = v.timeRangeStop
idealRunningTimeMs = float(v: int(v: stop) - int(v: start)) / 1000000.0

oeeFor = (station) => {
	writers =
		from(bucket: bucket)
			|> range(start: -30d)
			|> filter(fn: (r) => r._measurement == "opcua_metadata")
			|> filter(fn: (r) => strings.containsStr(v: r.metaName, substr: station))
			|> keep(columns: ["datasetWriterId"])
			|> group()
			|> distinct(column: "datasetWriterId")
			|> findColumn(fn: (key) => true, column: "_value")

	series = (field) =>
		from(bucket: bucket)
			|> range(start: start, stop: stop)
			|> filter(fn: (r) => r._measurement == "opcua_pubsub" and r._field == field)
			|> filter(fn: (r) => contains(value: r.datasetWriterId, set: writers))
			|> toFloat()
			|> group()

	scalar = (tables=<-, fn) => (tables |> fn() |> findRecord(fn: (key) => true, idx: 0))._value

	producedField = "Payload_NumberOfManufacturedProducts_Value"
	scrappedField = "Payload_NumberOfDiscardedProducts_Value"

	produced = scalar(tables: series(field: producedField), fn: max)
			 - scalar(tables: series(field: producedField), fn: min)
	scrapped = scalar(tables: series(field: scrappedField), fn: max)
			 - scalar(tables: series(field: scrappedField), fn: min)
	faultyMs = scalar(tables: series(field: "Payload_FaultyTime_Value"), fn: sum)

	runTimeMs   = idealRunningTimeMs - faultyMs
	totalPieces = produced + scrapped

	availability = if idealRunningTimeMs > 0.0 then runTimeMs / idealRunningTimeMs else 0.0
	performance  = if runTimeMs > 0.0 and totalPieces > 0.0 then
					   idealCycleTimeMs * totalPieces / runTimeMs
				   else 0.0
	quality      = if totalPieces > 0.0 then produced / totalPieces else 0.0

	return availability * performance * quality
}

array.from(
	rows: [
		{_time: stop, station: "assembly",  OEE: oeeFor(station: "assembly") * 100.0},
		{_time: stop, station: "test",      OEE: oeeFor(station: "test") * 100.0},
		{_time: stop, station: "packaging", OEE: oeeFor(station: "packaging") * 100.0},
	],
)
```

> The **MES** is deliberately excluded — it schedules shifts and produces no
> pieces, so it has no OEE.

## Step 4: OEE for the Whole Line

The line OEE is the **minimum** of the station OEEs (the bottleneck station
governs the line). Append this to the query from Step 3:

```flux
	// ... same query as Step 3, then:
	|> min(column: "OEE")
	|> map(fn: (r) => ({r with station: "munich (line)"}))
```

Or, as a single self-contained value for a gauge panel, reuse `oeeFor` and take
the smallest of the three:

```flux
	// ... after defining oeeFor as in Step 3:
	a = oeeFor(station: "assembly")
	t = oeeFor(station: "test")
	p = oeeFor(station: "packaging")

	lineOee = if a <= t and a <= p then a else if t <= p then t else p

	array.from(rows: [{_time: stop, line: "munich", OEE: lineOee * 100.0}])
```

## Step 5: Visualise It in Grafana

Open Grafana at `http://<device-ip>:3000` (see
[Dashboards with Grafana](./tutorial-grafana-dashboards.md)) and create a new
dashboard using the pre-provisioned **InfluxDB** data source.

Suggested panels:

| Panel | Query | Visualisation | Settings |
|---|---|---|---|
| **Line OEE** | Step 4 | **Gauge** | Unit `percent (0-100)`, Min `0`, Max `100`, thresholds: red `< 60`, orange `< 85`, green `>= 85` |
| **OEE by station** | Step 3 | **Bar gauge** | Unit `percent (0-100)`, Min `0`, Max `100` |
| **A / P / Q for a station** | Step 2 | **Stat** (3 values) | Unit `percent (0-100)`; add a dashboard variable for `station` |
| **OEE trend** | Step 6 | **Time series** | Unit `percent (0-100)`, Min `0`, Max `100` |

To make the station selectable, add a **dashboard variable** named `station`:

- Type: **Custom**, values: `assembly, test, packaging`

then replace the constant in the query with the Grafana variable:

```flux
station = "${station}"
```

> **Set the time range to a shift.** OEE is only meaningful over a defined
> production period. Use the dashboard time picker to select a shift (the Munich
> simulation runs Morning / Afternoon / Night shifts in `Europe/Berlin`), or add a
> fixed range such as `from: now-8h, to: now`.

## Step 6: OEE Over Time (Trend)

To see OEE develop over the day, compute it per time bucket instead of once for
the whole window. This variant computes **hourly** OEE for one station:

```flux
import "strings"

station          = "assembly"
idealCycleTimeMs = 6000.0
bucket           = "mqtt"
every            = 1h

writers =
	from(bucket: bucket)
		|> range(start: -30d)
		|> filter(fn: (r) => r._measurement == "opcua_metadata")
		|> filter(fn: (r) => strings.containsStr(v: r.metaName, substr: station))
		|> keep(columns: ["datasetWriterId"])
		|> group()
		|> distinct(column: "datasetWriterId")
		|> findColumn(fn: (key) => true, column: "_value")

base = (field) =>
	from(bucket: bucket)
		|> range(start: v.timeRangeStart, stop: v.timeRangeStop)
		|> filter(fn: (r) => r._measurement == "opcua_pubsub" and r._field == field)
		|> filter(fn: (r) => contains(value: r.datasetWriterId, set: writers))
		|> toFloat()
		|> group()

// per-bucket increase of a cumulative counter
increase = (field) =>
	base(field: field)
		|> aggregateWindow(every: every, fn: max, createEmpty: false)
		|> difference(nonNegative: true)

produced = increase(field: "Payload_NumberOfManufacturedProducts_Value")
	|> map(fn: (r) => ({_time: r._time, produced: r._value}))
scrapped = increase(field: "Payload_NumberOfDiscardedProducts_Value")
	|> map(fn: (r) => ({_time: r._time, scrapped: r._value}))
faulty = base(field: "Payload_FaultyTime_Value")
	|> aggregateWindow(every: every, fn: sum, createEmpty: false)
	|> map(fn: (r) => ({_time: r._time, faultyMs: r._value}))

windowMs = float(v: int(v: every)) / 1000000.0

join(tables: {p: produced, s: scrapped}, on: ["_time"])
	|> join(tables: {f: faulty}, on: ["_time"])
	|> map(fn: (r) => {
		runTimeMs   = windowMs - r.faultyMs
		totalPieces = r.produced + r.scrapped
		availability = if windowMs > 0.0 then runTimeMs / windowMs else 0.0
		performance  = if runTimeMs > 0.0 and totalPieces > 0.0 then
						   idealCycleTimeMs * totalPieces / runTimeMs
					   else 0.0
		quality      = if totalPieces > 0.0 then r.produced / totalPieces else 0.0

		return {_time: r._time, _field: "OEE", _value: availability * performance * quality * 100.0}
	})
```

> Using `difference(nonNegative: true)` on the counters correctly handles the
> counter resetting when the simulation restarts.

## Interpreting the Result

A common industry benchmark:

| OEE | Rating |
|---|---|
| 100 % | Perfect production (theoretical) |
| 85 % | World class for discrete manufacturers |
| 60 % | Typical for discrete manufacturers |
| 40 % | Low, but not unusual for plants without OEE tracking |

Because each factor is multiplied, a single weak factor drags the whole score
down — 90 % × 90 % × 90 % is only 73 %. That is the point of OEE: it exposes the
combined effect of downtime, slow running, and scrap in one number, and the
three factors tell you *which* of the three to attack.

In this reference stack you can go further and close the loop: point
[UA Cloud Action](./tutorial-command-and-control.md#automated-feedback-loop-with-ua-cloud-action)
at an OEE-derived signal so it invokes an OPC UA method automatically when a
factor degrades. Out of the box it watches a pressure threshold instead, so this
requires reconfiguring its `INFLUX_FIELD` / `INFLUX_THRESHOLD` and the target
method.

## Adapting This to Real Machines

The calculation is generic; only the inputs are simulation-specific. To apply it
to your own equipment:

1. **Map the four inputs** to variables your machines publish: good count, scrap
   count, fault/downtime duration, and the designed cycle time.
2. **Set `idealCycleTimeMs`** per station — it is a property of the machine, not
   something you can measure from the data. (Munich = `6000`; the Manufacturing
   Ontologies Seattle line uses `10000`.)
3. **Check the counter semantics of *your* machine.** In this simulation the piece
   counters are cumulative (`max - min`) while `FaultyTime` spikes once per fault
   with that fault's duration (`sum`) — both verified in the station source. A real
   PLC may instead expose downtime as a cumulative counter, in which case switch it
   to the `max - min` pattern, or derive downtime from a status/state signal.
4. **Filter by station** using whatever identifies your assets. If you onboarded
   them through
   [UA Edge Translator](./README.md#simulated-modbus-tcp-device-non-opc-ua), each
   asset has its own OPC UA namespace, so you can filter on that instead of the
   metadata lookup.
5. **Define the window as a shift**, not an arbitrary range — availability is
   measured against planned production time.

---

[< Back to the main README](./README.md)
