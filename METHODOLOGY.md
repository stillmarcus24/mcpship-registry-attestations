# Registry Attestation Methodology

## Source

Official MCP Registry list endpoint.

The census uses the Registry's latest-version query and follows pagination
until the cursor chain naturally terminates.

## Canonical population unit

The population unit is the canonical Registry server name.

No runtime or liveness filter is applied.

## Completeness

A run is eligible as an official complete observation only when:

- pagination ends through natural cursor exhaustion;
- the snapshot is marked complete;
- raw-row accounting is internally consistent;
- the latest-version invariant is satisfied;
- Registry status accounting is internally consistent;
- the observed population is non-empty.

A run ending because of:

- page error;
- malformed response;
- repeated cursor;
- local safety cap;
- any other non-natural termination

is not presented as a complete joint observation.

## Independence

MCPShip's implementation is independent of the comparison implementation.

For a coordinated comparison:

1. MCPShip runs its own sweep.
2. MCPShip freezes the result.
3. MCPShip publishes the result.
4. Only then is the comparison result inspected.
5. Agreement or disagreement is documented without altering either original result.

## Digest

Published result artifacts are accompanied by SHA-256 material so the exact
published bytes can be identified later.

## Interpretation

Exact population equality is not required for two independently complete
live sweeps to both be valid.

The Registry can change while independent walks are in progress.

Timing, completion state and any observed difference must therefore be
reported together.
