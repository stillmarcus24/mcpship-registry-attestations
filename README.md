# MCPShip Registry Attestations

Public, independently produced observations of the Official MCP Registry.

This repository exists to make MCPShip Registry census and attestation results
publicly inspectable, time-stamped and reproducible at the evidence level.

## Principles

- measurements are produced independently;
- no shared census implementation is used with comparison partners;
- MCPShip freezes and publishes its own result before inspecting the comparison result;
- incomplete or truncated walks are not presented as complete observations;
- disagreements are published rather than silently reconciled;
- Registry metadata is not treated as server liveness evidence.

## Published fields

Each official observation records at least:

- start time;
- finish time;
- raw rows;
- unique canonical names;
- pages;
- stop reason;
- digest;
- completeness decision.

Additional accounting fields may also be retained.

## First coordinated observation

The first coordinated independent observation is scheduled for:

`2026-09-25T16:05:00Z`

Equivalent local time for MCPShip:

`2026-09-25 18:05 Europe/Vienna`

The comparison partner is expected to run an independently implemented sweep
during the same coordinated window.

MCPShip's result will be frozen and published here before the partner result
is inspected.
