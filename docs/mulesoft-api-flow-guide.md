# MuleSoft API flow guide — Workday Contingent Worker to P38

## 1. What this integration does

This integration receives a Workday Contingent Worker XML file and delivers a transformed canonical XML file to the P38/CHRIS inbound directory.

The migration has three Mule 4 applications:

1. `workday-contingent-worker-system-api` receives and manages Workday files.
2. `contingent-worker-process-api` validates, enriches, and transforms the worker data.
3. `p38-contingent-worker-sys` performs location lookups and writes the output file.

For local testing, the Workday System API uses Mule VM queues. These queues replace the TIBCO/JMS queue behavior inside the application only; communication between the three APIs uses HTTP.

```text
PQ_CW*.xml
  → Workday System API
  → Process API
  → P38 System API
  → canonical Contingent_Worker_Events XML file
```

## 2. TIBCO-to-Mule mapping

| TIBCO process/publication | Mule application | Mule flow or action |
| --- | --- | --- |
| `.../_GenericFilePoller/FileSequencer.process` | Workday System API | File listener flow named `...FileSequencer.process` |
| `Workday.FileSequencer.AMM-SRC.ProcessFile.Publication` | Workday System API | VM publish subflow with the same publication name |
| `JmsFilePoller.process` | Workday System API | VM listener flow with the retained TIBCO flow name |
| `JmsPriorityFilePoller.process` | Workday System API | Separate VM listener flow with the retained TIBCO flow name |
| `Workday.ContingentWorker.AMM-SRC.Create.Publication` | Process API | HTTP listener flow at `POST /api/v1/contingent-workers` |
| `.../ContingentWorker/processFile.process` | Process API | Validation, XML parsing, flattening, enrichment, and canonical mapping subflow |
| `Workday.ContingentWorker.CMM-SRC.Create.Publication` | Process API | Trace/logging subflow after canonical mapping |
| HRSystem Core EventHandler / `Workday.ContingentWorker.CMM.Create.Publication` | Process API | Trace/logging subflow that represents the CMM hop |
| P38 MRP Core EventHandler / `P38.ContingentWorker.CMM-DST.Create.Publication` | Process API | HTTP call to the P38 System API file endpoint |
| P38 SAP SEAL EventHandler / Create File | P38 System API | File-output flow at `POST /api/v1/contingent-worker-files` |

The long flow names are intentionally retained so a TIBCO support team can identify the comparable Mule implementation.

## 3. Workday Contingent Worker System API

Source: `api-led-workday-p38/workday-contingent-worker-system-api`

### 3.1 Purpose

This API owns the file lifecycle. It watches for Workday files, moves them to a safe working location, sends a lightweight file descriptor through a VM queue, and calls the Process API with the raw XML.

### 3.2 FileSequencer flow

Flow: `HRSystemSealAndCores-Workday_root\SEAL\HRSystem\InterfaceImpl\_AllInstances\FromWorkday\_GenericFilePoller\FileSequencer.process`

It runs as follows:

1. The File listener polls `workday.inbound.directory` at `workday.polling.frequency`.
2. It selects only files matching `workday.contingent-worker.filename-pattern`, normally `PQ_CW*.xml`.
3. It stores the source filename in `vars.originalFileName`.
4. It creates `vars.adapterTrackingId` from Mule's `correlationId`.
5. It moves the file from inbound to `workday.working.directory`.
6. It invokes the FileSequencer publication subflow.

This move occurs before queueing, matching the TIBCO recovery boundary. A file that is not in `working` is not yet ready for downstream processing.

### 3.3 FileSequencer VM publication

Subflow: `Roche.Pharma.CrossDomain.OP.Workday.FileSequencer.AMM-SRC.ProcessFile.Publication`

The subflow publishes this Java payload to the normal VM queue:

```json
{
  "originalFileName": "PQ_CW_example.xml",
  "ae_tracking": "<Mule correlation ID>",
  "fileType": "ContingentWorker"
}
```

It uses `vm.queue.file-sequencer`. This retains the metadata that was previously carried as JMS properties. The payload does not contain the full Workday file; the consumer reads the file from the working directory.

### 3.4 Normal and priority poller flows

Flows:

- `...JmsFilePoller.process`
- `...JmsPriorityFilePoller.process`

Despite their retained legacy names, both are Mule VM listeners.

- The normal flow listens on `vm.queue.file-sequencer`.
- The priority flow listens on `vm.queue.priority-file-sequencer`.

Both call `dispatch-workday-file`. The current FileSequencer publication sends Contingent Worker files to the normal queue; the priority flow is retained for compatibility with future file-priority routing.

### 3.5 Dispatch and file-state handling

Subflow: `dispatch-workday-file`

1. Reads `originalFileName` and `ae_tracking` from the VM message payload.
2. Reads the XML content from the working folder.
3. Calls the Process API using `POST ${process.api.path}`.
4. Sends these HTTP headers:
   - `x-original-file-name`
   - `x-adapter-tracking-id`
5. If the call succeeds, moves the working file to `workday.processed.directory` and appends a timestamp.
6. If an error occurs, logs the error and moves the file to `workday.error.directory` with a timestamp.

The error handler uses `on-error-continue`. Therefore, a handled failure moves the file to the error folder rather than causing the VM message to be retried indefinitely.

## 4. Contingent Worker Process API

Source: `api-led-workday-p38/contingent-worker-process-api`

### 4.1 Purpose

This API contains the business transformation. It has no direct file-system or database connector. It accepts the raw Workday XML, requests required location data from the P38 System API, and creates the canonical CMM SOAP message.

### 4.2 Main endpoint

Flow: `Roche.Pharma.CrossDomain.OP.Workday.ContingentWorker.AMM-SRC.Create.Publication`

| Item | Value |
| --- | --- |
| Method | `POST` |
| Path | `${http.listener.path}`, default `/api/v1/contingent-workers` |
| Body | Workday `Contingent_Worker_Events` XML |
| Required header | `x-original-file-name` |
| Optional header | `x-adapter-tracking-id` |

At the start of the flow, the API stores the two headers in variables:

- `vars.originalFileName`
- `vars.adapterTrackingId`

If either header is absent, it uses a safe default (`unknown.xml` or Mule's correlation ID).

### 4.3 Process-file subflow

Subflow: `.../FromWorkday/ContingentWorker/processFile.process`

The subflow performs these steps in order:

1. **XSD validation** — `xml-module:validate-schema` checks the payload against `schema/Workday_ContingentWorker.xsd`.
2. **XML parsing** — DataWeave converts the XML payload to a Java object structure.
3. **Non-empty check** — `validation:is-true` requires at least one `Worker`; an empty document fails with the message `Unable to parse file from Workday because it contains no Contingent Worker data`.
4. **Flattening** — `flatten-contingent-worker-events.dwl` converts each `Worker` and each nested `Contingent_Worker_Event` into a flat array. Every item retains its `employeeId` and event data.
5. **Per-event enrichment** — for every flattened event, the API calls the P38 System API location-mapping endpoint.
6. **Event enrichment** — returned personnel-area, personnel-subarea, and city values are added to `Contact_Data` when the lookup returns a value.
7. **Rebuild canonical XML** — `build-contingent-worker-events.dwl` groups the mapped events by `employeeId` and produces canonical `Contingent_Worker_Events` XML.
8. **Wrap the SOAP envelope** — `wrap-canonical-envelope.dwl` adds the canonical CMM header, tracking data, and `originalFileName` body attribute.

### 4.4 Location enrichment call

For each Workday event, the Process API calls:

```text
POST /api/v1/location-mappings
```

Request body:

```json
{
  "locationId": "<Position_Data.Location_ID>",
  "companyCode": "<Position_Data.Company_Code>",
  "costCenterOdg": "<Position_Data.Cost_Center_ODG_Code>"
}
```

Expected response:

```json
{
  "personnelArea": "...",
  "personnelSubarea": "...",
  "cityKey": "..."
}
```

### 4.5 Canonical publication trace points

After transformation, the Process API executes the CMM-SRC and CMM trace subflows. They currently log the logical event creation; they preserve the event boundaries and TIBCO names without introducing a second cross-application message broker.

The final core-handler subflow calls the P38 System API output endpoint. The Process API sends the SOAP envelope as the body and forwards the source filename and tracking ID as HTTP headers.

## 5. P38 Contingent Worker System API

Source: `api-led-workday-p38/p38-contingent-worker-sys`

### 5.1 Purpose

This API owns P38-specific connectivity:

- Oracle/HUB location lookup; and
- P38/CHRIS output-file creation.

### 5.2 Location-mapping endpoint

Flow: `get-contingent-worker-location-mapping`

| Item | Value |
| --- | --- |
| Method | `POST` |
| Path | `/api/v1/location-mappings` |
| Input | `locationId`, `companyCode`, `costCenterOdg` JSON |
| Output | `personnelArea`, `personnelSubarea`, `cityKey` JSON |

Behavior:

1. If `hub.disabled=true`, returns all three response values as `null`. This supports local interface testing without Oracle.
2. If enabled and `costCenterOdg` is numeric, queries `LOCATION_MAPPING.odg_code`.
3. If enabled and the ODG is non-numeric, queries `LOCATION_MAPPING.odg_name`.
4. It performs a second lookup by `LOCATION_ID` to obtain `ZCITY_KEY`.
5. It returns the lookup results to the Process API.

### 5.3 Create-file endpoint

Flow: `.../ToSap/ContingentWorker/doAsyncCreateWorkdayContingentWorkerEvent/EventHandler.process`

| Item | Value |
| --- | --- |
| Method | `POST` |
| Path | `/api/v1/contingent-worker-files` |
| Body | Canonical CMM SOAP envelope |
| Input header | `x-original-file-name` |
| Output | JSON confirmation with `status` and `fileName` |

The flow:

1. Uses `x-original-file-name` as the output name.
2. Uses `DEFAULT_FILE_NAME_<timestamp>.xml` when the header is absent.
3. Uses `extract-contingent-worker-body.dwl` to extract `Contingent_Worker_Events` from the SOAP body.
4. Writes formatted XML to `p38.contingent-worker.output.directory` with `mode="OVERWRITE"`.
5. Returns `{"status":"written","fileName":"..."}`.

`OVERWRITE` intentionally preserves the TIBCO behavior. Replaying the same filename replaces the prior file.

## 6. Important configuration

| API | Important properties |
| --- | --- |
| Workday System API | `workday.inbound.directory`, `workday.working.directory`, `workday.processed.directory`, `workday.error.directory`, `workday.contingent-worker.filename-pattern`, `vm.queue.*`, `process.api.*` |
| Process API | `http.listener.*`, `p38.system.api.*`, `canonical.*`, `workday.amm-src.publication` |
| P38 System API | `http.listener.*`, `hub.disabled`, `hub.db.*`, `hub.location.mapping.table`, `p38.contingent-worker.output.directory` |

Use environment-specific secure configuration for database credentials and real endpoint addresses. The checked-in properties contain local placeholders.

## 7. Local test sequence

1. Set `hub.disabled=true` in the P38 System API for a database-free test.
2. Start the P38 System API on its configured port, normally `8082`.
3. Start the Process API on its configured port, normally `8081`.
4. Start the Workday System API.
5. Place a valid `PQ_CW*.xml` file in the Workday inbound directory.
6. Confirm the file moves to `processed` and that canonical XML appears in the P38 output directory.
7. For a malformed or empty file, confirm it moves to the Workday error directory.

## 8. Operational points to remember

- VM queues are in-memory and local to the Workday System API. They are suitable for local testing; they do not provide cross-runtime broker durability.
- The Process API makes one location lookup HTTP call for each individual worker event.
- The Process API requires a valid Workday XSD document and at least one worker.
- The P38 output-file behavior overwrites an existing file with the same name.
- The original filename and tracking ID must be forwarded through the HTTP headers to preserve traceability.
