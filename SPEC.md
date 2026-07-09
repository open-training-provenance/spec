# OpenTrainingProvenance Manifest Specification

**Version:** 0.2 (draft)
**Status:** Public draft, open for comment
**Editor:** OpenTrainingProvenance
**License:** MIT

> **v0.2 note.** This is a spec-document revision over the **unchanged 0.1 manifest wire format**. It adds two normative requirements — the §7.2 `anchored_hash` reconciliation (a new verification obligation that closes the "attach someone else's valid timestamp" hole) and the §6.1.1 no-floating-point constraint — plus editorial byte-exactness clarifications (§5, §6.1, §6.4) and anchor-agnostic vocabulary (§7). Manifests continue to carry `spec_version: "0.1"` because the wire format and parser are unchanged; see §11 and [CHANGELOG.md](./CHANGELOG.md).

## Abstract

This document specifies the format of a **provenance manifest** for AI training data: a JSON document that cryptographically commits to a corpus of files, identifies its publisher, records provenance claims (license, jurisdiction, description, contributors), and anchors these commitments in time via a public blockchain.

The specification is minimal and deliberately honest about its scope. A conformant manifest lets a third party verify **that a specific corpus existed and was published by a specific key at a specific time**, and lets that party inspect the publisher's claims about the corpus. It does **not** attempt to prove that any specific model was trained on the corpus, nor to enforce license terms downstream. Those problems remain open in 2026 and no cryptographic primitive currently deployed at consumer scale solves them.

## 1. Motivation

AI training data provenance is unsolved. Every open-weights model release contains claims about training data ("trained on FineWeb," "trained on our internal corpus," "trained on public code") that no third party can independently verify. The consequences include:

- **Reproducibility crisis.** Papers can't be replicated because training corpora aren't specified precisely.
- **Contamination.** Benchmarks routinely leak into training sets; without provenance there's no way to detect this after the fact.
- **License opacity.** Copyright-suspect data enters training pipelines with no audit trail.
- **Regulatory pressure.** EU AI Act Article 10, upcoming U.S. state laws, and industry-specific rules (HIPAA-adjacent AI, financial compliance) increasingly require training-data governance. There is no open standard for making the underlying attestations.
- **Emerging decentralized training networks** (Nous Psyche, Prime Intellect, Bittensor SN13, IOTA) generate training data at unprecedented volume with no shared provenance layer.

This spec provides a small, correct, honest primitive that solves one narrow problem — **verifiable identification and publication of a training corpus** — and stays out of the harder problems that current cryptography cannot solve efficiently.

The design draws on prior art that predates blockchain:

- Merkle (1979): cryptographic hashing of file sets into single roots.
- Haber & Stornetta (1991): timestamping documents by chaining hashes.
- Bernstein et al. (2011): Ed25519 signatures.
- Todd (2016): OpenTimestamps as a free, permissionless Bitcoin-anchored timestamping service.

The spec is what these primitives compose into when applied to AI training data.

## 2. Definitions

**Corpus.** An ordered set of files intended as input to an AI training or evaluation pipeline. A corpus may contain data of any modality (text, code, images, audio, tabular, etc.).

**Manifest.** A JSON document conforming to this specification. A manifest cryptographically commits to a corpus and records provenance claims.

**Publisher.** The entity that produces and signs a manifest. Identified by an Ed25519 public key. A publisher may be an individual, an organization, or a pseudonymous identity.

**Verifier.** Any party inspecting a manifest to check its cryptographic validity and the publisher's claims.

**Corpus hash.** The content hash of the corpus (a SHA-256 over the sorted file list), computed as defined in §5.

**Manifest signature.** An Ed25519 signature by the publisher over the canonical serialization of the manifest, computed as defined in §6.

**Timestamp proof.** An OpenTimestamps proof committing the manifest hash to the Bitcoin blockchain, as defined in §7.

## 3. Manifest structure

A manifest is a JSON object with the following top-level fields:

```json
{
  "spec_version": "0.1",
  "manifest_id": "urn:otp:<base32-hash-prefix>",
  "corpus": { ... },
  "publisher": { ... },
  "claims": { ... },
  "signature": { ... },
  "timestamp": { ... },
  "links": { ... }
}
```

### 3.1 `spec_version`

String. The **wire-format** version the manifest conforms to — always `"0.1"` for manifests conforming to this document. Note this is distinct from the spec-*document* version in the header (currently 0.2): the document version advances when the spec adds verification requirements or clarifications, while `spec_version` changes only when the manifest byte layout or parser changes. See §11.

### 3.2 `manifest_id`

String. A URN of the form `urn:otp:<base32>` where `<base32>` is the first 26 characters of the RFC 4648 base32 encoding (uppercase alphabet `A–Z2–7`, no padding) of the SHA-256 digest of the canonical manifest bytes (see §6.1) with the `manifest_id`, `signature`, and `timestamp` fields all set to `null`. Because `manifest_id` is itself nulled in the bytes it commits to, the derivation is well-defined and non-circular. Deterministic — the same manifest content always produces the same ID. The exact assembly order is given in §6.4.

### 3.3 `corpus`

Object describing the corpus:

```json
{
  "corpus_hash": "sha256:<hex>",
  "hash_algorithm": "otp-file-list-sha256-v1",
  "size_bytes": 12345678,
  "file_count": 42,
  "format": "jsonl",
  "description": "10,000 agent tool-use trajectories from Qwen3-4B-Instruct-2507 on HumanEval-style problems"
}
```

- `corpus_hash`: the corpus hash per §5, prefixed with the algorithm identifier.
- `hash_algorithm`: identifier for the hashing procedure. This spec defines `otp-file-list-sha256-v1`. Future versions may add algorithms.
- `size_bytes`: total size of the corpus in bytes.
- `file_count`: number of files in the corpus.
- `format`: a human-readable format identifier. Common values: `"jsonl"`, `"parquet"`, `"csv"`, `"txt"`, `"binary"`, `"mixed"`.
- `description`: a plain-English description of the corpus contents.

### 3.4 `publisher`

Object identifying the publisher:

```json
{
  "public_key": "ed25519:<base64>",
  "key_type": "ed25519",
  "name": "Steven Kozeniesky",
  "contact": "https://github.com/stevenk"
}
```

- `public_key`: the publisher's Ed25519 public key, base64-encoded, prefixed with the key type.
- `key_type`: currently only `"ed25519"` is defined.
- `name`: optional human-readable name of the publisher.
- `contact`: optional URL or identifier where the publisher can be reached (GitHub, personal site, DID, etc.).

### 3.5 `claims`

Object containing the publisher's provenance claims. **All fields in this object are publisher assertions, not cryptographic facts.** See §8 (Trust Model).

```json
{
  "license": {
    "spdx": "CC-BY-4.0",
    "url": "https://creativecommons.org/licenses/by/4.0/"
  },
  "jurisdiction": "US-CA",
  "consent_statement": "All source data derived from publicly-available benchmarks with permissive licensing. No user data.",
  "purpose": "training",
  "created_at": "2026-07-08T18:00:00Z",
  "contributors": [
    {"name": "Steven K.", "role": "primary curator"}
  ],
  "notes": "Generated via ..."
}
```

- `license.spdx`: an SPDX license identifier (https://spdx.org/licenses/). Use `"proprietary"` or `"custom"` for non-standard licenses and provide a URL.
- `license.url`: optional URL to the full license text.
- `jurisdiction`: ISO 3166-2 code where the publisher asserts they are producing this corpus. Optional but recommended.
- `consent_statement`: plain-English statement of consent basis for the underlying data.
- `purpose`: intended use. Values: `"training"`, `"evaluation"`, `"fine-tuning"`, `"pretraining"`, `"research"`, `"other"`.
- `created_at`: ISO 8601 timestamp of corpus creation (publisher's claim, not cryptographically anchored — see `timestamp` for the cryptographic time).
- `contributors`: optional list of contributor objects.
- `notes`: arbitrary human-readable notes.

### 3.6 `signature`

Object containing the publisher's Ed25519 signature over the canonical manifest bytes:

```json
{
  "algorithm": "ed25519",
  "value": "<base64>",
  "signed_over": "canonical-json-v1-excluding-signature-and-timestamp"
}
```

- `algorithm`: signature algorithm. Currently only `"ed25519"`.
- `value`: the signature, base64-encoded.
- `signed_over`: identifier for the canonicalization procedure. This spec defines `"canonical-json-v1-excluding-signature-and-timestamp"` — see §6.

### 3.7 `timestamp`

Object containing an OpenTimestamps proof:

```json
{
  "algorithm": "opentimestamps",
  "ots_proof": "<base64 of .ots file contents>",
  "anchored_hash": "sha256:<hex>",
  "block_height": null,
  "block_time": null
}
```

- `algorithm`: currently `"opentimestamps"`.
- `ots_proof`: the raw OpenTimestamps proof (`.ots` file contents), base64-encoded.
- `anchored_hash`: the SHA-256 hash that was submitted to OpenTimestamps. This MUST equal the SHA-256 of the canonical manifest bytes with `signature` present but `timestamp` set to `null`. See §7.
- `block_height`, `block_time`: optional cached fields populated after the OTS proof upgrades. A verifier MUST recompute these from the OTS proof rather than trusting the cached values.

### 3.8 `links`

Optional object with links to related resources:

```json
{
  "corpus_url": "ar://<arweave-tx-id>",
  "corpus_mirror_urls": ["ipfs://<cid>", "https://example.org/corpus.tar.gz"],
  "prior_version": "urn:otp:<previous-manifest-id>",
  "contributor_manifests": [],
  "downstream_models": []
}
```

All fields optional. Verifiers can use these to fetch the corpus for verification.

## 4. Overall example

See [`examples/manifest.example.json`](./examples/manifest.example.json) for a worked example.

## 5. Corpus hash algorithm (`otp-file-list-sha256-v1`)

The corpus hash commits to the content and layout of every file in the corpus. It is computed as follows.

1. Enumerate every file in the corpus and express its location as a relative path from the corpus root. Paths use forward slashes (`/`) as separators, contain no `.` or `..` segments and no leading `/`, and are encoded as UTF-8 in Unicode Normalization Form C (NFC).
2. For each file, compute the SHA-256 digest of its raw bytes and encode that digest as a lowercase, 64-character hexadecimal string.
3. For each file, build a **record** consisting of exactly these bytes, in order, with no other bytes:
   - the 64 ASCII bytes of the lowercase hex digest;
   - one horizontal-tab byte (`0x09`);
   - the UTF-8 bytes of the relative path;
   - one line-feed byte (`0x0A`).
4. Sort the records by relative path, comparing paths as raw UTF-8 byte sequences (bytewise — not locale-aware and not case-folded).
5. Concatenate the sorted records in order, with nothing added or removed between or after them. Every record — including the final one — retains its own terminating line-feed byte; no separator is inserted between records and none is stripped from the last.
6. Compute the SHA-256 digest of the concatenated byte sequence. The `corpus_hash` is that digest as a lowercase hex string prefixed with `sha256:`.

### 5.1 Worked byte-level example

The three-file corpus in [`examples/example-corpus/`](./examples/example-corpus/) yields these per-file digests:

| Relative path | SHA-256 of file bytes |
|---|---|
| `README.md` | `9ca0c10f2cc1a0865f9bf3ff9f01c620c2cd4397304f5a18a61aec12f1f2733a` |
| `example-1.txt` | `1084853246a01090c66ee3ebc471a80eff0039eff91ac1d780084abe1bfdb30d` |
| `example-2.txt` | `0ca7a97688901c0f5f9f1e485c49b921fd587b350fe83af9dadf5ff4b576e7e4` |

Sorted by path — `README.md` precedes `example-…` because `R` (`0x52`) sorts before `e` (`0x65`) — the concatenated bytes are the following, where `⇥` denotes the tab byte `0x09` and `␤` denotes the line-feed byte `0x0A` (there is no line break inside the stream; it is one continuous sequence):

```
9ca0c10f2cc1a0865f9bf3ff9f01c620c2cd4397304f5a18a61aec12f1f2733a⇥README.md␤1084853246a01090c66ee3ebc471a80eff0039eff91ac1d780084abe1bfdb30d⇥example-1.txt␤0ca7a97688901c0f5f9f1e485c49b921fd587b350fe83af9dadf5ff4b576e7e4⇥example-2.txt␤
```

The SHA-256 of those bytes is `f912628de77e191ffe14eb4a8f4cbd14acb708e2de10f5ca45956393438f18c4`, so:

```
corpus_hash = "sha256:f912628de77e191ffe14eb4a8f4cbd14acb708e2de10f5ca45956393438f18c4"
```

which matches [`examples/manifest.example.json`](./examples/manifest.example.json).

This procedure is deliberately simple. It commits to the entire corpus content and layout, and a verifier who holds the corpus can reproduce the hash with standard tools. A future spec version will add a Merkle-tree variant supporting proofs of individual file membership without downloading the whole corpus.

**Rationale:** avoiding IPFS CIDs, tarball hashing, and full Merkle trees in this version keeps the primitive dependency-free and inspectable with a shell one-liner.

## 6. Canonicalization and signing

Manifests are signed and hashed over a **canonical** JSON serialization defined here.

### 6.1 Canonical serialization

Manifests are hashed and signed over a single, deterministic byte serialization so that two independent implementations produce identical bytes for the same manifest content. The canonical serialization is **RFC 8785 (JSON Canonicalization Scheme, JCS)**, subject to the additional constraints in §6.1.1. A conformant implementation MUST produce byte-for-byte identical output to RFC 8785 for every manifest it emits or verifies.

Informally, RFC 8785 requires: UTF-8 encoding; recursive lexicographic (UTF-16 code-unit) sorting of object member names; no insignificant whitespace; strings escaped per RFC 8259 in shortest form (only `"`, `\`, and the C0 control characters are escaped — the forward slash `/` is **not** escaped); and a fixed number serialization.

#### 6.1.1 Number and field constraints

To keep canonicalization fully deterministic without a floating-point serializer — and to preserve the ability to inspect a manifest with ordinary command-line tools — a conformant manifest is further constrained:

- **No floating-point numbers.** Every JSON number in a manifest MUST be an integer in the range 0 … 2⁵³−1 (`size_bytes`, `file_count`, `block_height`, and any future numeric field). Manifests MUST NOT contain fractional or exponent-form numbers. Integers are serialized as their minimal decimal form with no leading zeros, no sign, and no exponent — which is exactly what RFC 8785 produces for integers in this range. The JSON Schema constrains the known numeric fields to integers, but cannot express this rule for the open `claims` and `links` objects; a conformant implementation therefore MUST reject any non-integer number anywhere in a manifest at sign and verify time.
- **Absent optional fields are omitted, not nulled.** An optional field carrying no value is simply absent from the object; it is neither present as `null` nor otherwise materialized. The only fields explicitly set to `null` during a computation are `manifest_id`, `signature`, and `timestamp`, as defined in §3.2, §6.2, §6.4, and §7.1 — these are *present-but-null*, not absent.
- **Object keys are ASCII.** All object member names in a manifest — including keys inside the open `claims` and `links` objects — are expected to be ASCII. RFC 8785 orders member names by UTF-16 code unit; for ASCII keys this equals byte order. Because every field this spec defines is ASCII, a conformant implementation MAY reject a manifest containing a non-ASCII object key rather than implement full UTF-16 ordering — failing loud rather than risk emitting non-JCS bytes. (The reference implementation rejects them.)

> **Design note.** Because a manifest contains no floating-point values, its canonical form is fully determined by JCS's integer and string rules alone; none of JCS's floating-point machinery (shortest round-trip / Ryū) is exercised. This is deliberate: it keeps a manifest reproducible with a small script plus a SHA-256 utility, without a full JCS library, while remaining a strict subset of RFC 8785. Contrast with formats that carry fractional fields (e.g. a proportional-weighting value), which cannot avoid the full JCS number algorithm.

### 6.2 Signature computation

To compute the manifest signature (after `manifest_id` has been derived per §6.4):

1. Take the manifest with `manifest_id` populated.
2. Set the entire `signature` field and the entire `timestamp` field to `null`.
3. Compute the canonical serialization per §6.1.
4. Sign the resulting bytes with the publisher's Ed25519 private key.
5. Construct the `signature` object: place the signature (base64) in `signature.value`, with `signature.algorithm` = `"ed25519"` and `signature.signed_over` = `"canonical-json-v1-excluding-signature-and-timestamp"`.

Note: because the entire `signature` field is `null` in the signed bytes, `signature.algorithm` and `signature.signed_over` are not themselves covered by the signature. In this version both are fixed single values (see the schema), so this carries no practical risk; a future version introducing algorithm agility will bind them explicitly.

### 6.3 Signature verification

To verify:

1. Take the manifest as received; retain the received `signature.value` for step 4.
2. Set the entire `signature` field and the entire `timestamp` field to `null`, leaving `manifest_id` populated.
3. Compute the canonical serialization per §6.1.
4. Verify the retained Ed25519 signature against the canonical bytes using the publisher's public key from `publisher.public_key`.

### 6.4 Manifest assembly and computation order

A manifest commits to itself with three distinct SHA-256 computations, each taken over the canonical serialization (§6.1) of the manifest in a specific fill-state. Because each covers a different set of populated fields, they MUST be performed in this order:

1. **Build the unsigned skeleton.** Populate every field the manifest will carry *except* `manifest_id`, `signature`, and `timestamp`; set those three to `null`.
2. **Derive `manifest_id` (§3.2).** Canonicalize the skeleton (all three fields `null`), SHA-256 it, base32-encode (RFC 4648, uppercase, no padding), take the first 26 characters, and set `manifest_id` to `urn:otp:<that value>`.
3. **Sign (§6.2).** With `manifest_id` populated and `signature`/`timestamp` set to `null`, canonicalize, sign, and construct the `signature` object.
4. **Timestamp (§7).** With `manifest_id` and `signature` populated and `timestamp` still `null`, canonicalize and SHA-256 to obtain `anchored_hash`, submit it to OpenTimestamps, and construct the `timestamp` object.

Fields populated (P) versus `null` (∅) in the bytes hashed at each step:

| Computation | `manifest_id` | `signature` | `timestamp` |
|---|:---:|:---:|:---:|
| `manifest_id` derivation (step 2) | ∅ | ∅ | ∅ |
| signature (step 3) | P | ∅ | ∅ |
| `anchored_hash` (step 4) | P | P | ∅ |

Verification reverses these: recompute `manifest_id` with all three fields `null`; verify the signature with `signature` and `timestamp` `null` and `manifest_id` populated (§6.3); reconstruct `anchored_hash` with only `timestamp` `null` (§7.2).

## 7. Timestamp anchoring

The `timestamp` object records a **timestamp anchor**: a proof that the manifest existed by a certain time, committed to a public ledger. The anchor is backend-selectable via `timestamp.algorithm`; this version defines exactly one backend, `"opentimestamps"`, which anchors SHA-256 hashes to the Bitcoin blockchain via OpenTimestamps aggregation servers. (Alternative backends — additional chains, multi-anchor manifests — are noted as future work in §10; they would add algorithms without changing the reconciliation requirement below.) The prose below says "Bitcoin block" where it reports a concrete OpenTimestamps attestation; the general concept is "anchor."

### 7.1 Timestamp computation

To timestamp a signed manifest:

1. Take the manifest with `signature` populated and `timestamp` set to `null`.
2. Compute the canonical serialization per §6.1.
3. Compute SHA-256 of those bytes; this is the `anchored_hash`.
4. Submit `anchored_hash` to an OpenTimestamps aggregator (e.g., `https://alice.btc.calendar.opentimestamps.org`).
5. Receive an unupgraded `.ots` proof (pending — not yet anchored in a block).
6. Base64-encode the `.ots` proof and populate `timestamp.ots_proof`.
7. Populate `timestamp.anchored_hash` with the hash from step 3.
8. After ~1 hour (approximately 6 Bitcoin blocks), fetch an upgraded `.ots` proof and replace `timestamp.ots_proof`. The upgraded proof includes the Bitcoin block that anchors the commitment.

The manifest is now fully signed and timestamped.

### 7.2 Timestamp verification

Verification has two parts: an **offline reconciliation** that binds the proof to *this* manifest (requires nothing but the manifest), and an optional **anchor confirmation** against the ledger that establishes the actual time.

**Offline reconciliation (normative — a verifier MUST perform all of these):**

1. Reconstruct the `anchored_hash` per §7.1 steps 1-3.
2. Confirm it equals `timestamp.anchored_hash`. If not, the timestamp is invalid.
3. Decode the base64 proof from `timestamp.ots_proof` and parse it. If it does not decode or parse as a valid OpenTimestamps proof, the timestamp is **invalid**.
4. **Confirm the proof commits to the reconstructed `anchored_hash`** — that is, the digest the proof timestamps MUST equal the `anchored_hash` from step 1. A proof that commits to any other digest does not attest *this* manifest, and the timestamp is invalid regardless of whether that other proof is itself validly anchored. *(This is the reconciliation that closes the "attach an unrelated but validly-anchored proof" hole. It is new and normative in v0.2.)*
5. Read the proof's anchor state:
   - **pending** — the proof carries only a calendar commitment; no ledger block yet. Run the upgrade step (§7.1 step 8) later.
   - **upgraded** — the proof carries a concrete ledger attestation (for the `opentimestamps` backend, a Bitcoin block-header attestation naming a block height).

A manifest whose timestamp fails steps 2–4 MUST be treated as having an **invalid** timestamp — a conformant verifier MUST NOT report such a manifest as validly timestamped.

**Anchor confirmation (establishes the time; requires ledger data):**

6. To learn the actual anchor time, verify the upgraded proof against a Bitcoin full node **or** a trusted Bitcoin block-header source: confirm the block at the attested height carries the merkle root the proof commits to. This yields the block height and time.
7. The manifest is then proven to have existed at or before that block time.

A verifier that performs steps 1–5 but not 6–7 has established that the proof authentically commits to this manifest and names an anchor, but has **not** confirmed the anchor against the ledger; it MUST report this state honestly (e.g. "anchor attested, not confirmed against block headers") and MUST NOT claim a confirmed time. Steps 6–7 need a full node or a *named* trusted header source — a verifier MUST NOT silently substitute one.

## 8. Trust model — what is proven vs trusted vs out of scope

**This section is normative.** Any conformant use of the spec MUST NOT claim guarantees beyond what this section defines.

### 8.1 Proven cryptographically

Given a valid manifest, a verifier can prove:

1. **The corpus content matches the manifest.** Recomputing `corpus_hash` from the actual corpus files yields the same hash.
2. **The manifest was signed by the holder of the publisher's private key.** The signature verifies against `publisher.public_key`.
3. **The manifest existed by a specific anchor time.** The timestamp proof commits *this* manifest's `anchored_hash` to a public ledger. Offline reconciliation (§7.2 steps 1–5) proves the proof authentically attests this manifest and names its anchor; confirming the anchor against Bitcoin block headers (§7.2 steps 6–7) establishes the specific block time. The time guarantee is only as strong as that confirmation — an unconfirmed anchor names a block but does not yet prove the time.

These three facts are cryptographic. They can be re-verified by anyone, at any time, without trusting the publisher.

### 8.2 Trusted socially (publisher's word)

The following are publisher assertions that a verifier must evaluate based on the publisher's reputation, not the manifest's cryptographic properties:

- **License accuracy.** The manifest says the corpus is CC-BY-4.0; whether that's true depends on the publisher's honesty and legal analysis.
- **Contributor consent.** The `consent_statement` is a publisher claim. There is no cryptographic way to prove consent.
- **Jurisdiction.** The publisher's claim about where they produced the corpus is not verifiable from the manifest alone.
- **Description accuracy.** Whether the corpus actually contains what `description` says.
- **Contributor identity.** The listed contributors are publisher claims unless each contributor also signs.
- **Any claim about training use.** If the publisher says "used to train model X," this is a claim, not a proof.

### 8.3 Out of scope for this spec

The following problems are **not addressed** by this spec, and any implementation MUST NOT claim otherwise:

- **Proof that a specific model was trained on the corpus.** No known cryptographic primitive supports this at scale in 2026. See [related research on cryptographic training provenance](https://github.com/open-training-provenance/spec/tree/main/docs) for the current state of the art (ZKML, Proof-of-Learning, TOPLOC, TEE-based attestation).
- **License enforcement.** The spec records license claims but does not enforce them. Downstream misuse remains a legal question.
- **Content moderation.** The spec does not evaluate whether the corpus contains illegal or harmful material.
- **Deduplication or quality assessment.** These are curation questions layered on top of provenance.
- **Attribution to individual contributors within the corpus** (per-example provenance). Future spec versions may address this via nested manifests.

### 8.4 Failure modes

The following legitimate criticisms of this spec are acknowledged:

- **Publisher key compromise.** If a publisher's private key is stolen, an attacker can produce manifests indistinguishable from legitimate ones. Key rotation is not addressed in this version.
- **Sybil publishers.** Anyone can generate an Ed25519 key. Publisher reputation is out of scope for the cryptographic layer.
- **False description.** A publisher can honestly commit to a corpus and lie about what's in it. Only third-party inspection of the corpus can reveal this.
- **Timestamp aggregator compromise.** OpenTimestamps aggregators can (in principle) delay or refuse to include hashes. Bitcoin's write ordering cannot be manipulated by aggregators. If an aggregator is unavailable, the publisher can use a different one.

These failure modes are documented rather than solved. Users of the spec should design their systems with these limitations in mind.

## 9. Verification procedure (canonical)

The full procedure a verifier follows for a manifest:

1. **Parse.** Confirm the manifest is valid JSON and conforms to the schema in [`schema/manifest.v0.1.schema.json`](./schema/manifest.v0.1.schema.json).
2. **Fetch the corpus** via `links.corpus_url` or `links.corpus_mirror_urls`.
3. **Recompute `corpus_hash`** per §5. Confirm it equals `corpus.corpus_hash`.
4. **Verify the signature** per §6.3.
5. **Verify the timestamp** per §7.2: perform the offline reconciliation (§7.2 steps 1–5, including confirming the proof commits to the reconstructed `anchored_hash`), then, if a full node or named header source is available, confirm the anchor (§7.2 steps 6–7) and record the block time as the "proven by" time. A manifest with an **invalid** timestamp (§7.2) is not validly timestamped and MUST NOT be reported as such.
6. **Inspect the claims.** Read `claims.license`, `claims.consent_statement`, `claims.description`. These are publisher assertions; evaluate accordingly.
7. **Optionally cross-check the publisher.** The publisher's `public_key` and `contact` can be validated against external sources (GitHub key, DNS TXT record, DID resolver, etc.).

A manifest that passes steps 1-5 is **cryptographically valid** (its corpus, signature, and timestamp authentically attest one another) — though if its anchor is unconfirmed, the *time* is named but not yet ledger-confirmed. A manifest whose claims a verifier chooses to trust based on steps 6-7 is **socially trusted**. These are distinct properties.

## 10. Open questions

Feedback especially welcome on:

- **Publisher key rotation.** How should a publisher migrate to a new key while preserving continuity of prior manifests?
- **Nested corpora.** A manifest of manifests (dataset composed of other datasets). Merkle-tree structure? Explicit reference list?
- **Multi-signer manifests.** How do multiple contributors co-sign a manifest?
- **Per-example attribution.** For a corpus with many contributors, how to record per-example provenance efficiently?
- **Integration with C2PA.** For downstream signed model outputs, what's the natural bridge?
- **Alternative anchoring backends.** Ethereum, Solana, Arweave AO for lower-cost timestamping. Multi-anchor manifests.

Open a Discussion or issue on the [spec repository](https://github.com/open-training-provenance/spec).

## 11. Versioning

Two version numbers are tracked separately:

- The **spec-document version** (this document's header) advances whenever the specification text changes — including new verification requirements or clarifications that do not alter the manifest byte layout. It follows semver: **patch** = editorial only; **minor** = backward-compatible additions, new algorithms via feature negotiation, or newly-required verification obligations that every honest manifest already satisfies; **major** = backward-incompatible changes.
- The **wire-format version** (`spec_version` inside each manifest) changes only when the manifest byte layout or parser changes, so clients can select the appropriate parser. It is `"0.1"` and is unchanged in this v0.2 document.

**v0.2** is a minor document revision over the unchanged 0.1 wire format. Its one behavior-affecting normative addition — the §7.2 `anchored_hash` reconciliation — only ever *rejects* manifests whose proof does not attest them; every honest v0.1 manifest remains valid. Prior manifest versions remain valid indefinitely. All changes are recorded in [CHANGELOG.md](./CHANGELOG.md), which separates normative from editorial changes.

## 12. References

- Merkle, R. C. (1979). "Secrecy, Authentication, and Public Key Systems." Ph.D. Thesis, Stanford.
- Haber, S., Stornetta, W. S. (1991). "How to time-stamp a digital document." Journal of Cryptology, 3(2), 99-111.
- Bernstein, D. J., et al. (2011). "High-speed high-security signatures." CHES 2011.
- Todd, P. (2016). "OpenTimestamps." https://opentimestamps.org/
- Nakamoto, S. (2008). "Bitcoin: A Peer-to-Peer Electronic Cash System." https://bitcoin.org/bitcoin.pdf
- RFC 8785. JSON Canonicalization Scheme (JCS).
- RFC 8259. The JavaScript Object Notation (JSON) Data Interchange Format.
- SPDX License List. https://spdx.org/licenses/

## Appendix A — Rationale for design choices

**Why Ed25519?** Fast, small keys (32 bytes), small signatures (64 bytes), no parameter choices to get wrong, well-audited implementations available in every major language.

**Why OpenTimestamps?** Free, permissionless, anchored to Bitcoin (the most secure timestamping backbone available), no operator to pay or trust beyond the Bitcoin network itself.

**Why sorted-file-list SHA-256 instead of Merkle tree?** Simplicity. This version optimizes for inspectability and dependency-free verification. A future version will add Merkle-tree hashing for partial-verification workflows.

**Why not IPFS CIDs?** IPFS CIDs are excellent but introduce a dependency on the IPFS protocol version. This spec keeps the hash format self-contained. A future version may add an optional `corpus_cid` field for IPFS interoperability.

**Why JSON instead of CBOR or Protobuf?** Manifests are meant to be human-inspectable at any point in their lifetime. A curious researcher in 2050 should be able to open a manifest with `cat` and understand it. JSON's inefficiency is a small cost for permanent readability.

**Why not solve the training-attribution problem?** Because it isn't solved in 2026. Pretending otherwise would poison the spec's credibility. Being explicit about the limit invites the community to build the missing layers on top rather than assuming they exist.
