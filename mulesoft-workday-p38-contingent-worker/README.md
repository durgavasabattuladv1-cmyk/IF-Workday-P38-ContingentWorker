# Mule 4 migration — Workday Contingent Worker to P38

This is an Anypoint Studio importable Mule 4.6 application for the fully traced integration path from the uploaded TIBCO projects:

`FileSequencer → queueFiles → JmsFilePoller/JmsPriorityFilePoller → processFiles → ContingentWorker/sendFile → ContingentWorker/processFile → HRSystem Core → P38 MRP Core → P38 SAP SEAL file output`.

## Implemented TIBCO behaviour

- Watches `PQ_CW*.xml`, moves it to the working folder, sends file metadata through the normal FileSequencer JMS queue, publishes the raw file to AMM-SRC, then moves it to processed with a timestamp. A separate priority-queue consumer is included to mirror the TIBCO route; `PQ_CW` is not priority-configured in the supplied source.
- Preserves `originalFileName` and the tracking ID as JMS properties.
- Parses Workday Contingent Worker XML, rejects an empty worker collection, flattens worker/event pairs, and rebuilds the canonical `Contingent_Worker_Events` document.
- Includes the two TIBCO source XSDs and validates the incoming AMM-SRC XML against `Workday_ContingentWorker.xsd` before transforming it.
- Executes both TIBCO database enrichments when HUB is enabled:
  - personnel area/subarea lookup, choosing `odg_code` for numeric ODG and `odg_name` otherwise;
  - `ZCITY_KEY` lookup by location.
- Publishes to AMM-SRC, CMM-SRC, CMM, and CMM-DST as JMS topics, retaining message properties across the core hops. The CMM message is a SOAP envelope with the canonical tracking header and a body-level `originalFileName` attribute, as in TIBCO.
- Writes UTF-8 XML to the configured CHRIS/P38 directory. The source filename is used when available; otherwise a timestamped default name is generated. The Mule file write uses `OVERWRITE`, matching TIBCO `FileCreate(overwrite=true)`.
- Handles errors with logging and publication to the configured error topic. This uses `on-error-continue`, matching the examined TIBCO handlers, which report and then confirm rather than relying on broker redelivery.

## Import and configure

1. Import this directory in Anypoint Studio as an existing Maven project.
2. Configure values in `src/main/resources/application.properties` through your environment/secure-properties mechanism. Do not use the supplied placeholder database URL or credentials in a deployed environment.
3. Supply the Oracle JDBC driver through your organisation's approved Maven repository or Studio configuration. It is intentionally not included because it is not redistributable.
4. Configure ActiveMQ-compatible JMS access, the four production topic names, and filesystem permissions for inbound, working, processed, error, and CHRIS directories.
5. Set `hub.disabled=false` only after database access and the `LOCATION_MAPPING` table have been verified.

## TIBCO source gaps and migration limits

The uploaded archives contain **489** `.process` definitions: **351** in the Workday archive and **138** in the P38 archive. The application in this directory covers the Contingent Worker chain above, which was the path supplied in the original request. It deliberately does not claim to migrate the other **480+** definitions (other Workday object types, SAP BAPI/IDoc integrations, web services, state management, reports, Seeburger, MDMS, and shared tracking utilities).

Those processes are separate executable integrations, not branches of this Contingent Worker implementation. Generating “complete” Mule code for them without separately tracing their mappings, service contracts, deployment variables, and target behaviour would violate the requirement not to make assumptions.

The following production inputs are absent from the export and must be supplied before the target can be deployed or fully regression-tested:

- resolved TIBCO global variables for queue/topic names, directories, polling periods, file masks, system/component IDs, and error codes;
- JMS broker connection/TLS/authentication and durable subscription policy;
- Oracle JDBC driver, endpoint, credentials, schema ownership, table name, and actual `LOCATION_MAPPING` data contract;
- the expected payloads and transport contracts of the shared `TrackingErrorHandling` implementation;
- sample valid/invalid Workday Contingent Worker files and expected CHRIS output files; and
- confirmation that output overwrite by original filename is intended in the target environment.

## Files

- `src/main/mule/workday-p38-contingent-worker.xml` — Mule flows, global connectors, database access, and error handler.
- `src/main/resources/dwl/*.dwl` — DataWeave 2.0 event flattening, enrichment, and canonical XML construction.
- `src/main/resources/application.properties` — deployment property keys and non-secret placeholders.
