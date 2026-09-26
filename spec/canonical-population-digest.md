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

The committed record body is serialized per **RFC 8785 (JCS)**: object keys sorted
**at every level**, no insignificant whitespace, arrays in document order. Two
implementations serializing the same logical record must produce identical bytes,
so key order cannot be left to insertion order.

Two constraints are added on top of JCS, both to remove the only places where two
correct implementations could still diverge:

- **Numbers are restricted to integers in `[-(2^53-1), 2^53-1]`.** JCS defers
  number formatting to ECMAScript `Number::toString`, which is the hardest part of
  the standard to reimplement and the easiest to get subtly wrong. Nothing in a
  window record is a float — every field is a count, a string, a boolean, or
  null — so the difficulty is avoidable rather than solvable. A record body
  containing a non-integer number is **malformed**, not rounded.
- **Strings are emitted as literal UTF-8**, escaping only what JSON requires:
  `"`, `\`, and U+0000–U+001F, using the two-character short forms `\b \f \n \r \t`
  where they exist and `\u00xx` otherwise. Non-ASCII is never `\u`-escaped.

> ⚠️ **JCS key ordering is by UTF-16 code unit, §2's name ordering is by Unicode
> code point. These are not the same order,** and they disagree for non-BMP
> characters: U+1F600 sorts *before* U+FFFD under UTF-16 code units and *after* it
> under code points. Both orderings are deliberate — §2 hashes a line-oriented
> name list of our own design, §3.1 follows an existing standard — but an
> implementation that reuses one comparator for both will produce a correct digest
> and a wrong commitment, or the reverse. Vector 9 below exists to catch exactly
> that.

#### 3.1.1 Canonical JSON test vectors

Computed, not asserted. `sha256` is over the canonical bytes.

Every row was produced by a Node stdlib implementation and reproduced
byte-for-byte by a second implementation in Python before publication. **Both
were written by StillOS, so this is a transcription check, not independent
confirmation** — it catches a typo or a language-specific string-handling
surprise, and it cannot catch a shared misreading of RFC 8785. The vectors become
independently confirmed when MCPShip reproduces them from its own code, which is
the reason they are published as bytes and digests rather than as a library.

What *is* verified against a source outside StillOS: the escape behaviour below
matches ECMA-262 `JSON.stringify` across all of U+0000–U+007F and for non-BMP
input, and the ordering, number and escaping rules were read from RFC 8785 rather
than recalled.

| # | Property | Input | Canonical bytes | SHA-256 |
| --- | --- | --- | --- | --- |
| 1 | empty object | `{}` | `{}` | `44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a` |
| 2 | empty array | `[]` | `[]` | `4f53cda18c2baa0c0354bb5f9a3ecbe5ed12ab4d8e11ba873c2f11161202b945` |
| 3 | keys sorted, top level | `{"b":1,"a":2}` | `{"a":2,"b":1}` | `d3626ac30a87e6f7a6428233b3c68299976865fa5508e4267c5415c76af7a772` |
| 4 | keys sorted at **every** level, empty key | `{"z":{"d":1,"c":2},"a":{"b":3,"":4}}` | `{"a":{"":4,"b":3},"z":{"c":2,"d":1}}` | `bfc2673dd6d9b854eaf96405f2686767a9d7c06afe5ede4f1d5448a1bdac678d` |
| 5 | arrays keep document order | `{"a":[3,1,2]}` | `{"a":[3,1,2]}` | `d934227f5a9f29b31bd6a8ceed3213dd498a07ce32bdfa9ae2488de0c0ab68f1` |
| 6 | integers / bool / null / `-0` → `0` | `{"n":null,"t":true,"f":false,"i":-0,"big":9007199254740991,"neg":-17}` | `{"big":9007199254740991,"f":false,"i":0,"n":null,"neg":-17,"t":true}` | `151269d7d237391bf7271983ad366670a165a439d0846e6769861fd249a25abc` |
| 7 | escapes; control chars are `\u00xx` | JSON string `"a\"b\\c\nd\te\u0001f"` as the value of key `s` | `{"s":"a\"b\\c\nd\te\u0001f"}` — pure ASCII, 28 bytes | `58a6f3f7e956012dfe12e838b3c0e7eed1f2a90082a78bc86271b358d4ae0a50` |
| 8 | non-ASCII stays literal UTF-8 | `{"k":"é中"}` | `{"k":"é中"}` (13 bytes) | `46af7c8fb8b65095424ce987112c67d2af79db4203787e6672236d2c716a4459` |
| 9 | **UTF-16 vs code-point ordering** | `{"😀":1,"�":2}` | `{"😀":1,"�":2}` (18 bytes) | `fd8b688bfa8b71822975ab3519e20b09e43b67d382a9f32831bfa384df21a82d` |
| 10 | realistic record body | see below | see below (335 bytes) | `ce61d01b9cc639847b177c23bdcddb574448a7c02ef6f75b4e0c2086f082667f` |

Vector 10's canonical bytes in full, so a fresh implementation has one full-size
case and not only minimal ones:

```
{"agreed_start_utc":"2026-09-26T16:05:00Z","canonical_population_sha256":"43782c49a3129acc84b8d6b28137dd4e532bdb10a52382d34a5c840d4354e494","counts":{"active":35599,"deprecated":387,"duplicates":0,"pages":360,"raw_rows":35986,"unique":35986},"participant_id":"stillos","stop_reason":"natural_exhaustion","window_id":"2026-09-26T1605Z"}
```

Vector 9 is the one worth running first. An implementation that sorts keys by code
point instead of UTF-16 code unit emits `{"�":2,"😀":1}` and gets a different
digest, while passing vectors 1–8 and 10 unchanged. It is the only row here that
fails silently on a plausible mistake rather than an obvious one.

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
3. Reveal opens as soon as **every** participant in `PARTICIPANTS.json` has a
   commitment published — not at any fixed time. Each side then publishes its
   record body and nonce. A full set of commitments is the trigger; the §5.4
   deadline is **not** a waiting period and never delays a window in which
   everyone showed up.
4. If the set is still incomplete at `agreed_start + 6h`, the deadline fires and
   §5.4 takes over: the participant set freezes to whoever committed, those
   participants reveal, and the window is `partial`. The deadline exists only to
   stop a missing participant from stalling the series — it is the fallback
   branch, not the normal one.
5. Either party — or any third party — recomputes
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
      "signing_keys_url": null,
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

**`signing_keys_url` is `null` for MCPShip because MCPShip does not publish a
signing-key endpoint.** That is recorded as a fact, not as a pending item. It has
three consequences, stated here so no other section of this document can imply
otherwise:

1. MCPShip's artifacts are **unsigned**, and this specification does not describe
   them as signed anywhere. Earlier drafts showed a placeholder URL in this slot,
   which asserted a capability that does not exist — the placeholder is removed
   rather than left to be filled in later.
2. A participant with `signing_keys_url: null` MUST NOT publish `.sig.json`
   sidecars. A sidecar from such a participant is **unverifiable by construction**
   and a verifier must report it as such, never as valid.
3. A verifier encountering an artifact with no sidecar from a `null`-key
   participant reports **`unsigned`** — a fourth outcome, distinct from `valid`,
   `invalid`, and §6.4's `not_evaluated`. `unsigned` is a statement about the
   artifact, `not_evaluated` is a statement about the verifier, and `invalid` is
   an accusation. Collapsing any of the three into another is the error §6.4
   exists to prevent.

`unsigned` does not weaken the window. Per §6.5, a signature never made any count
true; what constrains the numbers is two independent implementations converging
without either seeing the other's counts first. An unsigned MCPShip artifact is
fully eligible to be half of a joint observation. What it cannot do is prove
authorship to a third party who did not receive it from MCPShip directly — and
that limit is MCPShip's to close, whenever and if ever it chooses to.

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

`state` is one of:

| `state` | Meaning | Carries convergence evidence? |
| --- | --- | --- |
| `committed` | at least one commitment published, reveal phase not open yet | not yet |
| `revealed` | **every** participant committed and revealed, all reveals recompute | **yes** |
| `partial` | reveal deadline passed with at least one participant absent (§5.4) | **no** — one-sided observation |
| `rejected` | a reveal did not recompute to its commitment (§3.4) | no |
| `incomplete` | no participant produced an eligible commitment | no |

The `partial` / `revealed` split is the whole point of the enum. A consumer
computing "how often do the two implementations agree" MUST filter to `revealed`;
counting `partial` windows in that denominator would let either side improve the
agreement rate by not showing up.

**`state` alone is not sufficient, and the very first window proves it.**
`results/2026-09-25/` is recorded as `revealed`, but it predates this protocol:
both participants have `commit_reveal_used: false` and `commitment_sha256: null`,
because neither committed before seeing the other's counts. It converged on all
six counts, and MCPShip published blind — but StillOS did not, so the window
carries no *structural* guarantee against reading order, only a recorded
observation about how it happened.

So the filter for convergence evidence is:

```
state == "revealed"  AND  every participant has commit_reveal_used == true
```

Not `state == "revealed"` alone. Applying the looser filter would score window 1
as protocol-grade evidence, which is exactly the retroactive upgrade §5.4.2 and
§2.2 both refuse. Window 1 is kept in the index at its real status rather than
being promoted or deleted.

`windows.json` is a **convenience index, never the authority**. Every value in
it is recomputable from the files in `results/`. A consumer that trusts the
index over the files has moved the trust boundary to whoever last edited the
index.

### 5.4 Phase ordering, missing participants, and what none of it is

Phase 1 completeness is decided by file presence: a window is ready to reveal
when `<id>-commitment.json` exists for every participant in
`PARTICIPANTS.json`.

#### 5.4.1 A missing participant must not stall the series

As previously written, that rule waits forever. If one side's box is down, its
cron is wedged, or it simply skips a day, the other side's completed sweep is
held hostage and the window is lost. **The series is the one asset here that
cannot be backfilled by anyone, including the participants** — a window nobody
records is gone permanently — so an absent participant must cost one window's
*cross-check*, never one window's *observation*.

A single deadline fixes it:

```
freeze_deadline_utc = agreed_start_utc + 6h
```

**The deadline is a fallback, never a waiting period.** Reveal is triggered by a
complete set of commitments, and only by that:

- **As soon as every participant has a commitment published** — at any time, however
  early — the set is complete, reveal opens immediately, and the window resolves
  `revealed`. Nobody waits for the clock. An earlier draft of §3.3 and this section
  disagreed about this, which would have idled a window in which both sides did
  everything right.
- **Only if the set is still incomplete at `freeze_deadline_utc`** does the deadline
  fire. The participant set freezes to whoever has a commitment on file, those
  participants reveal, and:
  - non-empty proper subset → `partial`
  - empty → `incomplete`
- **After the freeze fires**, a late commitment is **not eligible** for that window.
  It is not added, not merged, and does not reopen the window. A participant that
  missed it may publish its record as an ordinary standalone artifact, clearly
  outside `results/<window_id>/`, carrying no joint standing.

#### 5.4.1a The freeze must be recorded, or `partial` is not reconstructable

`windows.json` is derived from the final `results/` tree (§5.5), but `partial`
depends on **which commitments existed at the deadline** — and a late commitment
sitting in the final tree is byte-for-byte indistinguishable from an on-time one.
A deriver reading only the tree would therefore promote a frozen `partial` window
to `revealed`, which is exactly the retroactive upgrade §5.3 forbids.

So the freeze writes one more file, and it is the only thing in this protocol whose
absence changes a verdict:

```
results/<window_id>/FREEZE.json
{
  "window_id": "2026-09-26T1605Z",
  "freeze_deadline_utc": "2026-09-26T22:05:00Z",
  "frozen_participant_set": ["stillos"],
  "absent_at_deadline": ["mcpship"],
  "frozen_by": "stillos"
}
```

Derivation rule, unambiguous in both directions:

- **`FREEZE.json` absent** → the window was never frozen → eligibility is simply
  "has a commitment in the tree", and the state is `revealed` if that covers every
  participant.
- **`FREEZE.json` present** → eligibility is `frozen_participant_set`, **verbatim and
  exclusively**. Any commitment in the tree from a participant outside that set is
  late by definition, is excluded from the window, and does not change the state.

`frozen_by` records who wrote it, because a freeze is one participant asserting a
wall-clock fact the other cannot check. That is acceptable for the same reason
§5.4.2 gives — a frozen window is `partial`, and `partial` asserts no agreement, so
there is nothing to gain by freezing early. A participant who disputes a freeze says
so in the thread; the file is evidence of what was claimed, not proof it was true.

Six hours is long enough to absorb a wedged cron caught by the next hourly check
and a manual restart, and short enough that a window resolves the same day it ran.
It is a parameter, not a principle; if it proves wrong, change the number and
record the change — do not special-case a window.

#### 5.4.2 Why the deadline does not need to be trustworthy

The deadline is wall-clock, and neither side can prove to the other what time it
published. That looks like a hole. It is not, because **a `partial` window makes
no cross-implementation claim in the first place.**

The attack the deadline would have to resist is: withhold my commitment, wait for
your reveal, then publish a commitment matching your numbers. That attack buys
nothing. Once the deadline has passed the window is `partial` and the late
commitment is ineligible; if instead the attacker publishes on time, it is bound
by §3.2 before it can see anything. There is no ordering in which late
publication produces a `revealed` window. So the deadline is a **liveness**
parameter — it decides when we stop waiting — and never a **safety** one. Safety
is carried entirely by the commitment scheme.

This is also why `partial` windows must be counted honestly and never quietly
promoted: the moment a `partial` window is presented as agreement, the deadline
*would* need to be trustworthy, and it isn't.

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

### 5.5 Concurrent writers

Two unattended daily jobs committing to one repository will collide. The rule
here is not a locking protocol — it is to **remove the shared mutable file from
the path that matters.**

**Every file a participant writes is under a path containing its own
`participant_id`.** `results/<window_id>/<id>-commitment.json`,
`results/<window_id>/<id>-reveal.json`, and their sidecars are disjoint by
construction. Two participants running at the same instant touch no common file,
so the common case has no conflict to resolve. Git's own failure mode here —
non-fast-forward push — is a push-time retry, not a merge: fetch, rebase onto the
remote tip, push again. Because the paths are disjoint, that rebase can never
produce a content conflict.

**`windows.json` is derived, never authored.** It is the output of a pure
function over `results/`, defined so that any participant regenerating it from
the same tree produces byte-identical output:

1. Enumerate every directory under `results/`. `window_id` is the directory
   name **verbatim**. The §5.2 grammar constrains the names of *new* windows; it
   is not a filter applied here, because filtering on it would silently drop the
   grandfathered `results/2026-09-25/` — an entry that is already published in
   `windows.json` — the first time anyone regenerated the index. A derivation
   rule that deletes real data on a clean run is worse than no rule.
2. Sort windows ascending by `window_id` as a byte string. `YYYY-MM-DD` and
   `YYYY-MM-DDTHHMMZ` are both fixed-width and zero-padded and share a prefix,
   so byte order is chronological order across both forms.
3. Within a window, sort participants ascending by `participant_id` as a byte
   string — **not** by publication order, which differs between observers.
4. Derive `state` from file presence and recomputation per §5.3/§5.4, never from
   the previous contents of `windows.json`. **Eligibility comes from
   `results/<window_id>/FREEZE.json` when that file exists (§5.4.1a), and from
   plain commitment presence when it does not.** Without this step the derivation
   is not a pure function of the tree for frozen windows: a late commitment and an
   on-time one are identical bytes, so a deriver that ignores `FREEZE.json` will
   silently promote `partial` to `revealed`.
5. Serialize with the §3.1 canonical JSON rules, then append a single trailing
   `\n`.

The conflict rule follows directly: **a merge conflict in `windows.json` is
resolved by deleting the file and regenerating it — never by merging hunks.**
Two correct implementations regenerating from the same `results/` tree produce
identical bytes, so the conflict carries no information worth preserving. Any
diff that survives regeneration is a real disagreement about the underlying
files and must be reported, not resolved in an editor.

This keeps §5.3's existing claim honest — the index is a convenience, never the
authority. A file that is regenerated on every conflict cannot quietly become the
source of truth, because nothing it contains survives that regeneration except
what the `results/` files already say.

Ordering note: a participant writes its own artifact files **first** and
regenerates `windows.json` **second**, in that order, and never regenerates the
index from a tree it has not just fetched. An index generated before the
participant's own files are on disk describes a state that never existed.

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

### 6.5 Known-answer vector

§6.2 gives two code recipes. A recipe is not a vector: two implementations can
each run their own code, each get `true`, and still disagree about what was
signed. This section pins the exact bytes.

> ⚠️ **The private key below is public and is a published test key. Never trust a
> signature from it and never use it for a real artifact.** It is RFC 8032 §7.1
> test vector 1's secret, chosen precisely because it is already public
> everywhere, so publishing it here leaks nothing and any implementation can
> reproduce the **signing** step as well as verification.

```
ed25519 seed (hex, PUBLIC TEST KEY)
  9d61b19deffd5a60ba844af492ec2cc44449c5697b326919703bac031cae7f60
ed25519 public key (raw, hex)
  d75a980182b10ab7d54bfed3c964073a0ee172f3daa62325af021a68f707511a
```

```
-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEA11qYAYKxCrfVS/7TyWQHOg7hcvPapiMlrwIaaPcHURo=
-----END PUBLIC KEY-----
```

Signed file (exact bytes, 59 bytes, no trailing newline):

```
{"window_id":"2026-09-26T1605Z","participant_id":"stillos"}
```

```
signed_file_sha256
  2f4d7f54134e7bf51f4598fb031a03991c37ca9e073b80cfae9ab513a1ec73f3
```

The signed message per §6.2, in full, so there is nothing left to interpret —
`domain` ASCII, then one `0x00`, then the 64 ASCII characters of the lowercase
hex digest (107 bytes for the commitment domain):

```
message (hex, commitment domain)
  6d6370736869702d7374696c6c6f732d6a6f696e742d77696e646f772f76312f636f6d6d69746d656e74
  00
  32663464376635343133346537626635316634353938666230333161303339393163333763613965303733623830636661653961623531336131656337336633
```

| Domain | Signature (base64) |
| --- | --- |
| `mcpship-stillos-joint-window/v1/commitment` | `rsvaf3a0NdgQORicr2qPFMZaaqTtjHWZHfDCU0MwSxPoKfu9CrBCYjO2BfzJpD6ZgXk/2NLs2BEiU8sBq6VkCA==` |
| `mcpship-stillos-joint-window/v1/reveal` | `e0TdptxvcZZk9EfWkL7KMG12cuXfJGrL64VfZ2X4lGwFqvbT/gfI1pqAyXspQkTYkofWNT7gNvDZ+ejN9WU2Cg==` |

A conforming implementation reproduces this 2×2, and the **two `false` cells
matter more than the two `true` ones** — they are the only evidence that domain
separation was actually implemented rather than declared:

| | verified under `/commitment` | verified under `/reveal` |
| --- | --- | --- |
| commitment signature | `true` | `false` |
| reveal signature | `false` | `true` |

Both signatures above were produced with Node's stdlib `crypto` and then verified
— including both rejections — in Python using `cryptography`, before being
written here. As in §3.1.1, both are StillOS implementations, so that is a
transcription check across two crypto libraries rather than independent
confirmation. The key material is the part that is externally anchored: the seed
and public key are RFC 8032 §7.1 TEST 1, read from the RFC.

An implementation that omits the `0x00` separator, signs the raw file bytes
instead of the digest's ASCII, or signs the digest as 32 raw bytes instead of 64
hex characters will fail all four cells rather than three, which is the useful
kind of failure.

### 6.6 What a signature does not prove

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
