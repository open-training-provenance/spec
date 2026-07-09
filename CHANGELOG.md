# Changelog

All notable changes to the OpenTrainingProvenance specification are recorded here.
Changes are grouped into **Normative** (they change what a conformant implementation
must do — an implementation that ignores them is non-conformant) and **Editorial**
(clarifications that do not change conformant behavior).

The **spec-document version** (this file, and the SPEC.md header) is versioned
independently from the **manifest wire-format version** (`spec_version` inside each
manifest). See SPEC.md §11.

## [0.2] — 2026-07-09

Spec-document revision over the **unchanged 0.1 manifest wire format**. Manifests
continue to carry `spec_version: "0.1"`; the byte layout and parser are unchanged, so
every honest v0.1 manifest remains valid under v0.2.

### Normative

- **§7.2 — `anchored_hash` reconciliation (new verification requirement).** A verifier
  MUST confirm that the timestamp proof commits to the reconstructed `anchored_hash` of
  the manifest being verified — not merely that the proof is a valid anchor of *some*
  digest. This closes the "attach an unrelated but validly-anchored proof to a manifest"
  hole. A proof that decodes/parses but commits to a different digest, or that fails to
  decode/parse, yields an **invalid** timestamp, and a conformant verifier MUST NOT
  report such a manifest as validly timestamped. This requirement only ever *rejects*
  manifests whose proof does not attest them; it never rejects an honest manifest.
- **§6.1.1 — no floating-point numbers.** A conformant manifest MUST NOT contain
  floating-point numbers; every number is an integer in 0…2⁵³−1. This makes the canonical
  serialization fully deterministic without a floating-point serializer. (Formalizes a
  constraint the 0.1 fixed fields already satisfied; enforced by implementations at sign
  and verify time, since the JSON Schema cannot express it for the open `claims`/`links`
  objects.)

### Editorial

- **§5 — corpus hash made byte-exact.** Replaced the ambiguous "no trailing newline beyond
  the final record" wording with an exact byte layout (64-hex digest + `0x09` + UTF-8 path
  + `0x0A` per record; sort by path as raw UTF-8 bytes; every record retains its trailing
  line feed). Added §5.1, a worked byte-level example reproducing
  `sha256:f912628d…438f18c4`.
- **§6.1 — canonicalization pinned to RFC 8785 (JCS).** Replaced the hand-rolled canonical
  rules with a normative reference to RFC 8785; forward slash `/` is not escaped; absent
  optional fields are omitted (not null). (The previous text was described as "equivalent
  to RFC 8785" but diverged in edge cases; this removes the divergence.)
- **§6.4 — computation-order table added.** Documents the three distinct hash fill-states
  (`manifest_id` derivation: all three computed fields null; signature: `signature`+
  `timestamp` null, `manifest_id` populated; `anchored_hash`: `timestamp` null, signature
  present) and pins their order, de-circularizing `manifest_id`.
- **§3.2 — `manifest_id` base32 pinned to RFC 4648** (uppercase, no padding) and nulled in
  its own derivation.
- **§7 — anchor-agnostic vocabulary.** The `timestamp` object is described as a generic
  "timestamp anchor" selected by `timestamp.algorithm` (`"opentimestamps"` is the only v0.2
  backend). The word "confirmation" is avoided for the offline check (it collides with
  Bitcoin's block-depth meaning); the offline reconciliation and the ledger confirmation are
  named as distinct steps.
- **§2 / abstract — "Merkle-style"** corpus-hash wording corrected to "sorted file list"
  (Merkle-tree hashing is deferred to a future version, not v0.2).
- **§11 — versioning** rewritten to distinguish the spec-document version from the wire-format
  version.
- **Header — Editor** set to `OpenTrainingProvenance`.

### Not changed

- Manifest wire format (`spec_version` stays `"0.1"`), JSON Schema, and all field names.

## [0.1] — 2026-07-08

- Initial public draft: manifest structure, corpus hash, Ed25519 signing, OpenTimestamps
  anchoring, and the §8 trust boundary (proven / socially trusted / out of scope).
- Public at https://github.com/open-training-provenance/spec.
