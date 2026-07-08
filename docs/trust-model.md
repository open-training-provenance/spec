# Trust Model — Extended Discussion

> This document expands on §8 of [`SPEC.md`](../SPEC.md). It's optional reading — the spec's §8 is authoritative. This is here so people evaluating the spec can understand *why* the trust boundary is drawn where it is.

## The core claim of this spec

**We claim that three specific facts about a corpus can be proven cryptographically. We claim nothing else.**

The three facts:

1. The corpus content is exactly what the manifest commits to.
2. The manifest was signed by the holder of a specific Ed25519 private key.
3. The manifest existed by a specific Bitcoin block time.

Every other statement in a manifest — license accuracy, consent basis, description accuracy, contributor identity, training use — is a **publisher assertion**, not a cryptographic proof.

This distinction is not a hedge. It is the entire innovation of the spec. Every existing on-chain AI-data project handwaves this boundary. This one draws it in bright ink.

## Why the boundary is where it is

The state of the art in 2026, drawn from [the OpenTrainingProvenance research pass](https://github.com/open-training-provenance/research):

| Layer | Achievable at scale in 2026? | Trust model |
|---|---|---|
| Content-addressed data (Merkle / SHA-256) | ✓ Yes | Cryptographic |
| Digital signatures (Ed25519) | ✓ Yes | Cryptographic |
| Timestamp anchoring (OpenTimestamps → Bitcoin) | ✓ Yes | Cryptographic |
| License claims | ✗ No cryptographic path | Social |
| Consent claims | ✗ No cryptographic path | Social |
| Jurisdiction claims | ✗ No cryptographic path | Social |
| TEE-attested training (H100 CC / TDX) | Partially — needs specific hardware | Trust silicon vendor |
| Zero-knowledge proofs of training | ✗ Millions of dollars per model at LLM scale | Trustless (in theory) |
| Interactive Proof-of-Learning (Gensyn) | Partially — narrow runtime only | Economic |
| Per-example influence attribution | ✗ Compute-prohibitive | Statistical |

The spec's cryptographic layer sits at the top row of each proven-column. The social layer sits at the "no cryptographic path" rows. The out-of-scope items sit where the math doesn't reach.

## What "socially trusted" means concretely

When a manifest says `license.spdx: CC-BY-4.0`, that string is:

- **Cryptographically bound** to the manifest by the signature.
- **Publicly visible** and can be examined by anyone.
- **Attributed to a specific key** (and, transitively, a specific publisher identity if the key is known).
- **NOT a proof** that the corpus is actually available under CC-BY-4.0.

If the publisher lied about the license, the manifest is still cryptographically valid. What the manifest gives you is **accountability**: the false claim is now permanently attributable to that specific key. If the publisher's identity is later discovered (or already known), they own the consequences.

This is the same accountability model as signed emails, notarized statements, or blockchain transactions. **The signature doesn't make the claim true. It makes the claim attributable and permanent.**

## Publisher accountability without publisher identification

An interesting property: the spec works with pseudonymous publishers. A publisher can be identified by nothing more than a public key. If they build reputation over many manifests, that reputation attaches to the key, not to a legal identity.

This mirrors how open-source contributors work today (pseudonymous GitHub handles) and how Bitcoin developers work (pseudonymous Satoshi). Reputation compounds on the key.

If a pseudonymous publisher is later revealed to be a specific human or organization, all their manifests become attributable to that entity. If they're never revealed, the reputation still stands on its own.

## Why not attempt to prove training use?

Because the cryptography doesn't exist at practical scale in 2026.

- **Zero-Knowledge proofs of training** are theoretically possible but cost millions of dollars per model at LLM scale using current SNARK systems (see zkLLM 2024). Not deployable.
- **Interactive Proof-of-Learning** (Gensyn) requires deterministic training runtime, which isn't achievable across heterogeneous GPU pools with floating-point non-associativity. Papers show adversarial spoofing is feasible.
- **Membership inference** on modern LLMs performs near-random for single-epoch pretraining (Duan et al. 2024). It works only for memorized fine-tuning data.
- **TEE-attested training** works on H100 Confidential Compute and TDX-enabled servers, but shifts trust to Intel / AMD / NVIDIA silicon vendors and requires specific hardware most publishers don't have.

If this spec claimed to prove training use, it would either be lying or it would be a narrow niche product. Instead the spec says: **"we solve the data-commitment half. The training-use half is documented as open research."**

When the missing primitives become practical — and they will, eventually — this spec is designed to compose with them. A future v0.2 or v0.3 can add optional `training_attestation` fields that carry TEE quotes or ZK proofs, without invalidating v0.1 manifests.

## What if the publisher's key is compromised?

Then the spec's guarantees for that publisher partially collapse.

- All prior manifests signed by that key remain **cryptographically valid**. They were signed by the key, whether or not the key was compromised at the time.
- The **timestamp anchor** remains valid. A prior manifest genuinely existed by its anchored Bitcoin block time.
- **New manifests signed by the compromised key** are indistinguishable from legitimate ones.

Recovery procedure (not defined in v0.1, planned for v0.2):

- Publisher issues a revocation manifest signed by both the compromised key and a new key, declaring the compromised key as of a specific time.
- Consumers of manifests should check for revocation before trusting new manifests from that key.
- Timestamp anchoring gives revocation a defined ordering.

This is standard PGP-style key management transplanted to the manifest layer.

## Why Bitcoin for anchoring specifically?

Bitcoin has the deepest, most economically secured timestamping backbone. Reorganizations beyond 6 blocks are essentially unheard of; reorganizations beyond 100 blocks would require attacker economic capacity that has never been demonstrated. For a manifest whose provenance should hold for decades, this matters.

OpenTimestamps additionally lets any number of manifests be aggregated into a single Bitcoin transaction, so the marginal on-chain cost per manifest is effectively zero.

Alternative anchoring backends (Ethereum, Solana, Arweave AO) are noted as future work. A multi-anchor manifest could commit to multiple chains for resilience.

## Why this is worth doing at all if it's "only" data commitment

Because right now, in 2026, no one can independently verify **anything** about the training corpora of open-weights models. Every claim is trust-based, with no primitive for even the parts that are provable.

Solving the provable half is a real contribution even if the unprovable half remains open. It:

- Provides an audit trail for regulatory compliance (EU AI Act Article 10).
- Enables reproducibility of open science.
- Detects benchmark contamination retrospectively.
- Attributes contribution to pseudonymous or named publishers.
- Sets up the composition point for future primitives (TEE, ZK) to plug in.

**Being honest about the boundary is a feature, not a bug.** It's what makes the primitive credible in the long run.

## Summary

Cryptographic:
- Corpus content
- Publisher identity
- Commitment time

Social:
- License, consent, jurisdiction, description accuracy
- Contributor claims
- Training-use claims

Out of scope:
- Cryptographic proof of training use
- License enforcement
- Content moderation
- Per-example attribution

If you build on the spec, make sure your users understand this taxonomy. Don't overclaim.
