# Example Corpus

Two tiny plain-text files used only to demonstrate how a manifest is constructed and verified against a real corpus.

**This corpus has no training utility.** It exists solely as a worked-example target for [`../manifest.example.json`](../manifest.example.json).

## Files

- `example-1.txt` — the first sample file
- `example-2.txt` — the second sample file

## Computing the corpus hash

Given the two files, the `otp-file-list-sha256-v1` corpus hash is computed as follows:

1. For each file, compute SHA-256.
2. Build a text listing, one line per file, in the form `<hex-sha256>\t<relative-path>\n`, sorted lexicographically by path.
3. SHA-256 that listing (no trailing newline beyond the final record's `\n`).

See the reference computation in [`../manifest.example.json`](../manifest.example.json). A future reference-implementation library will provide a one-line CLI for this.
