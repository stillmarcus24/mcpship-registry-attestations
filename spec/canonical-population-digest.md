# Canonical population digest and commit-reveal for coordinated windows

Proposed format for the joint MCP Registry attestation series agreed on
[stillos-notary-mcp#1](https://github.com/stillmarcus24/stillos-notary-mcp/issues/1).

This document specifies **formats only**. It contains no sweep implementation and
creates no dependency in either direction. Each side implements it independently;
the independence is the asset the series is built on.

## 1. Why this exists

Window 1 (2026-09-25 16:05 UTC) converged on all six counts — pages, raw rows,
unique canonical names, duplicates, active, deprecated — between two
independently written implementations. Two of the five agreed fields could not be
compared:

| Field | Comparable across implementations? | Why |
| --- | --- | --- |
| raw rows | yes | integer |
| unique canonical names | yes | integer |
| pages | yes | integer |
| stop reason | yes, by semantics | `natural_exhaustion` / `exhausted` are the same decision under different labels |
| digest | **no** | each side hashes its own artifact file layout |

A digest that structurally cannot agree is not evidence of divergence when it
differs, and it is not evidence of anything when it matches. This adds one field
that two implementations **can** compare, and leaves each side's own artifact
digest untouched alongside it.

It also closes a second gap. A 2m20s runner publishes before a 9m50s runner every
time, so "I published before reading yours" is not symmetrically checkable. In
window 1, MCPShip published blind and StillOS did not; StillOS recorded that in
the thread rather than letting the asymmetry stand unstated. Commit-reveal
removes the need for either side to be taken at its word.

## 2. Canonical population digest

Given the set of canonical server names observed in a window:

1. Deduplicate. Identity is the canonical `server.name` string, unchanged — no
   case folding, no normalization, no trimming.
2. Sort ascending by **Unicode code point** of the full name string. Not by
   locale, not case-insensitively. `/`, `.`, `-` are ordinary characters with no
   special precedence.
3. Emit each name followed by a single `\n` (U+000A). The last line is also
   `\n`-terminated. No BOM, no `\r`, no trailing blank line beyond that.
4. Encode UTF-8.
5. `canonical_population_sha256` = SHA-256 over exactly those bytes, lowercase hex.

Published as a field distinct from each implementation's own artifact digest:

```json
{
  "canonical_population_form": "unique canonical names, sorted by Unicode code point, one per line, LF-terminated, UTF-8",
  "canonical_population_bytes": 1156785,
  "canonical_population_sha256": "43782c49a3129acc84b8d6b28137dd4e532bdb10a52382d34a5c840d4354e494"
}
```

### 2.1 Test vectors

Computed, not asserted. Each is independently checkable with
`printf '...' | sha256sum`.

| Vector | Input | Canonical bytes | SHA-256 |
| --- | --- | --- | --- |
| empty population | `[]` | `` (0 bytes) | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| single name | `["a/b"]` | `a/b\n` | `241714c62f70695b332fd13a5feeb58b25bf8f3899a0e41d80d2dd6892d8f87a` |
| duplicate collapses | `["a/b","a/b"]` | `a/b\n` | `241714c62f70695b332fd13a5feeb58b25bf8f3899a0e41d80d2dd6892d8f87a` |
| uppercase before lowercase | `["b/y","A/x","a/z"]` | `A/x\na/z\nb/y\n` | `47653145ef51528968c234d1cba39057b00d96b483ecc43c5f88c35dbdef94a7` |
| non-ASCII by code point | `["zone.waggle/waggle","é/w","ac.snag/snag"]` | `ac.snag/snag\nzone.waggle/waggle\né/w\n` | `1e5775d9576802e87d4ef0fab62a8b12b0c88692443e024cde702971e79b2c0b` |
| `.` `/` `-` are ordinary | `["a.b/c","a/b.c","a-b/c"]` | `a-b/c\na.b/c\na/b.c\n` | `01cd5abf59eea955150ce963c41992a269f0cb915e75e4ff25c30832b38ea7e3` |

The empty vector matters: it is the SHA-256 of zero bytes, so an implementation
that silently produced an empty population would emit a *well-formed* digest.
Any consumer must gate on the count fields, never on the digest alone.

The duplicate vector matters for the opposite reason: the digest is over the
deduplicated set, so `duplicates` must stay a separately published integer. The
digest cannot carry that information.

### 2.2 Window 1, recomputed from the frozen artifact

```
window                        2026-09-25T16:05:05.148Z -> 16:14:55.074Z
unique canonical names        35,986
canonical_population_bytes    1,156,785
canonical_population_sha256   43782c49a3129acc84b8d6b28137dd4e532bdb10a52382d34a5c840d4354e494
first name in order           ac.inference.sh/mcp
last name in order            zone.waggle/waggle
```

Derived from StillOS's already-published frozen artifact, not from a re-sweep.
It is offered so the field has a value for a window that already happened; a
match here is a check on the *format*, since both sides' counts for that window
are already public and the reading order was not symmetric.

## 3. Commit-reveal

### 3.1 Canonical JSON

The committed record body is serialized with object keys sorted ascending by code
point **at every level**, no insignificant whitespace, arrays in document order.
Two implementations serializing the same logical record must produce identical
bytes, so key order cannot be left to insertion order.

### 3.2 Scheme

```
nonce                 32 random bytes, lowercase hex (64 chars), fresh per window
commitment_sha256     sha256( nonce_hex || canonical_json(record_body) )
record_sha256         sha256( canonical_json(record_body) )      // body only, no nonce
```

The nonce is required. Without it, a commitment over a record whose material
content is a handful of integers can be attacked by grinding candidate counts.

`record_sha256` is the body-only digest, matching the existing `recordSha256`
convention in this repository, and is computed before any self-referential hash
field is added.

### 3.3 Protocol

1. Both sides start independent sweeps at the agreed UTC start time.
2. The instant a side's own walk completes, it publishes **only** its
   `commitment_sha256`, the window start/finish timestamps, and the scheme
   string. No counts.
3. Once both commitments are public, each side publishes its record body and
   nonce.
4. Either party — or any third party — recomputes
   `sha256(nonce_hex || canonical_json(body))` and compares it to the published
   commitment.

Reading order stops carrying any weight, because a commitment published before
the other side's counts existed cannot be adjusted afterward.

### 3.4 Failure semantics

- A reveal that does not recompute to its commitment is **rejected**, not
  reconciled. Publish the rejection.
- A sweep that did not terminate on natural cursor exhaustion is not eligible to
  be a joint observation at all, and should not produce a commitment.
- Completeness is decided by the cursor walk, never by agreement between the two
  sides and never by a producer's own `truncated`/`complete` flag.

## 4. Execution provenance

Recommended fields, collected **before** the walk begins. Hashing after a
ten-minute run attests whatever is on disk at the end rather than the code that
produced the number.

```json
{
  "runner_sha256": "...",
  "census_module_sha256": "...",
  "third_party_dependencies_in_observation_path": [],
  "notifier_outside_observation_path": true,
  "node_version": "v...",
  "git_head": "...",
  "git_head_covers_execution": false
}
```

`git_head_covers_execution` exists because a repository HEAD proves nothing about
files that are not tracked in that repository. StillOS's executing files are
currently untracked, so StillOS publishes HEAD with this flag set to `false`
rather than implying coverage it does not have. The file digests are the real
provenance.

The dependency field is scoped to the **observation path** — the sweep runner and
the census module, up to and including sealing the record — and both require only
the language standard library there. It is deliberately not a claim about the
whole process: StillOS's runner also sends an operator alert once a commitment
exists, and that notifier's dependency chain does reach third-party packages. It
is therefore imported lazily, after the record and its digest are already written,
so it cannot sit underneath a number. An unscoped "no third-party dependencies"
claim would have been false, which is the reason the field carries its scope in
its name.

## 5. Out of scope

- Runtime liveness. `active` / `deprecated` are Registry metadata and are not
  treated as evidence that a server answers.
- Any shared implementation, vendored module, or common dependency between the
  two sides.
- Exit codes. Each implementation keeps its own.
