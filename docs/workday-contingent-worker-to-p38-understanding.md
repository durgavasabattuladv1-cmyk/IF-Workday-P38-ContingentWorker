# Workday Contingent Worker to P38 — TIBCO process understanding

## Scope

This document describes the executable integration represented by the supplied paths: Workday file intake, Contingent Worker conversion, canonical event publication, P38 consumption, and file delivery to CHRIS/SAP.

The two source archives were expanded under `extracted/` for this analysis. Configuration values such as physical queues, topics, directories, intervals, and system identifiers are deployment variables (`%%...%%` / `GlobalVariables`); the archives define their logical keys, not their runtime values.

## End-to-end flow

```text
Workday drop directory
  │
  ├─ FileSequencer.process — classify, order, move to working, queue
  │   ├─ normal JMS queue  → JmsFilePoller.process
  │   └─ priority JMS queue → JmsPriorityFilePoller.process
  │
  └─ processFiles.process — read each working file and dynamically call
       FromWorkday/<Object>/sendFile.process
       └─ ContingentWorker/sendFile.process
          └─ AMM-SRC topic (raw Workday XML + originalFileName)
             └─ ContingentWorker/processFile.process
                └─ CMM-SRC topic (canonical Contingent_Worker_Events)
                   └─ HRSystem Core EventHandler
                      └─ CMM publication topic
                         └─ P38 MRP Core EventHandler
                            └─ CMM-DST topic
                               └─ P38 SAP SEAL EventHandler
                                  └─ create/overwrite and write XML file
```

The publication names in the request are the logical event naming convention. The concrete destinations are configurable keys named `TopicAMMSRC`, `TopicCMMSRC`, `TopicCMMPub`, and `TopicCMMDST`, all under the Contingent Worker event configuration.

## 1. File discovery, classification, and dispatch

### `.../_GenericFilePoller/FileSequencer.process`

This timer-started process runs at `SEAL/HRSystem/Workday/FilesCheckIntervalInMillis` and lists files matching:

`<FilePollerDirectory>/<FileMask>`

For every file, it loads `workdayFileConfiguration.sharedvariable`, which references `workdayFileConfigValues.xml`. Classification uses the *first* configuration entry whose `prefix` matches the beginning of the filename. The match determines:

- `object` — target type used in dynamic process resolution;
- optional `rank` — order within a batch; and
- optional `priority` — select the priority JMS queue.

The Workday configuration maps `PQ_CW` files to object `ContingentWorker`. This is the routing rule that leads to the flow in this document.

Recognized files are moved from the polling directory into `WorkingFIlesDIrectory` before they are queued. Files that do not match any configured prefix are moved to `<original location>/error/<filename>` and are reported through generic tracking/error handling.

When ranked files are present, they are sorted numerically and queued as a sequence. Unranked recognized files are separately queued one at a time. The implementation can therefore start both branches when a batch contains both kinds of file; migration design should preserve this concurrency/ordering behaviour where it matters.

### `.../_GenericFilePoller/queueFiles.process`

`queueFiles` constructs a `FileSequence` containing the working filename and object name, adds `WorkdayFileName` as supplemental tracking metadata, and sends a persistent JMS message to one of these configured destinations:

| Condition | Destination key | Consumer |
| --- | --- | --- |
| Any file in the sequence has `priority=true` | `Queues/FileSequencer/processNewPriorityFile` | `JmsPriorityFilePoller.process` |
| Otherwise | `Queues/FileSequencer/processNewFile` | `JmsFilePoller.process` |

The file is already in the working directory before the JMS message is produced. This is a critical recovery boundary: a failure during queueing can leave a file in `working`; the process logs that it must be manually moved back to the polling directory for reprocessing.

### `JmsFilePoller.process` and `JmsPriorityFilePoller.process`

Both consumers initialize tracking, deserialize the SOAP-wrapped `FileSequence`, send supplemental tracking information, call `processFiles.process`, and confirm the JMS message. Errors are reported to generic error handling and then acknowledged, rather than being left unconfirmed for broker redelivery.

The only functional difference found is pacing:

- normal queue sets `sleepAfterFile=true`;
- priority queue sets `sleepAfterFile=false`.

Both definitions specify `maxSessions=1`; the source comment saying that the priority handler has a high maximum number of jobs is not reflected in these exported process definitions.

### `.../_GenericFilePoller/processFiles.process`

This is the actual dynamic override point. For each `FileInfo`, it reads the text file from the working directory and invokes:

`/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/<Object>/sendFile.process`

For `PQ_CW...` files, `<Object>` is `ContingentWorker`, so the resolved process is `ContingentWorker/sendFile.process`—not `processFile.process` directly. A multi-file sequence gets a per-file tracking ID suffix `:<index>`; otherwise the original adapter tracking ID is retained.

After successful processing, the file is moved from `working` to `processed` and a timestamp is appended. On a file-level failure, it is reported to tracking and moved from `working` to `error`, also timestamped. Normal sequences sleep for `sleepBetweenFilesInSeqInMillis` between files; priority sequences do not.

## 2. Workday-specific Contingent Worker processing

### `.../FromWorkday/ContingentWorker/sendFile.process`

This process publishes the raw file text to the configured AMM-SRC topic (`TopicAMMSRC`) using a persistent JMS message. It includes:

- `ae_tracking` = the process-file subtracking ID;
- body = the raw Workday XML text; and
- dynamic property `originalFileName` = the source filename.

It also publishes supplemental tracking information with `WorkdayFileName`. This is the source of the AMM-SRC Create publication in the supplied path sequence.

### `.../FromWorkday/ContingentWorker/processFile.process`

This process is a durable JMS topic subscriber on the AMM-SRC topic (`HRSystemSeal/notifyContingentWorkerChange`). It initializes tracking, parses the raw XML as `Contingent_Worker_Events` with schema validation, and calls `mapper.process`.

The mapper iterates workers/events and constructs the canonical worker event form. It enriches data by:

- obtaining personnel area/subarea using location, company code, and cost-centre ODG code;
- deriving `ZCityKey` from location; and
- copying contact, custom-ID, phone, worker, additional, biographic, and position data while mapping core fields such as names, email, dates, worker type, employee group/subgroup, action, language, and event metadata.

The resulting XML is published as a SOAP envelope to CMM-SRC (`TopicCMMSRC`) with the canonical header/tracking information. The `originalFileName` is placed on the SOAP body as an attribute and must be preserved by downstream implementations.

Malformed/non-empty files receive the configured parse error code; empty files receive a separate configured error code. Both go through generic error handling and are confirmed, so these exceptions do not drive automatic JMS redelivery.

## 3. Core publication hops

### HRSystem Core: `.../notifyCreateWorkdayContingentWorkerEvent/EventHandler.process`

This durable subscriber consumes CMM-SRC (`TopicCMMSRC`) with subscription `HRSystemCore/notifyCreateWorkdayContingentWorkerEvent`. The handler initializes tracking and discards messages when the shared initialization process marks them as discarded. Otherwise `_Main.process` republishes the SOAP body unchanged to CMM publication (`TopicCMMPub`), rebuilding the canonical header from the current core configuration and retained tracking context.

### P38 MRP Core: `.../doAsyncCreateWorkdayContingentWorkerEvent/EventHandler.process`

This durable subscriber consumes the CMM publication (`TopicCMMPub`) with subscription `SAPHRCore/doAsyncCreateWorkdayContingentWorkerEvent`. Its `_Main.process` republishes the body unchanged to the CMM destination (`TopicCMMDST`). It retains the tracking initialization information and rebuilds the canonical header for the P38 core context.

In both core handlers, exceptions are sent to generic error handling and the JMS event is then confirmed. Operational monitoring of the tracking/error channel is therefore required to detect failed deliveries.

## 4. P38 SAP file delivery

### `.../ToSap/ContingentWorker/doAsyncCreateWorkdayContingentWorkerEvent/EventHandler.process`

The P38 SAP SEAL handler is a durable subscriber on CMM-DST (`TopicCMMDST`). It initializes tracking and, unless the message is discarded, passes the full SOAP envelope to `_Main.process`. Errors are logged, reported through generic error handling, and confirmed.

### `.../ToSap/ContingentWorker/.../_Main.process`

This process implements the requested “Create File” step:

1. It selects a filename using the SOAP body `originalFileName` attribute.
2. If present, the output is `<SEAL/Sap/ContingentWorker/fileLocation>/<originalFileName>`.
3. If absent, it uses `DEFAULT_FILE_NAME_<timestamp>.xml` in the same location.
4. It creates the file with `overwrite=true`.
5. It serializes the `Contingent_Worker_Events` SOAP body as UTF-8, formatted XML and writes it to that file.

The log labels this target as “CHRIS”. Because create uses `overwrite=true`, repeated events with the same original filename can replace the existing output. A Mule migration should explicitly choose whether this is the required business behaviour or whether idempotency/versioned filenames are safer.

## Reliability and migration-critical behaviours

| Area | Existing behaviour to retain or intentionally change |
| --- | --- |
| Routing | Prefix-based classification; `PQ_CW` resolves dynamically to `ContingentWorker/sendFile.process`. |
| Ordering | Ranked files are sorted ascending before queueing; unranked files use a separate one-file route. |
| File lifecycle | `polling → working → processed` on success; `working → error` on processing failure; outputs get timestamps only in processed/error folders. |
| Priority | `priority=true` selects a separate JMS queue and removes inter-file sleeping. |
| Tracking | Preserve adapter tracking ID, per-file subtracking suffixes, `WorkdayFileName`, and `originalFileName`. |
| Payload | Keep the SOAP canonical header and the body-level `originalFileName` attribute across CMM-SRC, CMM, and CMM-DST hops. |
| Acknowledgement | Consumers confirm JMS even after generic error handling; broker redelivery is not the failure-recovery mechanism. |
| Output | Write UTF-8 `Contingent_Worker_Events` XML into the P38 configured destination, with current overwrite semantics. |

## Source evidence

- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/_GenericFilePoller/FileSequencer.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/_GenericFilePoller/queueFiles.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/_GenericFilePoller/JmsFilePoller.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/_GenericFilePoller/JmsPriorityFilePoller.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/_GenericFilePoller/processFiles.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/SharedResources/workdayFileConfigValues.xml`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/ContingentWorker/sendFile.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/SEAL/HRSystem/InterfaceImpl/_AllInstances/FromWorkday/ContingentWorker/processFile.process`
- `extracted/workday/HRSystemSealAndCores-Workday_root/CORE/HRSystemCore/EventHandlerImpl/HRSystemEvents/ContingentWorker/notifyCreateWorkdayContingentWorkerEvent/{EventHandler,_Main}.process`
- `extracted/p38/SAPHRSealAndCores-P38_root/CORE/MrpSystemCore/EventHandlerImpl/HRSystemEvents/ContingentWorker/doAsyncCreateWorkdayContingentWorkerEvent/{EventHandler,_Main}.process`
- `extracted/p38/SAPHRSealAndCores-P38_root/SEAL/Sap/InterfaceImpl/_AllInstances/ToSap/ContingentWorker/doAsyncCreateWorkdayContingentWorkerEvent/{EventHandler,_Main}.process`
