# Dashboards with Grafana

**Grafana** is deployed by `cloud.yaml` as an alternative to InfluxDB's
built-in dashboards, with richer visualization, templating, and alerting. It comes
**pre-provisioned** so no manual setup is needed beyond logging in.

What the manifest provisions automatically:

- **InfluxDB data source** (`grafana-datasources` ConfigMap) — points at
  `http://influxdb.cloud.svc.cluster.local:8086` using the **Flux** query
  language, org `iot`, and default bucket `mqtt`. The query token is injected from
  the `influxdb-auth` Secret via the `INFLUX_TOKEN` environment variable
  (interpolated into the provisioned data source at startup).
- **Dashboard provider** (`grafana-dashboard-provider` ConfigMap) — loads any
  dashboards found under `/var/lib/grafana/dashboards`.
- **Starter dashboard** (`grafana-dashboards` ConfigMap) — *OPC UA Telemetry
  Overview* (uid `opcua-overview`) with a time-series panel of `opcua_pubsub`
  values, an ingest-rate stat, an active-dataset-writers stat, and a `publisher`
  template variable for filtering.

Usage:

1. Browse to `http://<device-ip>:3000` and log in with your `IOT_USERNAME` /
   `IOT_PASSWORD` (Grafana admin credentials set via `GF_SECURITY_ADMIN_USER` /
   `GF_SECURITY_ADMIN_PASSWORD`).
2. Open **Dashboards → OPC UA Telemetry Overview** to see live data flowing from
   the `mqtt` bucket.
3. Use the **InfluxDB** data source in **Explore** or when adding new panels; write
   Flux queries exactly as in the [InfluxDB dashboard section](./tutorial-influxdb-queries.md).
4. Grafana settings and any dashboards you create are persisted on the Pi at
   `/grafana`. (The provisioned data source and starter dashboard are managed by
   the ConfigMaps and re-applied on restart.)

> **Note:** the starter dashboard's numeric time-series panel assumes numeric
> fields (e.g. `Payload_<NodeName>_Value`). Adjust the panel's Flux filter to
> match the exact `_field` names your assets publish.

---

[< Back to the main README](./README.md)
