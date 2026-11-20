# API-led connectivity migration — Workday Contingent Worker to P38

This directory contains Mule 4 System and Process APIs. Flow names follow the supplied TIBCO event-handler names.

```text
Workday file directory
  → System API file listener
  → HTTP request: POST http://localhost:8081/contingent-workers
  → Process API HTTP listener
  → P38 output directory
```

`contingent-worker-process-api` exposes `POST /contingent-workers` on port `8081` and contains the `ContingentWorker/processFile.process` mapping subflow. `p38-contingent-worker-system-api` exposes the same route on port `8082`, invokes the Process API with an HTTP request, and writes the returned canonical payload to the P38 output directory when it is started through its file listener.

No JMS broker is required for this API-led implementation. Configure the HTTP, directory, and deployment properties for local use. The TIBCO export does not contain resolved deployment values, so directory defaults remain placeholders.

This local interface path deliberately does not perform the prior HUB/Oracle enrichments; use it to validate the HTTP contract and mapping before restoring any environment-specific enrichment service.

For an interface-only local test, start `contingent-worker-process-api` and send a valid Workday XML document:

```bash
curl --request POST http://localhost:8081/contingent-workers \
  --header 'Content-Type: application/xml' \
  --header 'x-original-file-name: PQ_CW_test.xml' \
  --data-binary @sample-workday-contingent-worker.xml
```

To test the system API to Process API hop, start both applications and post the same request to `http://localhost:8082/contingent-workers`.

The original single-application migration remains under `../mulesoft-workday-p38-contingent-worker` as a traceable reference implementation.
