# API-led Mule migration — Workday Contingent Worker to P38

This migration separates the requested TIBCO path into deployable Mule 4 APIs:

| Layer | Mule application | Responsibility |
| --- | --- | --- |
| System API | `workday-contingent-worker-system-api` | File lifecycle and FileSequencer/JMS polling; invokes the Process API with the raw Workday XML. |
| Process API | `contingent-worker-process-api` | Validates, enriches and maps Workday Contingent Worker data into the canonical CMM SOAP envelope. |
| System API | `p38-contingent-worker-system-api` | Owns HUB location lookups and creates the P38/CHRIS file. |

The legacy publication names are retained as Mule flow names and trace points: `FileSequencer.AMM-SRC.ProcessFile`, `AMM-SRC`, `CMM-SRC`, `CMM`, and `CMM-DST`. HTTP is the API-to-API contract; JMS remains only inside the Workday System API for the two TIBCO file-poller queues.

The two Core event-handler path names are retained as Process API subflows, while the P38 SAP SEAL event handler is the P38 System API file endpoint. This keeps the migration traceable without retaining cross-domain JMS as the API contract.

## Requested-path coverage

| TIBCO path/event | Mule owner | Mule implementation |
| --- | --- | --- |
| `FileSequencer.process` → `Workday.FileSequencer.AMM-SRC.ProcessFile.Publication` | Workday System API | File listener, move to working storage, persistent file-sequencer queue message. |
| `JmsFilePoller.process` / `JmsPriorityFilePoller.process` | Workday System API | Separate queue consumers; both preserve filename/tracking headers and invoke the Process API. |
| `Workday.ContingentWorker.AMM-SRC.Create.Publication` → `ContingentWorker/processFile.process` | Process API | `POST /api/v1/contingent-workers`, XSD validation, flattening, lookup and canonical transformation. |
| `Workday.ContingentWorker.CMM-SRC.Create.Publication` → HRSystem Core EventHandler | Process API | CMM-SRC trace subflow followed by the retained HRSystem Core handler subflow. |
| `Workday.ContingentWorker.CMM.Create.Publication` → P38 MRP Core EventHandler | Process API | Retained P38 MRP Core handler subflow. |
| `P38.ContingentWorker.CMM-DST.Create.Publication` → SAP SEAL Create File | P38 System API | HTTP System API request, extract canonical SOAP body, UTF-8 overwrite file creation. |

## Deployment order

1. Deploy `p38-contingent-worker-system-api`.
2. Deploy `contingent-worker-process-api`, configured with the P38 System API URL.
3. Deploy `workday-contingent-worker-system-api`, configured with the Process API URL and file/broker settings.

All values are placeholders. Supply secure broker, database and endpoint configuration per environment. The Oracle JDBC driver is intentionally not bundled; use the approved enterprise repository/runtime configuration.

The TIBCO-to-Mule analysis and behaviour rationale are in [the source-trace document](../docs/workday-contingent-worker-to-p38-understanding.md).
