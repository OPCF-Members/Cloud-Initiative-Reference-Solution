# Importing an OPC UA Information Model into InfluxDB (UA Cloud Library)

You can pre-load the **full set of variables** an OPC UA server *could* expose —
not just the ones currently being published — by importing its **Information
Model** from the OPC Foundation [UA Cloud Library](https://uacloudlibrary.opcfoundation.org)
into InfluxDB.

Each model variable is written as a placeholder point (field `status="[Future]"`)
into a dedicated **`opcua_model`** measurement in the `mqtt` bucket, so you can see
every *potential* node alongside the live `opcua_pubsub` values.

> A separate measurement is used (rather than mixing into `opcua_pubsub`) because
> InfluxDB fields are single-typed — the model's placeholder is a string, while
> live telemetry values are numeric.

The importer is provided as an on-demand Kubernetes Job in
[`import-opcua-model.yaml`](./import-opcua-model.yaml). It uses a small
standard-library Python script that downloads the model's NodeSet2 XML from the
Cloud Library REST API, extracts every `UAVariable`, and writes them to InfluxDB
using the token from the existing `influxdb-auth` Secret.

Steps:

1. **Register** (free) at the UA Cloud Library and note the **model id** of the
   model you want (visible in its Explorer URL — e.g. the `Station` nodeset is
   `1627266626`).
2. **Run the import Job**, supplying your Cloud Library credentials and the model
   id (substituted at apply time):

   ```bash
   export UACLOUDLIB_USERNAME="myUser"
   export UACLOUDLIB_PASSWORD="myPass"
   export UACLOUDLIB_MODEL_ID="1627266626"
   kubectl delete job import-opcua-model -n cloud --ignore-not-found
   envsubst < import-opcua-model.yaml | kubectl apply -f -
   kubectl logs -f job/import-opcua-model -n cloud
   ```

   The log prints how many variables were imported.
3. **Query the imported model** in the InfluxDB Data Explorer or Grafana:

   ```flux
   from(bucket: "mqtt")
     |> range(start: -1h)
     |> filter(fn: (r) => r._measurement == "opcua_model")
     |> filter(fn: (r) => r._field == "displayName")
     |> keep(columns: ["_value", "nodeId", "dataType", "namespaceUri", "model"])
   ```

> The Cloud Library endpoint (`uacloudlibrary.opcfoundation.org`) must be
> reachable from the cluster for the import Job to run. The Job auto-cleans up one
> hour after completion (`ttlSecondsAfterFinished`).

---

[< Back to the main README](./README.md)
