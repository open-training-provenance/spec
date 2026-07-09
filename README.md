# OpenTrainingProvenance

An open specification for cryptographically-committed provenance manifests for AI training data.

**Status:** v0.2 draft — spec under active development. Read [`SPEC.md`](./SPEC.md) and [`CHANGELOG.md`](./CHANGELOG.md). v0.2 is a spec-document revision over the unchanged 0.1 manifest wire format (manifests still carry `spec_version: "0.1"`).

## What this is

A minimal, honest primitive for making verifiable claims about AI training datasets. A manifest cryptographically commits to:

- **What** the corpus is (content hash over the sorted file list)
- **Who** is publishing it (Ed25519 signature)
- **When** the corpus was committed (OpenTimestamps proof, Bitcoin-anchored)

It explicitly documents the boundary between what's **cryptographically proven** and what's **socially trusted** — see [`docs/trust-model.md`](./docs/trust-model.md).

## What this is not

- Not a token, not a marketplace, not a DAO.
- Not a cryptographic proof that any specific model was trained on the corpus. That's an open research problem in 2026 and no spec can promise what math cannot yet deliver.
- Not a license enforcement mechanism. The spec records license claims; enforcement remains a legal question.
- Not a curation layer. Anyone can publish a manifest; quality and trustworthiness of the underlying claims are downstream questions.

## Why this exists

AI training data provenance is broken. Every model release claims "trained on our internal corpus" or "public web data" and nobody can independently verify. As the AI industry moves toward regulatory frameworks (EU AI Act Article 10) that require training-data governance, and as decentralized training becomes real (Nous Psyche, Prime Intellect INTELLECT-1/2, Bittensor SN13), we need an open standard for making cryptographically verifiable claims about datasets — one that's honest about what it can and can't prove.

This spec inherits from the pre-blockchain provenance work — Haber & Stornetta (1991), Merkle trees (1979), Ed25519 (2011), OpenTimestamps (2016). It uses primitives that have been correct for decades. The parts it explicitly does not solve — proving a specific model descended from a specific corpus — are documented as future work, not hidden.

## Repository layout

- [`SPEC.md`](./SPEC.md) — the specification itself
- [`schema/manifest.v0.1.schema.json`](./schema/manifest.v0.1.schema.json) — JSON Schema for automated validation
- [`examples/`](./examples/) — a worked example manifest and its corpus
- [`docs/trust-model.md`](./docs/trust-model.md) — extended discussion of what's proven vs trusted
- [`docs/related-work.md`](./docs/related-work.md) — prior art (incl. DataLedger) and adjacent standards, and how OTP relates
- [`CHANGELOG.md`](./CHANGELOG.md) — spec changes by version, normative vs editorial
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — how to propose changes

## Status and roadmap

- **v0.1** — initial public draft: core manifest structure, cryptographic primitives, trust boundary.
- **v0.2 (this document)** — spec/implementation reconciliation: byte-exact corpus hash (§5), RFC 8785 canonicalization + no-floats (§6.1), the three-state computation order (§6.4), and the normative `anchored_hash` reconciliation (§7.2) that binds a timestamp proof to the manifest it attests. See [`CHANGELOG.md`](./CHANGELOG.md) for normative-vs-editorial detail.
- **Future** — Merkle-tree partial verification of subsets, publisher key rotation/revocation, multi-signature manifests, nested/derived-dataset lineage, and extension protocols (TEE attestation, ZKML, proof-of-learning) as the underlying cryptography matures. Extensions add to the core; they never replace core fields.

The spec is versioned and additive. Existing manifests remain valid indefinitely; future versions layer on top rather than replace.

## Feedback

Open a [GitHub Discussion](https://github.com/open-training-provenance/spec/discussions) or an issue. The spec is a public draft; substantive critiques on the trust model, the choice of primitives, or missing use cases are especially welcome.

## License

MIT for the spec, schema, and examples. See [LICENSE](./LICENSE).
