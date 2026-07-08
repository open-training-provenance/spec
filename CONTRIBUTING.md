# Contributing to OpenTrainingProvenance

Thank you for considering a contribution.

## Scope

This repository holds the **spec**, its JSON schema, worked examples, and companion documentation. It does not hold reference implementations, tooling, or datasets — those will live in separate repositories under the [`open-training-provenance`](https://github.com/open-training-provenance) organization when they exist.

## Kinds of contribution welcome

- **Substantive feedback on the spec.** Ambiguities, missing fields, alternative primitive choices, trust-model corrections.
- **Editorial fixes** to prose, examples, and the schema.
- **Test vectors.** Additional worked examples with real cryptographic material once the reference implementation ships.
- **Language translations** of `SPEC.md` and `README.md`.

## Kinds of contribution NOT welcome

- **Marketing.** No token discussion, no roadmap-inflation PRs, no logo work at this stage.
- **Scope creep.** Feature requests that turn the spec into a platform or a marketplace should be raised as [Discussions](https://github.com/open-training-provenance/spec/discussions), not PRs.
- **Attempts to claim provenance guarantees beyond what §8 permits.** The trust boundary is deliberate; PRs that erode it will be closed with an explanation.

## How to propose a change

For substantive changes:

1. Open a [GitHub Discussion](https://github.com/open-training-provenance/spec/discussions) describing the problem and proposed direction.
2. Wait for at least one round of comment. Substantive proposals should be publicly discussed before code is written.
3. Open a PR referencing the discussion. Keep the PR narrow — one substantive change per PR.

For editorial fixes:

1. Open a PR directly.
2. Keep the diff minimal.

## Versioning discipline

Manifests in the wild MUST remain parseable indefinitely. A change to the spec that would break existing v0.1 manifests requires a major version bump. Changes that would break the interpretation of existing manifests (e.g., re-defining what a field means) are almost never accepted.

Preferred changes are **backward-compatible additions**: new optional fields, new algorithms via feature negotiation, new documentation.

## Code of conduct

Be technical, be honest, be direct. Substantive critique is welcome; personal attack is not. This project is intended to be usable by pseudonymous contributors — treat pseudonymous handles with the same respect as named identities.

## License

By contributing, you agree that your contribution is licensed under the same MIT license as the rest of the repository.
