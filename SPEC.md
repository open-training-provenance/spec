# OpenTrainingProvenance Manifest Specification

**Version:** 0.1 (draft)
**Status:** Public draft, open for comment
**Editor:** _(publisher name here)_
**License:** MIT

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

**Corpus hash.** The Merkle-style content hash of the corpus, computed as defined in §5.

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

String. The spec version this manifest conforms to. For manifests conforming to this document, always `"0.1"`.

### 3.2 `manifest_id`

String. A URN of the form `urn:otp:<base32>` where `<base32>` is the first 26 characters of the base32-encoded SHA-256 hash of the canonical manifest bytes (see §6.1) with the `signature` and `timestamp` fields set to `null`. Deterministic — the same manifest content always produces the same ID.

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

To compute the corpus hash:

1. Enumerate every file in the corpus as a relative path from the corpus root. Paths use forward slashes and are normalized (no `..`, no leading `/`, Unicode NFC).
2. For each file, compute its SHA-256 hash.
3. Build a list of tab-separated `<hex-sha256>\t<relative-path>\n` records, one per line, sorted lexicographically by relative path.
4. Compute SHA-256 over the concatenated bytes of that list (no trailing newline beyond the final record).

The resulting hash is the `corpus_hash`.

This procedure is deliberately simple. It commits to the entire corpus content and layout. A verifier who has the corpus can independently reproduce the hash. A future spec version will add a Merkle-tree variant supporting proofs of individual file membership without downloading the whole corpus.

**Rationale:** avoiding IPFS CIDs, tarball hashing, and full Merkle trees in v0.1 keeps the primitive dependency-free and inspectable with a shell one-liner.

## 6. Canonicalization and signing

Manifests are signed and hashed over a **canonical** JSON serialization defined here.

### 6.1 Canonical serialization

The canonical form of a manifest is derived by:

1. Taking the manifest as a JSON object.
2. Recursively sorting all object keys in lexicographic (byte-wise) order.
3. Serializing with:
   - No whitespace between tokens
   - UTF-8 encoding
   - Numbers in shortest-form representation with no exponent when possible
   - Strings escaped per RFC 8259 with minimal escaping (only `\"`, `\\`, `\/` optional, control characters `\uXXXX`)
   - `null` for absent optional fields (as opposed to omission)

This is equivalent to RFC 8785 (JSON Canonicalization Scheme) for the subset of JSON this spec uses.

### 6.2 Signature computation

To compute the manifest signature:

1. Take the manifest.
2. Set `signature.value` to `null` and `timestamp` (entire object) to `null`.
3. Compute the canonical serialization per §6.1.
4. Sign the resulting bytes with the publisher's Ed25519 private key.
5. Place the resulting signature (base64) in `signature.value`.

### 6.3 Signature verification

To verify:

1. Take the manifest as received.
2. Set `signature.value` to `null` and `timestamp` (entire object) to `null`.
3. Compute the canonical serialization.
4. Verify the Ed25519 signature (`signature.value` from the received manifest) against the canonical bytes using the publisher's public key from `publisher.public_key`.

## 7. Timestamp anchoring

Timestamp proofs use **OpenTimestamps**, which anchors SHA-256 hashes to the Bitcoin blockchain via aggregation servers.

### 7.1 Timestamp computation

To timestamp a signed manifest:

1. Take the manifest with `signature` populated and `timestamp` set to `null`.
2. Compute the canonical serialization per §6.1.
3. Compute SHA-256 of those bytes; this is the `anchored_hash`.
4. Submit `anchored_hash` to an OpenTimestamps aggregator (e.g., `https://alice.btc.calendar.opentimestamps.org`).
5. Receive an unupgraded `.ots` proof (pending Bitcoin confirmation).
6. Base64-encode the `.ots` proof and populate `timestamp.ots_proof`.
7. Populate `timestamp.anchored_hash` with the hash from step 3.
8. After ~1 hour (approximately 6 Bitcoin blocks), fetch an upgraded `.ots` proof and replace `timestamp.ots_proof`. The upgraded proof includes the Bitcoin block that anchors the commitment.

The manifest is now fully signed and timestamped.

### 7.2 Timestamp verification

To verify:

1. Reconstruct the `anchored_hash` per §7.1 steps 1-3.
2. Confirm it matches `timestamp.anchored_hash`.
3. Decode the base64 OTS proof from `timestamp.ots_proof`.
4. Verify the OTS proof against a Bitcoin full node (or a trusted Bitcoin block header source). This yields a Bitcoin block height and timestamp.
5. The manifest is proven to have existed at or before that Bitcoin block time.

## 8. Trust model — what is proven vs trusted vs out of scope

**This section is normative.** Any conformant use of the spec MUST NOT claim guarantees beyond what this section defines.

### 8.1 Proven cryptographically

Given a valid manifest, a verifier can prove:

1. **The corpus content matches the manifest.** Recomputing `corpus_hash` from the actual corpus files yields the same hash.
2. **The manifest was signed by the holder of the publisher's private key.** The signature verifies against `publisher.public_key`.
3. **The manifest existed by a specific Bitcoin block time.** The OpenTimestamps proof anchors the manifest hash to Bitcoin.

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

- **Publisher key compromise.** If a publisher's private key is stolen, an attacker can produce manifests indistinguishable from legitimate ones. Key rotation is not addressed in v0.1.
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
5. **Verify the timestamp proof** per §7.2. Record the Bitcoin block time as the "proven by" time.
6. **Inspect the claims.** Read `claims.license`, `claims.consent_statement`, `claims.description`. These are publisher assertions; evaluate accordingly.
7. **Optionally cross-check the publisher.** The publisher's `public_key` and `contact` can be validated against external sources (GitHub key, DNS TXT record, DID resolver, etc.).

A manifest that passes steps 1-5 is **cryptographically valid**. A manifest whose claims a verifier chooses to trust based on step 6-7 is **socially trusted**. These are distinct properties.

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

This spec follows semver:

- **Patch** (0.1.x): editorial clarifications only. No wire-format changes.
- **Minor** (0.x): backward-compatible additions (new optional fields, new algorithms via feature negotiation).
- **Major** (x.0): backward-incompatible changes. Prior manifest versions remain valid indefinitely.

Manifests carry `spec_version` to allow clients to select the appropriate parser.

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

**Why sorted-file-list SHA-256 instead of Merkle tree?** Simplicity. v0.1 optimizes for inspectability and dependency-free verification. v0.2 will add Merkle-tree hashing for partial-verification workflows.

**Why not IPFS CIDs?** IPFS CIDs are excellent but introduce a dependency on the IPFS protocol version. This spec keeps the hash format self-contained. A future version may add an optional `corpus_cid` field for IPFS interoperability.

**Why JSON instead of CBOR or Protobuf?** Manifests are meant to be human-inspectable at any point in their lifetime. A curious researcher in 2050 should be able to open a manifest with `cat` and understand it. JSON's inefficiency is a small cost for permanent readability.

**Why not solve the training-attribution problem?** Because it isn't solved in 2026. Pretending otherwise would poison the spec's credibility. Being explicit about the limit invites the community to build the missing layers on top rather than assuming they exist.
