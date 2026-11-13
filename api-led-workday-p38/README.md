# API-led connectivity migration — Workday Contingent Worker to P38

This directory replaces the single application with two separately deployable Mule 4 applications:

```text
Workday file directory
  → P38 Contingent Worker System API (AMM-SRC topic)
  → Contingent Worker Process API (validate, enrich, map)
  → P38 Contingent Worker System API (CMM-SRC → CMM → CMM-DST → CHRIS file)
```

The Process API is the reusable business orchestration layer. The System API owns the Workday filesystem, JMS, HUB/Oracle, and CHRIS filesystem connectivity. No client- or channel-specific API façade is required for this batch integration.

Deploy the System API first and then the Process API. Set hostnames, ports, TLS, credentials, and deployment properties for each environment. The TIBCO export does not contain resolved deployment values, so defaults remain placeholders.

The original single-application migration remains under `../mulesoft-workday-p38-contingent-worker` as a traceable reference implementation.
