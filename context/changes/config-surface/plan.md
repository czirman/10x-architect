# Config-write guard registry de-duplication — Implementation Plan

## Overview

The two config-write REST handlers — `updateConfig` (full replace) and `patchConfig` (sparse patch) — each inline their **own hand-copied list of "cannot be changed via API" protected fields**. This duplication is the mechanism behind a real shipped defect (MM-68976: `PluginSettings.SignaturePublicKeyFiles` was guarded in `update` but not `patch` until 2026-06). This plan removes the duplication by introducing a **single source-of-truth registry of protected fields** in `api4/config.go`, guarded by a **table-driven parity test** that fails if either handler ever stops covering a registry field. It is a **pure refactor**: each endpoint's current enforcement behavior (update silently coerces, patch 403-rejects) is preserved exactly.

## Current State Analysis

Both handlers live in `server/channels/api4/config.go`:

- `updateConfig` — `:120-251`. After a permission-filtered merge, it **re-applies** protected fields from the live config, **silently coercing** the incoming value back:
  - `PluginSettings.EnableUploads` — `:155`
  - `PluginSettings.SignaturePublicKeyFiles` — `:160`
  - `ImportSettings.Directory` — `:163`
  - `PluginSettings.MarketplaceURL` (conditional on EnableUploads) — `:167`
  - `ComplianceSettings.Directory` (cloud only) — `:174` → 403 (`ComplianceSettings.Directory` is the one field `update` also rejects rather than coerces)
- `patchConfig` — `:279-406`. The same protected set, but mostly **403-rejects** on change:
  - `EnableUploads` — `:307` (reject)
  - `SignaturePublicKeyFiles` — `:314` (coerce — same as update)
  - `ImportSettings.Directory` — `:318` (reject)
  - `MarketplaceURL` — `:326` (reject)
  - `ComplianceSettings.Directory` (cloud) — `:334` (reject)

There is **no shared wrapper** — the "which fields are protected" knowledge is hand-maintained in two places. The drift between them is exactly the class of bug MM-68976 was.

**Verified state at `ee04f28` (this plan's baseline):** `patchConfig` **already guards all 5 fields** — the MM-68976 gap is closed (`c45a675553`, 2026-06-15). So this refactor is **not** fixing an open security hole; it is removing the ongoing *drift hazard* (every future protected field must still be added to two lists correctly) and closing the **test-parity gap (N4)**: `TestUpdateConfig` has a `ComplianceSettings.Directory` cloud subtest (`config_test.go:292`), `TestPatchConfig` (`config_test.go:728`) has none.

The local-mode handlers (`config_local.go` `localUpdateConfig`/`localPatchConfig`) deliberately **do not** carry these guards (privileged admin socket) — confirmed by inspection — so the registry is scoped to the two `api4` REST handlers only.

## Desired End State

- A single package-level registry in `api4/config.go` names the protected config fields once.
- Both `updateConfig` and `patchConfig` reference that registry as the authoritative "which fields" source; each keeps its own enforcement loop (coerce vs reject) unchanged.
- A table-driven parity test iterates the registry and asserts **both** handlers guard **every** field — so adding a field to the registry without wiring both handlers fails CI.
- `TestPatchConfig` gains the cloud/compliance parity subtest it was missing (N4), and the 3 divergent fields have their coerce-vs-403 behavior pinned in both handlers.
- **No observable API change**: every existing behavior (silent-coerce in update, 403 in patch) is byte-for-byte preserved.

**Verification:** `go test ./server/channels/api4/ -run 'TestUpdateConfig|TestPatchConfig|TestConfigGuard'` passes against a test DB; `golangci-lint` clean; a deliberate omission (drop a field from one handler) makes the parity test fail.

### Key Discoveries:

- Both handlers and every guard line verified exact: `updateConfig` `api4/config.go:120-251`, `patchConfig` `:279-406`; guard fields at `:155/160/163/167/174` and `:307/314/318/326/334`.
- Enforcement genuinely diverges by design: `update` coerces (`*cfg.X = *appCfg.X`), `patch` rejects (`NewAppError(..., http.StatusForbidden)`) for `EnableUploads`/`ImportSettings.Directory`/`MarketplaceURL`; `SignaturePublicKeyFiles` coerces in both; `ComplianceSettings.Directory` is cloud-gated in both.
- Cloud subtest idiom to copy: `th.App.Srv().SetLicense(model.NewTestLicense("cloud"))` + `defer RemoveLicense()`, then `CheckForbiddenStatus(t, resp)` — `config_test.go:292-305`.
- `config_test.go` (api4) is 1066 lines and tests by API, not by internal layout — it survives the extraction unchanged.
- Existing package-global pattern to mirror for placement: `writeFilter`/`readFilter` at `api4/config.go:22-23`.

## What We're NOT Doing

- **Not** unifying the enforcement style (silent-coerce vs 403-reject). That is an API-contract decision for product/API owners, not a refactor — explicitly out of scope.
- **Not** collapsing `updateConfig` and `patchConfig` into one handler. Full-replace vs sparse-patch is a deliberate two-endpoint design.
- **Not** touching the local-mode handlers (`config_local.go`) — they intentionally bypass these guards.
- **Not** moving the registry into the `model` package or making it cross-layer reusable — only the two api4 handlers use it.
- **Not** addressing any other research item (C2 schema single-source, N2 MySQL CI, emitter concurrency, etc.).

## Implementation Approach

Test-first, in two phases. Phase 1 pins the **current** contract of both handlers (the research's stated first prerequisite) so the Phase 2 extraction cannot silently change either endpoint's behavior. Phase 2 introduces the registry as the single "which fields" source and adds the parity test that makes future drift a CI failure, while leaving each handler's enforcement loop in place. Because enforcement is unchanged, the registry is realized as a **declarative list of field identities** that both handlers consult and the parity test iterates — not a generic apply engine (the fields are heterogeneous: pointer-bool, nested string, conditional).

## Critical Implementation Details

- **Enforcement divergence is intentional and must survive.** The extraction shares only *which* fields are protected, never *how* each handler responds. A reviewer diffing handler behavior before/after must see zero change. The parity test asserts *coverage* (both handlers touch each field), not *identical response*.
- **Ordering within `updateConfig` matters.** The guard re-application runs **after** the permission-filtered `writeFilter` merge (`:144-152`) and **before** `IsValid()` (`:191`). The registry wiring must not move guard application outside that window, or a filtered-in value could reach validation/persist.

## Phase 1: Characterization tests (the gate)

### Overview

Lock the current, verified behavior of both handlers before any code moves. Adds the missing `patchConfig` cloud/compliance parity subtest (N4) and pins the coerce-vs-403 behavior of the 3 divergent fields in both handlers. Independently valuable even if Phase 2 never lands.

### Changes Required:

#### 1. `patchConfig` cloud/compliance parity subtest (N4)

**File**: `server/channels/api4/config_test.go`

**Intent**: Give `TestPatchConfig` the `ComplianceSettings.Directory`-in-cloud guard subtest that `TestUpdateConfig` already has, so the two endpoints' cloud guard is symmetrically tested.

**Contract**: New `t.Run("Should not be able to modify ComplianceSettings.Directory in cloud", ...)` inside `TestPatchConfig` (`config_test.go:728`). Mirror the update-side idiom: `th.App.Srv().SetLicense(model.NewTestLicense("cloud"))` + `defer RemoveLicense()`; attempt `th.SystemAdminClient.PatchConfig` with a changed `ComplianceSettings.Directory`; assert `CheckForbiddenStatus`.

#### 2. Divergent-field characterization tests

**File**: `server/channels/api4/config_test.go`

**Intent**: Pin the *current* observable contract for the 3 fields whose enforcement differs between endpoints, so a shared registry cannot silently flip one.

**Contract**: For `EnableUploads`, `ImportSettings.Directory`, and `MarketplaceURL`, add/confirm assertions that **`updateConfig` silently coerces** (request succeeds, value reverts to prior — `assert.Equal(old, result)`) and **`patchConfig` 403-rejects** (`CheckForbiddenStatus`). Reuse existing subtests where they already cover a case; add only the missing direction. No production code changes in this phase.

### Success Criteria:

#### Automated Verification:

- Targeted tests pass: `go test ./server/channels/api4/ -run 'TestUpdateConfig|TestPatchConfig'` (against a Postgres test DB per `main_test.go`)
- Linting passes: `golangci-lint run ./channels/api4/...` (from `server/`)
- New `patchConfig` cloud subtest is present and green

#### Manual Verification:

- Reviewer confirms the new/changed tests assert the *current* behavior (coerce in update, 403 in patch) and would fail if that behavior changed
- No production (`.go` non-test) files modified in this phase

**Implementation Note**: After Phase 1 automated verification passes, pause for human confirmation that the characterization tests faithfully pin current behavior before starting Phase 2.

---

## Phase 2: Extract the guard registry + drift guard

### Overview

Introduce the single source-of-truth registry of protected fields, wire both handlers to reference it, and add the parity test that turns future drift into a CI failure. Enforcement loops stay where they are; only the "which fields" knowledge is unified.

### Changes Required:

#### 1. Protected-field registry

**File**: `server/channels/api4/config.go`

**Intent**: Declare, once, the set of config fields that are protected from API modification, so there is a single authoritative list both handlers and the parity test read.

**Contract**: A package-level declaration placed alongside `writeFilter`/`readFilter` (`:22-23`). A slice of field descriptors keyed by dotted config path (e.g. `"PluginSettings.EnableUploads"`, `"PluginSettings.SignaturePublicKeyFiles"`, `"ImportSettings.Directory"`, `"PluginSettings.MarketplaceURL"`, `"ComplianceSettings.Directory"`), each carrying enough metadata for the parity test to exercise it (path + a mutator that sets a differing value on a `*model.Config`). Cloud-gated entries flagged so the test applies a cloud license.

#### 2. Wire both handlers to the registry

**File**: `server/channels/api4/config.go`

**Intent**: Make each handler's guard block reference the registry as its "which fields" source rather than an implicit inline list, without changing enforcement.

**Contract**: `updateConfig` (`:154-178`) and `patchConfig` (`:306-334`) keep their per-field enforcement (coerce vs reject) but derive the *set* of guarded fields from the registry — so a field cannot exist in the registry yet be silently unguarded in a handler. Keep guard application inside the existing window (after `writeFilter` merge, before `IsValid`). No change to error ids, status codes, or coercion semantics.

#### 3. Registry parity / drift-guard test

**File**: `server/channels/api4/config_test.go`

**Intent**: Fail CI if either handler ever stops guarding a registry field — the standing drift hazard that produced MM-68976.

**Contract**: New `TestConfigGuardRegistryParity` (name illustrative) that iterates the registry and, for each field, drives both `UpdateConfig` and `PatchConfig` with a mutated value and asserts the field is protected (value unchanged for coerce-fields, or 403 for reject-fields, per the field's registry metadata). Applies a cloud license for cloud-gated entries. Adding a field to the registry without wiring both handlers makes this test fail.

### Success Criteria:

#### Automated Verification:

- All targeted tests pass: `go test ./server/channels/api4/ -run 'TestUpdateConfig|TestPatchConfig|TestConfigGuard'`
- Linting passes: `golangci-lint run ./channels/api4/...`
- Type/build check: `go build ./channels/api4/...`
- **Negative control**: temporarily dropping one field from one handler's guard wiring makes `TestConfigGuardRegistryParity` fail (then reverted)

#### Manual Verification:

- Reviewer diffs `updateConfig`/`patchConfig` behavior and confirms **zero** observable change (same coercion, same 403s, same error ids/status codes)
- Registry is the only place the protected-field set is declared; no residual second list
- Guard application still occurs after the permission merge and before `IsValid`

**Implementation Note**: After Phase 2 automated verification passes, pause for human confirmation (especially the behavior-parity diff and the negative-control demonstration) before considering the change complete.

---

## Testing Strategy

### Unit / handler Tests:

- `TestUpdateConfig`, `TestPatchConfig` — existing + the Phase 1 additions (N4 cloud subtest, 3 divergent-field characterizations).
- `TestConfigGuardRegistryParity` — Phase 2 table-driven drift guard over the registry.

### Key edge cases:

- Cloud-licensed vs non-cloud for `ComplianceSettings.Directory`.
- `MarketplaceURL` guard is conditional on `EnableUploads` — the registry entry/test must reflect that condition.
- `SignaturePublicKeyFiles` coerces in *both* handlers (not a reject) — the registry metadata must not mislabel it.

### Manual Testing Steps:

1. Run the full api4 config suite against a Postgres test DB; confirm green.
2. Run the negative control (drop a field from one handler) and confirm the parity test fails.
3. Diff handler behavior before/after via the characterization tests to confirm no contract change.

## Performance Considerations

None. This is request-path code that runs once per config write; the registry is a static slice iterated over ~5 fields. No measurable impact.

## Migration Notes

None. No schema, config-format, or persisted-data changes — this is internal handler refactoring with no wire-format or DB impact.

## References

- Research (verified): `context/changes/config-surface/research.md` — Refactor opportunity #1 (C3); `## Claim verification (ast-grep)` for exact line numbers.
- Handlers: `server/channels/api4/config.go:120-251` (`updateConfig`), `:279-406` (`patchConfig`), guard globals `:22-23`.
- Tests: `server/channels/api4/config_test.go:292` (update cloud subtest to mirror), `:728` (`TestPatchConfig`).
- Prior art for the drift class: MM-68976 (`c45a675553`), MM-66789 (`84e267e9e8`).

## Progress

> Convention: `- [ ]` pending, `- [x]` done. Append ` — <commit sha>` when a step lands. Do not rename step titles. See `references/progress-format.md`.

### Phase 1: Characterization tests (the gate)

#### Automated

- [ ] 1.1 Targeted tests pass: `go test ./server/channels/api4/ -run 'TestUpdateConfig|TestPatchConfig'`
- [ ] 1.2 Linting passes: `golangci-lint run ./channels/api4/...`
- [ ] 1.3 New `patchConfig` cloud subtest is present and green

#### Manual

- [ ] 1.4 Reviewer confirms tests pin current behavior (coerce in update, 403 in patch) and would fail on change
- [ ] 1.5 No production (non-test) files modified in this phase

### Phase 2: Extract the guard registry + drift guard

#### Automated

- [ ] 2.1 All targeted tests pass: `go test ./server/channels/api4/ -run 'TestUpdateConfig|TestPatchConfig|TestConfigGuard'`
- [ ] 2.2 Linting passes: `golangci-lint run ./channels/api4/...`
- [ ] 2.3 Build check: `go build ./channels/api4/...`
- [ ] 2.4 Negative control: dropping a field from one handler makes the parity test fail (then reverted)

#### Manual

- [ ] 2.5 Reviewer confirms zero observable behavior change (coercion, 403s, error ids/status codes)
- [ ] 2.6 Registry is the sole declaration of the protected-field set; no residual second list
- [ ] 2.7 Guard application still runs after the permission merge and before `IsValid`
