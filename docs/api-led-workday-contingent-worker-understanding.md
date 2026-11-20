# Workday Contingent Worker to P38 — API-led understanding

## Purpose

This document explains the target MuleSoft design for the Workday Contingent Worker integration. It translates the existing TIBCO file, JMS, publication, mapping, and file-output chain into API-led connectivity.

The detailed TIBCO behaviour and evidence are documented in [workday-contingent-worker-to-p38-understanding.md](workday-contingent-worker-to-p38-understanding.md). This document describes the target-state responsibilities and contracts.

## Target architecture

```text
Workday XML file
  │
  ▼
Workday Contingent Worker System API
  - file listener and file lifecycle
  - normal / priority VM file-sequencer queues
  │  POST raw XML, original filename and tracking ID
  ▼
Contingent Worker Process API
  - schema validation and event mapping
  - location enrichment orchestration
  - canonical CMM SOAP envelope
  │  POST canonical envelope, original filename and tracking ID
  ▼
P38 Contingent Worker System API
  - HUB location-mapping access
  - CHRIS/P38 file creation
  ▼
P38 / CHRIS inbound directory
```

There are three deployable Mule applications: two System APIs and one Process API.

## API responsibilities

| API | Owns | Does not own |
| --- | --- | --- |
| Workday System API | Workday file folders, filename recognition, working/processed/error movements, VM FileSequencer queues, raw-file hand-off | Canonical business mapping and P38-specific database/file access |
| Contingent Worker Process API | Validation, Workday-to-canonical transformation, tracking propagation, orchestration of enrichment and output | Direct filesystem, broker, Oracle/HUB, or CHRIS connectivity |
| P38 System API | Oracle/HUB location lookup and P38/CHRIS output directory | Workday parsing or business transformation rules |

## Legacy path to target path

| TIBCO step | Target Mule implementation |
| --- | --- |
| `FileSequencer.process` | Workday System API file listener moves a `PQ_CW*.xml` file into working storage. |
| `Workday.FileSequencer.AMM-SRC.ProcessFile.Publication` | Workday System API publishes a file descriptor to its internal normal VM file-sequencer queue. |
| `JmsFilePoller.process` / `JmsPriorityFilePoller.process` | Separate Workday System API VM queue consumers read the working file and call the Process API. The legacy flow names are retained for traceability. |
| `Workday.ContingentWorker.AMM-SRC.Create.Publication` | Process API entry point: `POST /api/v1/contingent-workers`. |
| `ContingentWorker/processFile.process` | Process API validates against `Workday_ContingentWorker.xsd`, flattens worker events, requests location enrichment, and constructs canonical events. |
| `Workday.ContingentWorker.CMM-SRC.Create.Publication` | Process API trace subflow after canonical mapping. |
| HRSystem Core EventHandler and `Workday.ContingentWorker.CMM.Create.Publication` | Process API core-publication trace subflow. |
| P38 MRP Core EventHandler and `P38.ContingentWorker.CMM-DST.Create.Publication` | Process API calls the P38 System API output endpoint. |
| P38 SAP SEAL EventHandler / Create File | P38 System API extracts the SOAP body and writes the XML file. |

The legacy names remain in Mule flow/subflow names so operational support can relate a Mule log or error to the former TIBCO process path.

## Contracts and metadata

### Workday System API → Process API

`POST /api/v1/contingent-workers`

- Body: raw Workday `Contingent_Worker_Events` XML.
- `x-original-file-name`: the source Workday filename.
- `x-adapter-tracking-id`: the end-to-end tracking/correlation ID.

### Process API → P38 System API: location lookup

`POST /api/v1/location-mappings`

- Body: `locationId`, `companyCode`, and `costCenterOdg`.
- Response: `personnelArea`, `personnelSubarea`, and `cityKey`.
- Lookup selection follows the TIBCO rule: a numeric ODG value uses `odg_code`; otherwise it uses `odg_name`.

### Process API → P38 System API: output

`POST /api/v1/contingent-worker-files`

- Body: canonical CMM SOAP envelope containing `Contingent_Worker_Events`.
- Headers: `x-original-file-name` and `x-adapter-tracking-id`.
- Result: the P38 System API writes the canonical XML to the configured CHRIS/P38 directory.

## Data transformation

The Process API keeps the source worker/event structure while generating the canonical `Contingent_Worker_Events` document. It preserves contact, custom IDs, phone, worker, additional, biographic, and position information. It enriches contact data with:

- personnel area;
- personnel subarea; and
- working city (`ZCITY_KEY`).

The canonical document is wrapped in a SOAP envelope. The envelope contains the canonical header and tracking details; the filename is retained in both request metadata and the body envelope for output selection.

## Processing and recovery

The Workday System API implements the file lifecycle:

```text
inbound → working → processed  (successful Process API call)
                  └→ error     (processing failure)
```

Processed and error files receive a timestamp suffix. The normal and priority queue flows are separate. The supplied TIBCO configuration maps `PQ_CW` to `ContingentWorker`; priority selection remains available for source configurations that mark a file type as priority.

The P38 System API deliberately retains the legacy output behaviour: it writes UTF-8 XML and overwrites an existing output file with the same original filename. Confirm whether this behaviour is acceptable before production deployment, since a replay can replace a prior output file.

## Deployment and configuration

Deploy in this order:

1. `p38-contingent-worker-system-api`
2. `contingent-worker-process-api`
3. `workday-contingent-worker-system-api`

Configure environment-specific values through secure properties:

- Workday inbound, working, processed, and error folders;
- local VM queue names;
- Process and P38 System API endpoint addresses;
- Oracle JDBC driver, connection details, and `LOCATION_MAPPING` table; and
- P38/CHRIS output directory.

`hub.disabled=true` allows the P38 System API to run without executing location lookups. It is appropriate only for interface testing: production mapping requires the HUB/Oracle configuration and approved JDBC driver.

## Open production decisions

- Decide whether production should retain VM queues or use an enterprise broker with an equivalent durable/retry policy.
- Confirm whether normal and priority queues need different concurrency/pacing policies.
- Provide representative valid, malformed, and empty Workday XML files with expected P38 output.
- Confirm the required response/error contract for callers and operational monitoring.
- Confirm whether filename overwrite must remain, or whether an idempotency/versioning strategy is required.
