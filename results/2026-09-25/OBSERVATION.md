# MCPShip Registry attestation — 2026-09-25 16:05 UTC

This is MCPShip's independently produced observation for the coordinated
2026-09-25 Official MCP Registry attestation window.

The result was frozen and published before inspecting the comparison
partner's independently produced result.

## Timing

- coordinated window: `2026-09-25T16:00:00Z/2026-09-25T17:00:00Z`
- intended start: `2026-09-25T16:05:00Z`
- actual runner start: `2026-09-25T16:05:01.144Z`
- finish: `2026-09-25T16:07:21.322Z`

## Observation

- pages: `360`
- raw rows: `35986`
- unique canonical names: `35986`
- duplicate canonical names: `0`
- active: `35599`
- deprecated: `387`
- other statuses: `{}`
- rows with `isLatest=true`: `35986`
- stop reason: `natural_exhaustion`
- structurally complete: `true`
- runner error: `None`

`active` and `deprecated` are Registry metadata. They are not treated as
runtime-liveness evidence.

Structural completeness means this walk ended by natural cursor exhaustion
and satisfied MCPShip's accounting invariants. It is not inferred from exact
agreement with another independently timed live sweep.

## Integrity

Frozen JSON artifact SHA-256:

`51c094950225dc3cc969d1636dee5d65ff97b0aaec477b93247037228724b0fc`

Record-body SHA-256 stored by the runner:

`ba899fd1427b8c8be04cf91e79ba93830cbdc6fe6541700081a21be0e6dbd6b0`

The artifact SHA-256 above hashes the exact published JSON file. The
`recordSha256` field is the runner's hash of the record body before the
self-referential hash field was added.

## Execution provenance

MCPShip repository HEAD at execution/publication:

`83dd364b46875e1adfd4eb2ea31674945bf765ba`

Scheduled runner SHA-256:

`225d1f3fa170941596342d02ab9b43b6419d44ab6ac465b63f917d39886e29a7`

Executed Registry census module SHA-256:

`006c91b3cec8929472008cc6a90c03e44a987612958efe90a599fa13e5cee7f0`

Executed Registry census HTTP module SHA-256:

`fae8120c11b8f56c59a4f93102663624f15f522e2e76f8a8992f88715428b7c3`

## Independence order

1. MCPShip produced this observation independently.
2. MCPShip froze the result and its digest.
3. MCPShip published this artifact before inspecting the partner result.
4. Only after publication will the independently produced partner observation
   be inspected and compared.
