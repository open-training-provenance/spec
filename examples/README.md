# Worked example

This directory holds a **real, signed, Bitcoin-anchored** OpenTrainingProvenance manifest — not
a placeholder. It commits to the tiny demo corpus in [`example-corpus/`](./example-corpus/)
(three plain-text files, no training utility) and exercises every layer of the spec end to end.

## Files
- `manifest.example.json` — the manifest: content hash of the corpus, publisher key, Ed25519
  signature, and an OpenTimestamps proof anchored to **Bitcoin block 957,265**.
- `publisher-key.pub` — the publisher's Ed25519 public key (matches `publisher.public_key` in the
  manifest). **Example-use only, not a durable identity** — it signs nothing but this demo corpus.
- `example-corpus/` — the three files the manifest commits to.

## Verify it yourself

With the reference implementation ([`otp-manifests`](https://github.com/open-training-provenance/otp-manifests)):

```
otp verify examples/manifest.example.json --corpus examples/example-corpus
```

You should see the corpus hash match, the signature verify, the `manifest_id` reconcile, and the
timestamp anchor report **Bitcoin block 957,265**, committing to this exact manifest (SPEC §7.2).

The anchor is also checkable **without any OTP tooling** — that's the point of anchoring to a
public ledger. The proof commits this manifest's `anchored_hash` into block 957,265
(`00000000000000000000e434e3b18f87e274f66bb36eff429f8b0ae827c4164f`), viewable at any block
explorer, e.g. <https://mempool.space/block/957265>. OpenTimestamps' own verifier at
<https://opentimestamps.org> can independently confirm the embedded `.ots` proof.

## What this proves (and what it doesn't)

Per SPEC §8: it proves the corpus content, the signing key, and that this manifest existed by that
block's time. It does **not** prove the license/consent claims are true, or that any model trained
on the corpus. See [`../docs/trust-model.md`](../docs/trust-model.md).
