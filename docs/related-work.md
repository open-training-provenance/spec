# Related work

OpenTrainingProvenance (OTP) is not the first project to propose cryptographic provenance
for AI training data. This document credits the prior and adjacent work we're aware of and
states, honestly, where OTP is the same and where it differs. If you're evaluating OTP, you
should also look at these.

## DataLedger (Eva Paunova)

**Repository:** https://github.com/epaunova/DataLedger
**Compared against:** the repository as of its latest commit on 2026-03-12 (the state at the
time of writing). DataLedger predates OTP's public v0.1 draft (2026-07-08) by roughly four
months, and OTP was developed independently — this is convergent design, not derivation.
DataLedger deserves acknowledgment as prior art in this space.

DataLedger, by Eva Paunova, is described as an "Open Standard for Verifiable AI Training
Data Provenance." It defines a **signed dataset manifest** and an **unsigned training-run
attestation**, using the same well-chosen primitives OTP independently landed on:
**Ed25519** signatures, **SHA-256** content hashing, and **RFC 8785 (JCS)** canonicalization.
That two independent efforts reached for the same primitives is, we think, a good sign that
those are the right primitives for this problem.

DataLedger also does several things well that are worth calling out:

- Its design-rationale section arguing **why not extend in-toto/SLSA or Croissant** is
  genuinely well-reasoned — in particular the observation that JSON-LD (Croissant) has no
  canonical serialization, which is why a signable format needs plain JSON + JCS.
- It is explicitly **composable with Croissant and Hugging Face dataset/model cards** —
  concrete ecosystem integration that OTP has not yet specified.
- It frames itself against **EU AI Act Annex IV** documentation requirements.

### Where OTP differs

These are real, deliberate differences in scope and design — not claims that one project is
"better." Different tools fit different jobs.

1. **Temporal anchoring.** OTP anchors each manifest to the Bitcoin blockchain via
   OpenTimestamps (SPEC §7), so a verifier can establish that the manifest *existed by a
   specific time* — and, under OTP's v0.2 §7.2 reconciliation, that the proof commits to
   *that* manifest. DataLedger's manifest is signed but not time-anchored: it proves *who*
   and *what*, and records a self-declared `created_at`, but does not provide a
   third-party-verifiable *when*. Anchoring is the axis where OTP is most distinct.

2. **What we decline to attest.** DataLedger includes a training-run attestation with a
   `proportion` field intended to record which datasets entered a training run and in what
   fraction. That value is a floating-point number, and the attestation is **unsigned and
   self-reported by the training pipeline**. OTP deliberately does *not* offer this. Our
   trust model (SPEC §8.3) marks "proof that a specific model was trained on a corpus" as
   **out of scope** — no cryptographic primitive establishes it at scale today — and OTP's
   canonical form forbids floating-point numbers entirely (SPEC §6.1.1) so that a manifest
   never carries a value implying more assurance than the math delivers. DataLedger is
   upfront that its attestation is unsigned; the difference is that OTP keeps that class of
   claim out of the signed artifact rather than adjacent to it.

3. **An explicit, load-bearing trust boundary.** OTP's SPEC §8 partitions every guarantee
   into **proven cryptographically** (corpus content, publisher identity, anchor time),
   **trusted socially** (license, consent, jurisdiction, description — the publisher's word),
   and **out of scope** (training-use proof, license enforcement, content moderation). Being
   explicit about what the primitive does *not* prove is, for OTP, the whole point. DataLedger
   has a "What DataLedger Does Not Provide" section in the same spirit; OTP simply makes this
   partition the center of its design and its public positioning.

### Summary

DataLedger and OTP are **complementary, convergent efforts** on the same problem, reaching
for the same cryptographic primitives from different angles: DataLedger leans toward
ecosystem composability (Croissant, Hugging Face) and training-run bookkeeping; OTP leans
toward time-anchoring and a strict, explicit boundary between what is proven and what is
merely asserted. If your need is dataset-metadata interoperability, look closely at
DataLedger. If your need is a time-anchored, minimally-scoped commitment with an unambiguous
trust boundary, OTP may fit better. Many real deployments could reasonably use ideas from
both.

## Foundational prior art

OTP is a composition of primitives that long predate it, and the design owes them explicitly
(see SPEC §1 and §12):

- **Merkle (1979)** — hashing file sets into a single commitment.
- **Haber & Stornetta (1991)** — timestamping documents by chaining hashes; the intellectual
  root of blockchain timestamping.
- **Bernstein et al. (2011)** — Ed25519 signatures.
- **Todd (2016), OpenTimestamps** — free, permissionless Bitcoin-anchored timestamping. OTP's
  time-anchoring layer *is* OpenTimestamps; we add nothing to it and depend on it directly.

## Adjacent standards

These solve related but distinct problems; OTP aims to be complementary, never competitive
(see the roadmap's cross-standard alignment work):

- **C2PA / Content Authenticity Initiative** — provenance for media content (images, video,
  audio), including AI-generated output. OTP covers *training data*; C2PA covers *content*.
- **IETF SCITT** — a general supply-chain attestation framework; an OTP manifest can be viewed
  as a domain-specific instance of the same idea.
- **MLCommons Croissant** — a dataset-metadata vocabulary. Not a provenance/integrity protocol,
  but composable with one (a point DataLedger makes well).
- **W3C DPV** — vocabulary for consent and legal basis, which OTP's socially-trusted layer
  could reference rather than reinvent.

---

*If you maintain a project you think belongs here — prior art we missed, or an adjacent
standard OTP should align with — please open an issue or Discussion on the spec repository.
We would rather over-credit than under-credit.*
