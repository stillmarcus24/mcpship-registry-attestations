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

## 5. Publication and discovery

§3.3 says "publishes" without saying where, which makes the protocol unrunnable
by a machine and unauditable by a third party. This section fixes the location.

### 5.1 Participant registry

One file at the repository root, `PARTICIPANTS.json`. It is the only place a
consumer needs to start from.

```json
{
  "participants": [
    {
      "id": "mcpship",
      "operator": "Heaviside Solutions",
      "signing_keys_url": "https://.../keys",
      "artifact_base_url": null
    },
    {
      "id": "stillos",
      "operator": "StillOS Digital Holdings",
      "signing_keys_url": "https://stillosdigitalholdings.com/notary/keyring",
      "artifact_base_url": null
    }
  ]
}
```

`id` is lowercase ASCII `[a-z0-9-]+` and appears verbatim in every filename
below. `artifact_base_url` is optional: when non-null, the same files MUST also
be reachable under it at the identical relative paths, so a consumer who does
not want to depend on a single code host has a second route to byte-identical
files. When null, this repository is the only publication path, which is a
stated fact about that participant, not a defect.

### 5.2 Deterministic paths

```
results/<window_id>/<participant_id>-commitment.json     phase 1
results/<window_id>/<participant_id>-reveal.json         phase 2
results/<window_id>/<participant_id>-<phase>.sig.json    detached signature (§6)
```

`window_id` is the **agreed** UTC start, not the actual runner start:
`YYYY-MM-DDTHHMMZ` — e.g. `2026-09-26T1605Z`. The agreed start is used because
both sides know it before either runs, so both can compute the path without
coordination. Actual start and finish are fields inside the files.

The existing `results/2026-09-25/` directory predates this convention and is
grandfathered as-is; it is not renamed.

### 5.3 Machine-readable index

`windows.json` at the repository root, append-only, newest last:

```json
{
  "windows": [
    {
      "window_id": "2026-09-26T1605Z",
      "agreed_start_utc": "2026-09-26T16:05:00Z",
      "state": "revealed",
      "participants": {
        "stillos":  { "commitment_sha256": "...", "record_sha256": "...", "revealed": true },
        "mcpship":  { "commitment_sha256": "...", "record_sha256": "...", "revealed": true }
      }
    }
  ]
}
```

`state` is one of `committed` (at least one commitment, not all reveals in),
`revealed` (every participant's reveal present and recomputing), `rejected` (a
reveal did not recompute — §3.4), `incomplete` (a participant's walk did not
terminate on cursor exhaustion, so it produced no commitment).

`windows.json` is a **convenience index, never the authority**. Every value in
it is recomputable from the files in `results/`. A consumer that trusts the
index over the files has moved the trust boundary to whoever last edited the
index.

### 5.4 Phase ordering, and what it is not

Phase 1 completeness is decided by file presence: a window is ready to reveal
when `<id>-commitment.json` exists for every participant in
`PARTICIPANTS.json`.

The git history of this repository shows the order in which those files
appeared, which is useful. It is **not a trusted clock** — commit and author
dates are settable by whoever makes the commit, and both participants have write
access to their own commits. Publication order is therefore evidenced by the
commitment scheme itself (§3.2), which does not depend on anyone's timestamp,
and the git history is corroborating detail rather than proof.

A stronger time bound is available and is offered rather than required: from
window `2026-09-26T1605Z` onward, StillOS's runner commits each commitment digest
to its receipt chain as a final step after the record is already sealed on disk.
Window 1 has no such anchor and is not retroactively given one. The chain's
hourly heads are sealed, prefix-chained and free to read at
`https://stillosdigitalholdings.com/notary/head?period=YYYY-MM-DDTHH`. A
commitment appearing in the sealed head for period `H` was published before `H`
ended, checkable by anyone without asking StillOS. That anchor is **one
participant's own instrument**, so it is evidence about StillOS's publication
time that MCPShip is free to use, ignore, or mirror with its own anchor. It is
deliberately not written into the protocol as a requirement, because a protocol
that requires one party's infrastructure is not a protocol between independent
implementations.

## 6. Signature semantics

§3 binds a record to a digest. A digest alone says nothing about **who**
produced it: anyone can commit to any numbers. This section adds authorship and
is explicit about the boundary of what a signature proves.

### 6.1 Detached, never embedded

A signature over a file cannot live inside that file. Each signed artifact gets
a sidecar:

```json
{
  "alg": "ed25519",
  "signed_file": "stillos-commitment.json",
  "signed_file_sha256": "<lowercase hex>",
  "domain": "mcpship-stillos-joint-window/v1/commitment",
  "key_fingerprint": "21de066900082465",
  "signature": "<base64>"
}
```

### 6.2 What is signed

```
message = domain || 0x00 || signed_file_sha256_bytes_lowercase_hex_ascii
```

The domain string is prepended and separated by a zero byte so a signature over
a commitment can never be replayed as a signature over a reveal. The two domains
are:

```
mcpship-stillos-joint-window/v1/commitment
mcpship-stillos-joint-window/v1/reveal
```

Signing over the file's digest rather than its raw bytes keeps the signed
message a fixed 64-hex-character length regardless of artifact size, and the
digest is published in the sidecar so a verifier recomputes it from the file
before checking the signature. Both steps are required: a matching signature
over a digest that does not match the file proves nothing about the file.

`ed25519` is RFC 8032. Both recipes below were **run**, against the same
signature, before being written here — not quoted from documentation:

```js
// Node, stdlib only
const msg = Buffer.concat([Buffer.from(domain, 'utf8'), Buffer.from([0]),
                           Buffer.from(signed_file_sha256, 'ascii')]);
crypto.verify(null, msg, public_key_pem, Buffer.from(signature, 'base64'));
```

```python
# Python, cryptography
msg = domain.encode() + b'\x00' + signed_file_sha256.encode('ascii')
load_pem_public_key(public_key_pem).verify(base64.b64decode(signature), msg)
```

Both return true for a valid signature and both reject the same signature when
the domain is switched to the other one, which is the property §6.2 exists for.
Ed25519 is chosen because each side already has it in its standard library or
existing dependencies, and StillOS's notary chain is already Ed25519 under the
same keyring, so no new key material is introduced.

### 6.3 Key discovery resolves by fingerprint, never "current"

`signing_keys_url` returns:

```json
{ "keys": [ { "fingerprint": "...", "public_key_pem": "-----BEGIN PUBLIC KEY-----\n..." } ] }
```

A verifier resolves `key_fingerprint` from the sidecar against that array. It
MUST NOT verify against whichever key the endpoint currently advertises as
active. StillOS's key has already rotated once (2026-07-31), and a verifier
pinned to "current" silently fails every artifact signed before a rotation while
appearing to work on new ones — a failure mode that looks like a signature
problem and is actually a discovery problem.

Keys are therefore **never removed** from the registry. A compromised key is
marked as such with the time from which it is untrusted; artifacts signed before
that time keep verifying, and what changed is their interpretation, not their
arithmetic.

### 6.4 Unknown algorithm is not a failure of the artifact

If a verifier does not implement the `alg` named in a sidecar, the correct
outcome is **not evaluated** — a statement about the verifier — not *invalid*,
which is a statement about the artifact. Reporting an instrument's own gap as a
finding about the thing measured is the specific error the four-state
verification vocabulary exists to prevent, and it applies here as much as to
receipt verification.

### 6.5 What a signature does not prove

A valid signature proves exactly one thing: the holder of the named private key
produced these bytes.

It does not prove the counts are correct. It does not prove a sweep ran at all.
It does not prove when the file was created. It does not make the participant
honest. A participant can sign a fabricated record, and the signature will
verify — that is not a weakness in the scheme, it is the boundary of what
authorship means.

What constrains the numbers is the separate, weaker-looking property that two
independent implementations sweeping the same window converge, with neither able
to see the other's counts before committing. The signature stops a third party
from forging either side's record. It does not, and cannot, make either side's
record true.

## 7. Out of scope

- Runtime liveness. `active` / `deprecated` are Registry metadata and are not
  treated as evidence that a server answers.
- Any shared implementation, vendored module, or common dependency between the
  two sides.
- Exit codes. Each implementation keeps its own.
