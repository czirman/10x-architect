# Config-write guard registry de-duplication — Plan Brief

> Full plan: `context/changes/config-surface/plan.md`
> Research: `context/changes/config-surface/research.md`

## What & Why

The two config-write REST handlers (`updateConfig`, `patchConfig`) each maintain their **own hand-copied list** of "cannot be changed via API" protected fields. This duplication is the drift hazard behind a real shipped defect (MM-68976: `SignaturePublicKeyFiles` was guarded in `update` but not `patch` until mid-2026). This plan removes the duplication with a **single source-of-truth registry** plus a **parity test** that fails whenever a handler stops covering a registry field — as a pure refactor that preserves each endpoint's current behavior.

## Starting Point

Both handlers live in `server/channels/api4/config.go` (`updateConfig` `:120-251`, `patchConfig` `:279-406`), each inlining the same protected set. Verified at `ee04f28`: `patchConfig` **already guards all 5 fields** (MM-68976 is closed), but the two lists are still hand-maintained, and `TestPatchConfig` lacks the cloud/compliance parity subtest `TestUpdateConfig` has (N4).

## Desired End State

The protected-field set is declared once; both handlers read it as their authoritative "which fields" source while keeping their own enforcement loops; a table-driven parity test asserts both handlers guard every registry field, so future drift is a CI failure. No observable API change.

## Key Decisions Made

| Decision | Choice | Why | Source |
| --- | --- | --- | --- |
| Which opportunity | C3 — guard registry (#1) | Lowest change cost, ranked #1, first step independently valuable | Research |
| Registry shape | Single-source the field list | Matches research's named target; no reflection/accessor machinery | Plan |
| Enforcement divergence | Keep both behaviors as-is | Silent-coerce vs 403 is an API-contract decision, out of refactor scope | Research + Plan |
| Test depth | N4 subtest + 3 divergent fields | Research's stated first prerequisite — lock contract before extraction | Research + Plan |
| Placement | Same file, package-level | Mirrors existing `writeFilter`/`readFilter` globals; local handlers confirmed not to need it | Plan |
| Urgency framing | Maintainability, not live fix | Verification: the MM-68976 security gap is already closed at `ee04f28` | Research (verification) |

## Scope

**In scope:** single protected-field registry in `api4/config.go`; both handlers wired to it; drift-guard parity test; missing `patchConfig` cloud subtest (N4); characterization tests for the 3 divergent fields.

**Out of scope:** unifying enforcement style (coerce vs 403); merging the two handlers; local-mode handlers; moving anything to `model`; every other research item (C2, N2, emitter, etc.).

## Architecture / Approach

Test-first, two phases. Phase 1 pins current behavior of both handlers so Phase 2 can't silently change either. Phase 2 adds a declarative registry (field path + a mutator + a cloud flag) that both handlers consult for *which* fields to guard and the parity test iterates for coverage. Enforcement loops stay per-handler — the registry unifies the "what," never the "how."

## Phases at a Glance

| Phase | What it delivers | Key risk |
| --- | --- | --- |
| 1. Characterization tests | N4 cloud subtest + coerce-vs-403 pins for 3 divergent fields | Mis-pinning current behavior (mislabel a coerce field as reject) |
| 2. Registry + drift guard | Single-source registry, both handlers wired, parity test | Accidentally altering an endpoint's observable contract during wiring |

**Prerequisites:** a Postgres test DB (api4 config tests are Postgres-gated per `main_test.go`); working `go`/`golangci-lint` toolchain in `mattermost/server`.
**Estimated effort:** ~1–2 sessions across 2 phases; single-file production change plus test additions.

## Open Risks & Assumptions

- Enforcement divergence (coerce vs 403) is preserved deliberately; the observable inconsistency between endpoints remains (documented, not fixed) — routing it to product/API owners is a separate decision.
- The parity test asserts *coverage* (both handlers touch each field), not identical responses, since responses legitimately differ.
- `MarketplaceURL`'s guard is conditional on `EnableUploads` and `SignaturePublicKeyFiles` coerces in *both* handlers — registry metadata must encode these correctly or the test misfires.

## Success Criteria (Summary)

- Both endpoints behave exactly as before (same coercion, same 403s, same error ids) — confirmed by the characterization tests.
- The protected-field set is declared in one place; dropping a field from one handler makes the parity test fail (negative control).
- `TestPatchConfig` now has the cloud/compliance parity subtest it was missing.
