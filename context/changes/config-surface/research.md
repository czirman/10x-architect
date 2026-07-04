---
date: 2026-06-27T00:00:00Z
researcher: czirman (service.mak@proton.me)
git_commit: fe3886d7b7b39ade2c09fcbf9fbeb7cf84285151
branch: module-4-lesson-3
repository: shaping-claude (analysis target: nested mattermost repo @ ee04f28e873d990a1719cfc13809ad8aa7cc6554)
topic: "Configuration surface flow — e2e trace, test gaps, blast radius"
tags: [research, codebase, config, mattermost, config-surface, blast-radius, test-coverage]
status: complete
last_updated: 2026-07-04
last_updated_by: czirman
last_updated_note: "Refactor-feasibility follow-up (2026-07-04): classified every recorded problem as structural-candidate vs not, then ran a 3-lens exploration (current shape / intentionality / migration feasibility, read-only) on the structural candidates. New section `## Follow-up Research 2026-07-04 — Refactor opportunities` appended, ending with a ranked opportunity list. Prior note: ast-grep structural verification pass — corrected item #1 (local handlers mostly covered via th.LocalClient; only localGetClientConfig untested) and admin_definition import count (92, not 87)."
---

# Research: Configuration Surface Flow (Mattermost)

**Date**: 2026-06-27
**Researcher**: czirman (service.mak@proton.me)
**Git Commit (this repo)**: b6c0fa5
**Analysis target**: nested `mattermost/` repo, HEAD `ee04f28`, branch `master`
**Branch**: module-4-lesson-3
**Repository**: shaping-claude

## Research Question

Analyze the chosen flow (the **configuration surface**, rooted at `server/public/model/config.go`, per the Research Objective in `context/map/repo-map.md`), paying attention to the related areas defined in the repo map. Three parallel investigations:

1. **Trace e2e** — reconstruct the path from entry point, through the layers, to persistence and back. Step sequence with `file:line` + a Mermaid diagram.
2. **Test gaps** — which methods/branches on the path are covered vs. not.
3. **Blast radius** — what must change together when this flow changes (interface seam, generated layers, model, migrations, tests), combining the static graph with git co-change.

Describe only the **current state** of the repository. Report must contain two explicit sections: **Feature overview** and **Technical debt**. Separate **evidence** from **inference** from **unknown**.

> **Scope note.** All `file:line` references are relative to the nested `mattermost/` directory (HEAD `ee04f28`). Git co-change counts are bounded to the repo-map window **2025-06-20 → 2026-06-19** so they reconcile with `context/map/`. Some line numbers carried from sub-agent reads are approximate (`~`) where flagged.

---

## Summary

The configuration surface is a **5-layer pipe** — `api4` (REST) → `app` (facade) → `platform` (`PlatformService`, owns the store + listeners) → `config.Store` (caches `*model.Config`, applies/strips env overrides, emits to listeners) → `BackingStore {File | Database | Memory}` (the path forks here). Reads are served entirely from an in-memory cache (no backend hit); writes flow down to a backend and then **fan out synchronously through an emitter** to runtime subscribers (client-config regeneration, a WebSocket `config_changed` broadcast, logger reconfiguration, search reconfiguration), plus a cluster (HA) fan-out to peer nodes.

One finding dominates the technical-debt picture, plus a significant correction the ast-grep pass made to the prior report's second finding:

1. **The config surface is THE backend↔frontend seam.** The repo map (Risk Zone 4) called the cross-stack ripple an *inference, not a measured co-change pair.* Git history **refutes that caveat**: `config.go ↔ config.ts` co-change **34** times, `config.go ↔ admin_definition.tsx` **24**, in the window. It is measured fact and should be upgraded `[inference] → [git]`.
2. **The privileged local-mode admin path (`config_local.go`) is *mostly* covered, with one real gap.** ⚠️ *Corrected by the ast-grep pass — the original claim of "essentially untested" was wrong.* `config_test.go` exercises `localGetConfig` (3×), `localUpdateConfig` (8×), `localPatchConfig` (3×) and `localMigrateConfig` (1×) through `th.LocalClient` (which talks to the local-mode socket and therefore routes to the `APILocal(...)` handlers — `api4/apitestlib.go:237`). The genuine gap is **`localGetClientConfig` only** — zero test reach. The earlier conclusion came from grepping the handler *identifiers*, which never appear in tests because the handlers are invoked via routing, not by name. See item #1 in Technical debt and `## Structural verification (ast-grep)`.

There are **two nearly-disjoint blast radii**: a **config-field change** (big, cross-stack, git-measured) and a **config-store-interface change** (small, static-only, git-empty). They barely overlap because the store seam is **field-agnostic** — it moves the whole `*model.Config` as an opaque JSON blob. Consequently, **a config-field add needs no migration**.

---

## Feature overview

### The 5 layers (evidence)

```
api4 (REST)  →  app (App facade)  →  platform (PlatformService)  →  config.Store (commonStore + emitter)  →  BackingStore {File | Database | Memory}
```

| Layer | Files | Responsibility |
|---|---|---|
| **api4** | `server/channels/api4/config.go`, `config_local.go` | HTTP handlers, permission filtering, merge/diff, audit |
| **app** | `server/channels/app/config.go` | thin delegation facade |
| **platform** | `server/channels/app/platform/config.go`, `service.go` | owns `*config.Store`, cached client-config atomics, registers master listener, cluster fan-out |
| **config.Store** | `server/config/store.go`, `emitter.go`, `environment.go`, `diff.go`, `client.go` | caches `*model.Config`, applies/strips env overrides, persists, emits |
| **BackingStore** | `server/config/file.go`, `database.go`, `memory.go` | the three persistence backends (path forks) |
| **contract** | `server/public/model/config.go` | the schema: `SetDefaults`, `IsValid`, `Sanitize`, `Clone` (5,765 lines) |

### READ path — `GET /api/v4/config` → `getConfig` (evidence)

1. Route `GET /config` → `getConfig` — `api4/config.go:35`.
2. Permission gate `SessionHasPermissionToAny(... SysconsoleReadPermissions)` — `api4/config.go:53`.
3. `c.App.GetSanitizedConfig()` — `api4/config.go:61` → clones live config + sanitizes — `app/config.go:215-221`.
4. `App.Config()` → `ch.cfgSvc.Config()` — `app/config.go:31-33` → `ps.configStore.Get()` — `platform/config.go:40-42`.
5. `Store.Get()` returns cached `s.config` under `RLock` — **no backend hit on reads** — `config/store.go:126-130`.
6. `Config.Clone()` (marshal+unmarshal round-trip) — `model/config.go:4226-4237`.
7. `Sanitize(...)` masks secrets with `model.FakeSetting` — `app/config.go:224-234`, `model/config.go:5316+`.
8. Permission field-filter via `config.Merge(&Config{}, sanitized, readFilter)` — keeps only readable fields (per `access:` struct tag) — `api4/config.go:61-65`, filter factory `:408-465`.
9. Query post-filter `remove_masked`/`remove_defaults`/cloud-tag → `model.FilterConfig` — `api4/config.go:71-93`.
10. `Cache-Control: no-cache` + JSON-encode — `api4/config.go:75, 95-97`.

Client-config read (`GET /config/client`) is served from a **separate pre-computed atomic cache**, not the full config — `api4/config.go:253-264`, cache build `platform/config.go:220-244`.

### WRITE path — `PUT /api/v4/config` → `updateConfig` (evidence)

1. Decode body → `*model.Config` — `api4/config.go:121-126`.
2. `cfg.SetDefaults()` (accept partial body) — `api4/config.go:131`; impl `model/config.go:4276-4338`.
3. Write permission gate (`SysconsoleWritePermissions`) — `api4/config.go:133`.
4. Read live `appCfg`; refuse to clear non-empty SiteURL — `api4/config.go:138-142`.
5. Permission-filtered merge of body over existing (`writeFilter`, honors `write_restrictable`/`cloud_restrictable` + `RestrictSystemAdmin`) — `api4/config.go:144-152`.
6. Re-apply "cannot be changed via API" overrides (`PluginSettings.EnableUploads`, `SignaturePublicKeyFiles`, `ImportSettings.Directory`, marketplace URL, cloud Compliance dir) — `api4/config.go:154-178`.
7. Elasticsearch autocomplete pre-check — `:182-187`; MessageExport timestamp rewrite — `:189` (impl `app/config.go:247-261`).
8. **Validation** `cfg.IsValid()` → 400 on failure — `api4/config.go:191-194`; impl `model/config.go:4340+`.
9. **Persist** `c.App.SaveConfig(cfg, true)` — `api4/config.go:196` → `app/config.go:243-245` → `platform/config.go:91-129`:
   - plugin `ConfigurationWillBeSaved` hook may mutate/veto — `platform/config.go:92-108`.
   - `ps.configStore.Set(newCfg)` — `:113`; read-only → 403, other → 500.
   - **Cluster (HA) fan-out** `clusterIFace.ConfigChanged(removeEnvOverrides(old/new), send)` — `:120-126`.
10. **`Store.Set`** core (write lock) — `config/store.go:168-241`:
    - read-only guard → `ErrReadOnlyStore` `:172-174`; clone+snapshot `:176-178`; `SetDefaults()` again `:181`.
    - **`Desanitize(old,new)`** restores real secrets where body still carried `FakeSetting` `:185`.
    - **`applyEnvironmentMap(new, GetEnvironment())`** overlays `MM_*` env vars `:189-190`.
    - **Second `IsValid()`** inside the store `:192-194`.
    - **`removeEnvOverrides(...)`** → `newCfgNoEnv` (env values never persisted) `:197`.
    - FeatureFlags nil'd when `readOnlyFF` `:203-211`.
    - **`backingStore.Set(newCfgNoEnv)`** persist env-stripped `:213`.
    - `equal(old,new)` → `hasChanged` gates emit `:217`; update caches `:229-230`.
    - **Emitter fan-out** if changed: unlock → `invokeConfigListeners(old,newCopy)` → relock `:234-238`.
11. **Backend fork** (`backingStore.Set`):
    - **File** — refuse if `ClusterSettings.Enable && ReadOnlyConfig`; else `os.WriteFile(path, b, 0600)` — `config/file.go:104-126`.
    - **Database** — marshal+sha256; skip write if checksum unchanged; else txn `UPDATE ... Active=NULL` + `INSERT` new active row (version history; `cleanUp` prunes) — `config/database.go:156-220, 334-352`.
    - **Memory** (tests) — clone into `savedConfig` — `config/memory.go:60-69`.
12. **Emitter** `invokeConfigListeners` synchronously ranges a `sync.Map` of `Listener func(old,new *model.Config)` — `config/emitter.go:14, 33-40`.
13. **Master listener effects** (registered in `PlatformService.Start`) — `platform/service.go:468-482`:
    - `regenerateClientConfig()` rebuilds cached client/limited-client atomics — `platform/config.go:220-244`.
    - Broadcasts `WebsocketEventConfigChanged` to all clients — `service.go:471-477` (async `ps.Go(...)`).
    - `ReconfigureLogger()` — `service.go:478-481`; search-engine reconfig — `platform/searchengine.go:17`.
14. **Back out** through handler: re-init translations if locale changed `:202-209`; `config.Diff(old,new)` for audit prior-state `:211-216`; `Sanitize` + readFilter merge `:218-228`; audit success `:231-233`; JSON response `:235-250`.

`patchConfig` (`api4/config.go:279-406`) is the same shape with a partial-patch merge. `configReload` (`:100-118`) → `platform.ReloadConfig()` → `configStore.Load()` (`config/store.go:244-346`) re-reads the backend, re-applies env, re-validates, writes back if changed, and emits — same emitter path as `Set`.

### Sequence diagram (evidence-derived)

```mermaid
sequenceDiagram
    participant Client
    participant API as api4/config.go<br/>updateConfig
    participant App as app/config.go
    participant PS as platform/config.go<br/>SaveConfig
    participant Store as config.Store.Set
    participant Env as environment.go
    participant Back as BackingStore<br/>(File|DB|Memory)
    participant Emit as emitter.go
    participant Listeners as service.go listener

    Note over Client,Listeners: WRITE
    Client->>API: PUT /api/v4/config (JSON)
    API->>API: SetDefaults / perm check / writeFilter Merge
    API->>API: IsValid()  (model/config.go:4340)
    API->>App: SaveConfig(cfg, true)
    App->>PS: platform.SaveConfig
    PS->>PS: plugin ConfigurationWillBeSaved hook
    PS->>Store: configStore.Set(newCfg)
    Store->>Store: SetDefaults, Desanitize
    Store->>Env: applyEnvironmentMap (overlay MM_*)
    Store->>Store: IsValid() (2nd)
    Store->>Env: removeEnvOverrides -> newCfgNoEnv
    Store->>Back: Set(newCfgNoEnv)  [persist env-stripped]
    alt File
        Back->>Back: os.WriteFile(config.json, 0600)
    else Database
        Back->>Back: sha check; UPDATE Active=NULL; INSERT new active
    else Memory
        Back->>Back: savedConfig = clone
    end
    Store->>Store: s.config=newCfg; hasChanged?
    Store-->>Emit: invokeConfigListeners(old,new) [if changed]
    Emit->>Listeners: listener(old,new)
    Listeners->>Listeners: regenerateClientConfig()
    Listeners-->>Client: WS WebsocketEventConfigChanged
    Listeners->>Listeners: ReconfigureLogger / search reconfig
    PS->>PS: cluster.ConfigChanged(removeEnvOverrides) [HA fan-out]
    Store-->>API: oldCfg, newCfg
    API->>API: Diff(old,new)+audit; Sanitize; readFilter Merge
    API-->>Client: 200 JSON (filtered)

    Note over Client,Store: READ
    Client->>API: GET /api/v4/config
    API->>App: GetSanitizedConfig()
    App->>PS: Config()
    PS->>Store: configStore.Get()  [in-memory cache, no backend]
    Store-->>App: *model.Config (clone)
    App->>App: Sanitize() -> FakeSetting
    API->>API: readFilter Merge + FilterConfig
    API-->>Client: 200 JSON
```

### Key types / interfaces (evidence)

- `config.Store` struct (caches `config`, `configNoEnv`, embeds `emitter`, holds `backingStore`) — `config/store.go:27-38`.
- `config.BackingStore` interface (`Set/Load/GetFile/SetFile/HasFile/RemoveFile/String/Close`) — `config/store.go:42-68`.
- `Store.Set` `:168` · `Store.Get` `:126` · `Store.Load` `:244`.
- `config.Listener = func(oldCfg, newCfg *model.Config)` — `config/emitter.go:14`; `emitter{ listeners sync.Map }` `:17-40`.
- Backends: `FileStore{path}` `config/file.go:28`; `DatabaseStore{...}` `config/database.go:44`; `MemoryStore` `config/memory.go` (`Set` :60).
- App facade: `App.Config()` `app/config.go:31`; `App.SaveConfig` `:243`; `App.GetSanitizedConfig` `:215`.
- Platform: `PlatformService.configStore` accessor `GetConfigStore()` `platform/config.go:192-194`; `Config()` `:40`; `SaveConfig` `:91`.
- Model: `SetDefaults()` `model/config.go:4276`; `IsValid()` `:4340`; `Sanitize(...)` `:5316`; `Clone()` `:4226`; `model.FilterConfig` `:5492`.
- Diff: `config.Diff` `config/diff.go:154`; `configSensitivePaths` `:39`. Env: `GetEnvironment()` `environment.go:16`; `applyEnvironmentMap` `:89`; `removeEnvOverrides` `:138`.
- Permission filter: `makeFilterConfigByPermission(filterType)` `api4/config.go:408`; globals `writeFilter`/`readFilter` `:22-23`.

---

## Technical debt

Ranked, most → least significant. Each item tags evidence vs. inference.

> **Re-ranking note (ast-grep pass).** Item #1 was the original report's top risk but has been **downgraded** — the ast-grep verification showed the local handlers are exercised by tests (only `localGetClientConfig` is genuinely untested). It is kept at position #1 for traceability with the original report, but its true significance is now **low**; treat item #2 (database backend tested on Postgres only) as the effective top gap.

### 1. Local-mode admin handlers — one untested handler, not four `[evidence — CORRECTED]`
⚠️ **This item was substantially overstated in the original report and is corrected by the ast-grep pass.** Original claim: all four local handlers "essentially untested." Reality:

`th.LocalClient` is built against the local-mode socket (`th.CreateLocalClient(...LocalModeSocketLocation)` — `api4/apitestlib.go:237`), so its calls route to the `APILocal(...)` handlers registered in `config_local.go:19-24`. `config_test.go` makes these calls:
- `localGetConfig` (:27) ← `LocalClient.GetConfig` **3×** (e.g. `config_test.go:63, 545, 769`) — **reached by tests**.
- `localUpdateConfig` (:52) ← `LocalClient.UpdateConfig` **8×** (`:216, 222, 250, 256, 331, 337, 552, 774`) — **reached by tests**.
- `localPatchConfig` (:99) ← `LocalClient.PatchConfig` **3×** (`:770, 1015, 1022`) — **reached by tests**.
- `localMigrateConfig` (:160) ← `LocalClient.MigrateConfig` **1×** (`TestMigrateConfig`, `config_test.go:1034/1063`) — **reached by tests**.
- **`localGetClientConfig` (:191) ← zero test reach — the one genuine gap.**

Why the original grep missed it: grepping the handler *identifiers* (`localUpdateConfig`, …) returns **0** in `*_test.go` (re-confirmed: ast-grep=0 **and** grep=0) — but that is expected, because the handlers are reached through HTTP routing, not by direct call. The original report drew "untested" from that zero.

**Net correction:** the literal "zero handler-level tests" is refuted — these handlers are *exercised* by tests (e.g. `:552` asserts on the returned config). The residual gap is `localGetClientConfig` (the local client-config read) — zero reach. **Caveat (preserves the original author's intent):** "reached" counts call sites, not assertion depth — some calls discard returns (e.g. `:774` looks like a state-restore, not an assertion). So a *narrower* depth-of-coverage question may remain: whether the **permission-bypass branch specifically** (the security concern that motivated the original item) is asserted, versus merely traversed. That is a smaller, sharper gap than "essentially untested," and no longer the report's top risk.

### 2. Database backend tested on Postgres only `[evidence]`
`server/config/main_test.go:48` hard-fails on any non-Postgres driver; `database_test.go` hardcodes Postgres assertions (e.g. `:1058`). The **MySQL `DatabaseStore`** persist/load/DSN-parse path is unexercised in this environment. **Unknown:** behavior on MySQL is not determinable here.

### 3. Cross-stack coupling under-acknowledged in the repo map `[evidence → corrects inference]`
Repo-map Risk Zone 4 labels the config cross-stack ripple *"inference, not a measured co-change pair."* Git refutes the caveat: `config.go ↔ config.ts` **34**, `config.go ↔ admin_definition.tsx` **24**, `config.ts ↔ admin_definition.tsx` **23** (window-bounded). The config surface is the one place the map's "two stacks barely co-change" claim **does not hold** — it is the backend↔frontend seam (alongside i18n). **Recommendation:** upgrade Risk Zone 4 from `[inference]` to `[git]`.

### 4. `patchConfig` Cloud-guard parity gap `[evidence]`
`updateConfig` has a subtest "Should not be able to modify ComplianceSettings.Directory in cloud"; the parallel branch in `patchConfig` (~`config.go:332`) has **no matching subtest**. Also unverified in both: the `restricted_merge.app_error` internal-error path and the Cloud `ToJSONFiltered` response branch.

### 5. Emitter fan-out has no concurrency test `[evidence]`
`invokeConfigListeners` + `sync.Map` registration signal thread-safety intent, but `TestEmitter` (`emitter_test.go:15`) is single-threaded. Concurrent AddListener/RemoveListener/invoke races are uncovered. **Inference:** the emitter is on the synchronous write path and triggers logger/search reconfig, so a race here has runtime blast radius.

### 6. Thin platform/app lifecycle coverage `[inference]`
Platform methods `ConfigureLogger`, `CleanUpConfig`, `EnsureAsymmetricSigningKey`, `isUpgradedFromTE`, `regenerateClientConfig` have no targeted tests (only partial indirect reach). App-layer `IsConfigReadOnly`, `GetConfigFile`, `MailServiceConfig`, `ensure*` similarly lack direct assertions. Many are thin pass-throughs (covered indirectly via api4 tests), so this is lower-confidence than items 1–2.

### 7. Migration coverage limited to file↔database `[evidence]`
`migrate_test.go` (`TestMigrate` :19) covers database↔file combos (skipped under `-short`); memory-store source/dest combos and `Migrate` error paths (open failure, partial copy) are uncovered.

### Structural debt (not a coverage gap)
- **Env-vs-persisted ambiguity** `[inference]`: a value set via API that is *also* present as `MM_*` is **persisted as the API value yet served as the env value** (env applied after load/set, stripped before persist via `removeEnvOverrides`). Operationally surprising; correct by design.
- **Client config is asynchronous to the write** `[inference]`: the browser-facing config is a separate atomic cache refreshed only by the post-write listener via `ps.Go(...)` — a write's client-visible effect lags the 200 response.

---

## Blast radius

**Two nearly-disjoint radii.** The store seam is field-agnostic (`BackingStore.Set/Load` move the whole `*model.Config` as opaque JSON — `config/store.go:42-68`), so a field-add never touches the store, and a store change never touches a field.

### A. Config-FIELD change (cross-stack, git-measured)
Grounded in a textbook single-commit field-add: **`471fd8d1`** (added `FileSettings.ExtractContentTimeout`). Co-change counts are window-bounded against `model/config.go` (70 commits) unless noted.

**Must change together (mandatory):**
| File | Role | Co-change |
|---|---|---|
| `server/public/model/config.go` | field decl + `SetDefaults` (`:1811`, `:1908` in `471fd8d1`) | — |
| `server/public/model/config_test.go` | defaults/validation tests | 32/70 |
| `webapp/platform/types/src/config.ts` | FE TS type mirror | 34/70 |
| `webapp/channels/src/components/admin_console/admin_definition.tsx` | admin-console UI binding (`[import]` config-driven, **92 import statements** = 91 runtime + 1 `import type`; the repo-map's "87" is a dependency-cruiser edge count, not a statement count — *refined by ast-grep*) | 24/70 |
| `server/i18n/en.json` | server labels + `IsValid` error strings | 39/70 (top partner) |
| `webapp/channels/src/i18n/en.json` | FE labels | 33/70 |
| `e2e-tests/playwright/lib/src/server/default_config.ts` | e2e config snapshot | 31/70 |
| `e2e-tests/cypress/tests/support/api/on_prem_default_config.json` | e2e config snapshot | 14/70 |

**Conditional (only if the field has the property):**
- `IsValid`/`isValid` block in `config.go` — only if constrained. Adding validation **forces a new i18n key** in `server/i18n/en.json` (e.g. `model.config.is_valid.extract_content_timeout.app_error` at `config.go:4599`).
- `server/config/client.go` (`GenerateClientConfig`/`GenerateLimitedClientConfig`, `:16`/`:280`) + `client_test.go` — only if the field is browser-exposed. Co-changes 15/70 — ~1 in 4 field-adds is client-visible.

**Cheap regen / coincidental (NOT config coupling):**
- `server/channels/store/{retrylayer,timerlayer,opentracinglayer}.go` (8–9 co-changes) — regenerate off the `channels/store` interface, ride along in large PRs. `[regen]`.
- `go.mod`/`go.sum`, `package-lock.json` — dependency churn. `[regen]`.

**NOT in the field-add radius — migrations.** `server/config/migrations/{mysql,postgres}/` holds only 3 infra DDLs (create `Configurations` table, create `ConfigurationFiles` table, add `SHA` column). Config persists as `value text` (JSON blob) — schema is field-agnostic. The migrations dir appears in **zero** co-change top-20 lists. **A field-add needs no migration.** `[evidence: grep absence + DDL inspection]`

### B. Config-STORE-INTERFACE change (`BackingStore`) — static-only, git-empty
`server/config/store.go` had only **3 commits** in the window; no reliable git signal — rely on the manual-grep static view (Go has no static import graph — `[unknown]`).

**Must change together (manual-grep):** `store.go` (interface `:42-68`) + the 3 implementers `file.go` / `database.go` / `memory.go` + their `_test.go`. Migrations belong **here** (touched only if the DatabaseStore table shape changes), not in the field-add radius.

**No mock layer:** `server/config` has no `.mockery.yaml`, no `mocks/`, no `*Mock*` for `BackingStore`. Changing the interface is a pure hand-edit across 3 implementers + tests — cheap fan-out, **no `[regen]` layer** (unlike `channels/store`/`einterfaces`).

### Top git co-change partners (window-bounded)
- **`model/config.go`** (70): `server/i18n/en.json` 39 · `config.ts` 34 · `webapp i18n/en.json` 33 · `config_test.go` 32 · e2e `default_config.ts` 31 · `admin_definition.tsx` 24 · `config/client.go` 15 · cypress `on_prem_default_config.json` 14.
- **`config.ts`** (59): e2e `default_config.ts` 42 · webapp `i18n/en.json` 39 · `config.go` 34 · `server/i18n/en.json` 32 · `admin_definition.tsx` 23 · `config/client.go` 17 · `client4.ts` 14.
- **`admin_definition.tsx`** (55): webapp `i18n/en.json` 45 · `config.go` 24 · `server/i18n/en.json` 24 · `config.ts` 23 · e2e `default_config.ts` 21.
- **`api4/config.go`** (6, low churn): `config_test.go` 4 · `config_local.go` 4.
- **`config/store.go`** (3, statistically empty): no partner > 2 — ignore git, use static view.

**Model contract static fan-in (`model.Config`):** `app` 77, `api4` 59, `config` 18, `app/platform` 16, `mmctl` 11, `jobs` 10. *(ast-grep pass: confirmed — the metric is **non-recursive file count** = files directly in the dir that reference `model.Config`; mmctl's 11 live in `cmd/mmctl/commands`. At **reference granularity** ast-grep counts far more usage sites — `app` 772, `api4` 1014, `config` 247, `app/platform` 95, `cmd/mmctl/commands` 146, `channels/jobs` 85 — a finer fan-in measure than file count.)*

### Correction to a second repo-map claim
The map says e2e tests are "committed in isolation (best partner 54)." **For the config surface specifically that is false** — `e2e-tests/playwright/lib/src/server/default_config.ts` is the **#1** co-change partner of `config.ts` (42) and #5 of `config.go` (31). Config e2e snapshots are tightly coupled, not isolated.

---

## Structural verification (ast-grep)

Every **structural** claim in this report (call-site / fan-in counts, "only here" single definitions, interface method counts, "always via X", repeated call shapes) was re-checked with `ast-grep` (v0.44.0) against the nested repo @ `ee04f28`. Per the method requirement, **every ast-grep `0` was cross-checked with classic `grep`** to distinguish a real absence from a bad pattern. Verdicts: ✅ confirmed · 🔧 refined · ❌ refuted.

| # | Structural claim | ast-grep pattern (lang) | Result | Verdict |
|---|---|---|---|---|
| 1 | `SetDefaults`/`IsValid`/`Clone`/`Sanitize` each defined **once** on `*Config` | `func ($R *Config) <name>(…) {…}` (go) | 1 each — `config.go:4276/4340/4226/5316` | ✅ confirmed (lines exact) |
| 2 | `makeFilterConfigByPermission` defined once | `func makeFilterConfigByPermission($$$) $$$ {…}` (go) | 1 — `api4/config.go:408` | ✅ confirmed |
| 3 | `model.Config` fan-in: app 77 / api4 59 / config 18 / app·platform 16 / mmctl 11 / jobs 10 | `model.Config` (go), counted per dir | exact match to **non-recursive file count**; mmctl in `cmd/mmctl/commands` | ✅ confirmed (metric clarified) + 🔧 refined to reference-level counts (app 772 / api4 1014 / config 247 / app·platform 95 / mmctl-cmds 146 / jobs 85) |
| 4 | `BackingStore` interface = **8 methods** (Set/Load/GetFile/SetFile/HasFile/RemoveFile/String/Close) | read of `config/store.go:42-68` | exactly 8, names match | ✅ confirmed |
| 5 | `BackingStore` has **3 implementers** (File/Database/Memory) | `func ($R $T) Load() ([]byte, error) {…}` (go) | 3 — `file.go:129`, `database.go:223`, `memory.go:72` | ✅ confirmed |
| 6 | `config.Listener = func(oldCfg, newCfg *model.Config)` | `type Listener func(oldCfg, newCfg *model.Config)` (go) | 1 — `emitter.go:14` | ✅ confirmed |
| 7 | `invokeConfigListeners` ranges a `sync.Map` | grep within `emitter.go` | `listeners sync.Map` (:18) ranged at `:35` (`invokeConfigListeners` :34) | ✅ confirmed |
| 8 | Reads served from cache — `Store.Get()` makes **no backend hit** | body of `func (s *Store) Get()` (go) | 0 `backingStore` refs; returns `s.config` under `RLock` | ✅ confirmed |
| 9 | Cluster (HA) fan-out via `clusterIFace.ConfigChanged(...)` | grep call-site | `platform/config.go:121` | ✅ confirmed |
| 10 | `patchConfig` is the **same shape** as `updateConfig` | per-func grep of shared calls (go) | both: `MakeAuditRecord` → `writeFilter` → `IsValid` → `SaveConfig` → `Diff` (`config.go:120` vs `:279`) | ✅ confirmed |
| 11 | `admin_definition.tsx` "**87 imports**" | `import $$$ from '$_'` (tsx) | **92** statements (91 runtime + 1 `import type`) | 🔧 refined — 87 is a dependency-cruiser edge count, not statement count |
| 12 | Local handlers `localGetConfig`/`localUpdateConfig`/`localPatchConfig`/`localGetClientConfig` have **zero handler-level tests** | `<handler-name>` identifier in `config_test.go` (go) → 0, **grep-confirmed 0** | identifiers absent, **but** handlers are reached via routing: `LocalClient.GetConfig` 3× / `UpdateConfig` 8× / `PatchConfig` 3× / `MigrateConfig` 1×; only `localGetClientConfig` = 0 | ❌ refuted — see Technical-debt item #1 (only `localGetClientConfig` is genuinely untested) |

**Method note on the zeros (claim 12):** ast-grep returned `0` for every `local*Config` identifier inside `config_test.go`, and `grep -c` confirmed `0` — so the pattern was correct and the absence is real. The absence is just *not evidence of no test coverage*: it reflects that HTTP handlers are wired by route, not called by name. The actual coverage was found by matching the `LocalClient.<Op>` call shapes, which the local-mode socket routes to the `APILocal(...)` handlers (`api4/config_local.go:19-24`, client built at `api4/apitestlib.go:237`). This is the single substantive correction from the ast-grep pass.

## Evidence / Inference / Unknown (consolidated)

**Evidence (read/measured):**
- All handler/app/platform/store/backend logic and line numbers (sub-agents read the files).
- Store ordering: `SetDefaults → Desanitize → applyEnvironmentMap → IsValid → removeEnvOverrides → backingStore.Set → emit`.
- Backend fork (File `os.WriteFile 0600`; Database sha-skip + active-row swap + version history; Memory clone).
- Synchronous emitter `sync.Map` fan-out; master listener → client-config regen + WS broadcast + logger reconfig.
- All co-change counts (git over the stated window); field-add commit `471fd8d1`; absence of config-store mocks and config migrations for field-adds.
- Test-file inventory and the specific test funcs cited.

**Inference (interpretation):**
- Reads never hit the backend (served from `s.config` cache; `Load()` is the only re-read).
- The two blast radii are disjoint because `BackingStore` is field-agnostic.
- `IsValid` and `client.go` are *conditional* layers; validation additions force a new i18n error key.
- `store/{retry,timer,opentracing}layer.go` co-changes are coincidental regen ride-alongs.
- Env-vs-persisted precedence and async client-config effect (above).
- Test gap rankings and "covered indirectly" labels.

**Unknown (needs a live run / not opened):**
- **Coverage analysis is static inference** — no instrumented `go test -cover` (needs DB/build). No coverage percentages claimed.
- The Go backend has **no static import graph** (`[unknown]` per repo map); all backend "static" edges are manual-grep, not tool-derived.
- MySQL DatabaseStore path; cluster receive-side `ConfigChanged` apply; internals of `Merge`/`Desanitize`/`fixConfig`/`equal`/`marshalConfig`; the full `IsValid` branch set; whether indirectly-exercised methods hit their error sub-branches.

---

## Code References

- `server/channels/api4/config.go:35-250` — REST handlers (get/update/patch/reload), permission filter `:408`.
- `server/channels/api4/config_local.go:27-191` — local-mode handlers (reached via `th.LocalClient`; only `localGetClientConfig` untested — corrected item #1).
- `server/channels/app/config.go:31-261` — app facade.
- `server/channels/app/platform/config.go:40-244` — platform service, client-config cache.
- `server/channels/app/platform/service.go:468-482` — master config listener (regen + WS + logger).
- `server/config/store.go:27-241` — Store + BackingStore interface + core `Set`.
- `server/config/{file,database,memory}.go` — three backends.
- `server/config/emitter.go:14-40` — listener signature + fan-out.
- `server/config/environment.go:16-149` — env overrides; `migrate.go` — config migration.
- `server/public/model/config.go:4226-5492` — Clone/SetDefaults/IsValid/Sanitize/FilterConfig.
- `webapp/platform/types/src/config.ts`, `webapp/channels/src/components/admin_console/admin_definition.tsx` — FE end of the seam.

## Architecture Insights

- **Cache-first reads, fan-out writes.** The store is the source of truth held in memory; the backend is write-through + reload-only. Runtime effect of a write is delivered by the emitter, not by re-reading.
- **Env overrides are runtime-only.** Applied after load/set, stripped before persist — a deliberate "env wins at runtime, never written" design with a surprising API/env precedence corner.
- **The config surface is the backend↔frontend seam.** It and i18n are the two cross-cutting connectors in an otherwise two-stack repo.
- **Field-agnostic persistence** keeps the field-add radius free of migrations and store/mocks, but concentrates the cost in the contract+UI+i18n+e2e-snapshot fan-out.

## Historical Context (from prior changes)

- `context/map/repo-map.md` — Research Objective pins this flow to `config.go`; Risk Zone 4 names the config surface and (now-corrected) cross-stack inference; Risk Zone 5 notes the backend's `[unknown]` import graph.
- `context/map/artifact-2-structure.md` — FE import edges for `admin_definition.tsx` / `config.ts` / `client4`.
- `context/changes/config-surface/change.md` — this change ("Analyse data flow in a chosen area").

## Open Questions

1. Should `localGetClientConfig` get a dedicated test (the one local handler with zero test reach), and should the permission-*bypass* branch of the already-reached local handlers get explicit assertions rather than just being traversed? (narrowed item #1)
2. Is the MySQL DatabaseStore path covered by CI elsewhere, or genuinely untested (item #2)?
3. Should repo-map Risk Zone 4 be updated `[inference] → [git]` with the measured cross-stack counts?
4. Cluster receive-side `ConfigChanged` apply path — worth a follow-up trace?

---

## Follow-up Research 2026-07-04 — Refactor opportunities

**Date**: 2026-07-04
**Researcher**: czirman (service.mak@proton.me)
**Method**: This section builds ON the evidence already recorded above (it does not re-derive it). It classifies every recorded problem as a *structural refactor candidate* (a fix that would change **code structure**) vs. not, then runs a **read-only, 3-lens exploration** on the candidates: (1) current shape in code, (2) history & intentionality, (3) migration feasibility. Three sub-agents, one per lens, each covering all candidates; all `file:line` relative to nested `mattermost/` @ `ee04f28`. **No code was changed.** This is exploration only — the ranking at the end is a *proposal for a separate planning session*, not a decision.

> **Provenance discipline.** Each candidate is tagged **report-flagged** (the report above calls it debt/risk) or **agent-inferred** (surfaced here from a *neutral* observation the report records without labelling it debt). "Regardless of label" was the instruction — but the reader must be able to see which is which.

### Candidate audit (every recorded problem, classified)

**CANDIDATE = fix changes code structure.** Everything else is retained as a *feasibility/cost input*, not investigated as a refactor.

| ID | Problem (as recorded) | Provenance | Class |
|----|----------------------|-----------|-------|
| **C1** | `model/config.go` is a ~5,765-line file co-locating schema + `SetDefaults`/`IsValid`/`Sanitize`/`Clone`/`FilterConfig` | agent-inferred (size is a descriptive cell; never flagged as debt) | candidate — conditional |
| **C2** | Config schema hand-mirrored across Go/TS/i18n/e2e — 8-file fan-out per field-add, no single source of truth | **report-flagged** (Debt #3, Risk Zone 4, Blast radius A) | candidate — strong |
| **C3** | `updateConfig`/`patchConfig` duplicate the write pipeline | agent-inferred (report records the *sameness* as verification + a *test-parity* gap, not the duplication as debt) | candidate — strong |
| **C4** | `app/config.go` is a "thin delegation facade" | agent-inferred | candidate — weak |
| **C5** | env-overlay logic interleaved into `Store.Set` | agent-inferred | candidate — weak |
| N1 | `localGetClientConfig` untested + permission-bypass assertion depth | Debt #1 | not a candidate — test gap (kept as input) |
| N2 | DatabaseStore tested on Postgres only | Debt #2 | not a candidate — test/CI gap (kept as input) |
| N3 | repo-map RZ4 label should move `[inference]→[git]` | Debt #3 | not a candidate — doc fix |
| N4 | `patchConfig` Cloud-guard **subtest** missing | Debt #4 | not a candidate — test gap **(becomes C3's first step)** |
| N5 | Emitter fan-out no concurrency test | Debt #5 | not a candidate — test gap |
| N6 | Thin platform/app lifecycle coverage | Debt #6 | not a candidate — test gap |
| N7 | Migration coverage file↔database only | Debt #7 | not a candidate — test gap |
| R1 | Env-vs-persisted precedence ("persisted as API value, served as env value") | report-flagged | **redesign, not code-structure** — env-precedence *semantics*; per hard boundary, stop |
| R2 | Client config async to the write | report-flagged | **redesign/behavior, not structural** — making it synchronous is a perf/behavior decision |

### Per-candidate findings

#### C1 — `model/config.go` mega-file `[candidate — conditional]`

- **Current shape (evidence):** 5,765 lines confirmed; behavior outweighs schema ~5:1 (~4,334 func-lines vs ~803 struct-lines). Per-group structs are co-located with their `SetDefaults` (mechanical seam), but per-group `isValid` methods are *centralized* after `Config.IsValid` (:4340) — organization is **inconsistent**. Entangled residue that resists a per-group split: `Config.IsValid`'s **cross-group** checks (SiteURL×EmailBatching :4341, Intune×Office365/SAML :4399, FIPS×password :4415), the **monolithic `Sanitize`** (:5316) hand-enumerating ~71 secret fields across structs, and 163 `NewAppError` sites in the validation span. `[evidence]`
- **Intentionality (evidence):** One file since creation `44714dfcb1` (2015); 776 commits; the only "config refactor" on record (`3a71709103`, MM-13893, 2019) reshaped the *package* boundary, not this file. No split attempt, no "keep it one file" rationale, no rejected-split found. **Verdict: accidental accretion, not a deliberate monolith.** `[evidence]`
- **Feasibility (evidence):** A same-package file split is mechanical and highly reversible; the 77/59/18/16/11/10 fan-in files key on the *type name* `model.Config`, untouched by a split. **BUT** two hazards: (a) CI guard `config-change-checker.yml` + `.github/scripts/check_config_changes_ci.py` are **hardcoded to the path `server/public/model/config.go`** — a field moved to a new file goes *undetected* by the release-note/field scanner; (b) `tools/mattermost-govet/configtelemetry` parses this file for `// telemetry:` comments (file-vs-package scope `[unknown]`). Guard: `config_test.go` (3,459 lines) tests by API not layout — survives a split. `[evidence]`
- **Does it address recorded pain? NO** (all three lenses agree). The field-add fan-out (C2) is driven by TS types / i18n keys / e2e mirrors, which key on field *names*, not the Go filename. A Go-side split touches none of `config.ts`, `admin_definition.tsx`, or i18n. C1 is a **readability** move, not debt reduction.

#### C2 — schema hand-mirrored across stacks `[candidate — strong, report-flagged]`

- **Current shape (evidence):** Mirror set confirmed by tracing one field (`EmailSettings.EnableEmailBatching`) through 9 representations: Go struct (`config.go:2157`, with `access:` tag) → TS full type (`config.ts:631`) → **a second in-stack TS mirror**, the stringified `ClientConfig` (`config.ts:74`) → UI binding (`admin_definition.tsx:3302`, dotted-string key, not type-linked) → webapp i18n (`en.json:1166`) → server i18n validation ids (`en.json:11550/12102`) → `client.go:71` (`GenerateClientConfig`) → playwright `default_config.ts:369` → cypress `on_prem_default_config.json:255`. **The one existing structural link:** playwright `default_config.ts` *imports* the TS types and is compile-checked against them; every other pairing (Go↔TS, TS↔UI, ↔i18n, ↔client.go, ↔cypress-JSON) is hand-authored with **no enforced link**. `[evidence]`
- **Intentionality (evidence + honest unknown):** A generator *does* exist — `server/scripts/config_generator` (orig. MM-14400) — but it emits **`default.json` only**, one-directionally, Go struct → JSON. The **cross-language** mirror was never generated: no `go:generate` near `config.go`, no removed cross-lang generator, no "keep in sync" contributor doc; the PR template only asks contributors to *list* config changes. Exemplar `471fd8d1` (field-add) edits **7 mirror files by hand** in one commit. **Verdict: codegen applied only where cheap/single-source; cross-lang mirror is unbuilt-by-default, explicit "we chose not to" decision is `[unknown]` — record thin.** `[evidence]`
- **Feasibility (evidence):** Target shape (named only, per hard limit): **struct-tag-driven schema source generating the TS/i18n/e2e mirrors** — largely a **tooling/build** change layered on existing metadata (581 `access:` + 306 `restrictable` tags already machine-readable). **No CI job enforces cross-stack parity today** — `config-change-checker.yml` is a *non-failing release-note generator*, not a drift guard; every other guard is single-stack. **First prerequisite: a fail-on-mismatch CI drift-check** across the mirrors (make drift *detectable and enforced*) **before** making any stack generated. Reversible per-stack if generated output stays committed/reviewable. `[evidence]`

#### C3 — `updateConfig`/`patchConfig` duplicate pipeline `[candidate — strong]`

- **Current shape (evidence):** `updateConfig` :120–251 (~132 ln), `patchConfig` :279–406 (~128 ln); **no extracted wrapper** — both inline the whole pipeline; shared code is only the library calls (`Merge`/`Diff`/`IsValid`/`SaveConfig`/filters). The **tail** (`IsValid → SaveConfig → Diff/audit → SanitizedConfig → readFilter Merge → response`) is ~40–50 near-identical lines differing only by the handler-name string. The **head divergence is genuine**: `update` = full-replace (`SetDefaults` + silent overrides + ES/locale side-effects); `patch` = partial-merge (no `SetDefaults`, reject-on-forbidden-field, empty-plugins normalization). `[evidence]`
- **Intentionality — CRITICAL (evidence):** Two endpoints are a *deliberate* full-vs-sparse design. But the **hand-copied security-guard list drifted and caused real harm**: `c45a675553` (2026-06-15, MM-68976) — *"the full PUT /config endpoint silently preserves `PluginSettings.SignaturePublicKeyFiles`… but the sparse /config/patch endpoint had no equivalent guard, so a session with `sysconsole_write_plugins` could modify the field through it"* — a **latent authorization gap** (the plugin-signature cert allow-list was mutable via `patch`) open until June 2026. Earlier, `84e267e9e8` (2026-01-06, MM-66789) added the `ImportSettings.Directory` restriction to `patch` that `update` already had. **Verdict: deliberate handlers, accidental guard-duplication that demonstrably shipped a security-relevant defect.** `[evidence]`
- **Feasibility (evidence):** **Not** a mechanical extraction — for `EnableUploads` / `ImportSettings.Directory` / `MarketplaceURL`, `update` **silently coerces** while `patch` **403-rejects**: same intent, *different observable API contract*. Collapsing them into one pipeline is an **API-contract decision, not code cleanup** — per the hard boundary, that part is flagged and stopped. The safely-shareable structure is narrower: the **guard *list*** (which fields are protected) and the response tail. Guard state: `TestUpdateConfig` has a cloud-guard subtest; `TestPatchConfig` has **none** (parity gap confirmed — this is N4). **First prerequisite: characterization tests pinning *both* handlers' current behavior** — write the missing `patchConfig` cloud-guard subtest **and** tests asserting the silent-coerce-vs-403 divergence — *before* any extraction, or a shared helper silently changes one endpoint's contract. `[evidence]`

#### C4 — thin app facade `[candidate — weak → reject]`

- **Shape (evidence):** `App.Config()`/`SaveConfig` are one-line pass-throughs, but `GetSanitizedConfig`/`SanitizedConfig`/`HandleMessageExportConfig` carry logic, and `PlatformService.SaveConfig` beneath is *not* thin (plugin hook loop, cluster fan-out). **Intentionality (evidence):** deliberate App↔platform-service boundary (config extraction `3a71709103`, MM-13893, 2019). **Feasibility:** removing the indirection couples 77 app + 16 platform files directly to platform internals the facade hides — load-bearing. **Reject.**

#### C5 — env-overlay in `Store.Set` `[candidate — weak → reject]`

- **Shape (evidence):** `Set` (:189 apply, :197 remove) and `Load` weave `applyEnvironmentMap`/`removeEnvOverrides` (`environment.go` :89/:138) into the write path; the Store holds dual `config`/`configNoEnv`. **Intentionality (evidence):** deliberate "env wins at runtime, never persisted" contract (`configNoEnv`, `GetNoEnv`, `GetEnvironmentOverrides` are a first-class API). **Feasibility:** the env step cannot leave `Set` without relocating the dual-representation invariant; risks persisting env-injected values. **Reject.**

### CI / test safety net (feasibility ground truth)

- `server-ci.yml → test-postgres-normal` runs `server/config` tests, **Postgres-only** (`main_test.go:48` fails other drivers) — BackingStore has a net, one shard only (this is N2). `[evidence]`
- `config_test.go` (model, unit, no DB) — strong net for C1 and schema defaults. `[evidence]`
- `config_test.go` (api4) — net for C3, with the confirmed **`patchConfig` cloud-guard parity gap** (N4). `[evidence]`
- `webapp-ci.yml` (`check-types`, `test-platform`) catches TS-side breakage *within* the TS stack, **not** Go↔TS drift. `[evidence]`
- `config-change-checker.yml` — release-note generator, single-stack, **non-failing**; NOT a sync/drift guard. `[evidence]`
- **Gap for C2:** no CI job enforces cross-stack config parity — every guard is single-stack.

---

### Refactor opportunities (ranked — proposal for a separate planning session)

Ranked by **debt cost vs. change cost**, on the evidence above. Two candidates clear the bar as genuine structural refactors; the rest are rejected below with reasons.

#### #1 — C3: de-duplicate the config-write **guard registry** (not the handlers)

- **Current → target shape:** two handlers each inlining a hand-copied protected-field guard list and persistence tail → **a single source-of-truth guard registry both handlers consult**, handlers retained as distinct full-vs-sparse endpoints. (Target *named only*; the merge policy is out of scope — see caveat.)
- **Why #1 (debt cost vs change cost):** Highest **debt cost** of any candidate — the *only* one with a documented shipped defect (MM-68976 latent authorization gap, open ~to 2026-06; MM-66789 earlier drift). The hazard is ongoing: every future protected field must be added to two hand-maintained lists correctly. **Change cost of the safe slice is low** (a guard list + parity tests, one file `api4/config.go`). **Caveat that caps the scope:** unifying the *enforcement behavior* (silent-coerce vs 403-reject) is an **API-contract decision, not a refactor** — stop there and leave it to product/API owners.
- **Blast radius:** one file (`api4/config.go`, 465 ln); consumers are HTTP clients → behavior parity matters more than call-site count. `[evidence]`
- **Incremental, reversible path (sketch):** (1) write characterization tests pinning both handlers' current behavior incl. the missing `patchConfig` cloud-guard subtest; (2) extract the protected-field *list* to one place both handlers read; (3) leave silent-coerce-vs-reject enforcement untouched. Each step reversible; net behavior unchanged.
- **First prerequisite step:** the **N4 test** — write the `patchConfig` cloud-guard parity subtest (+ characterization tests for the 3 divergent fields). This is a pure test addition, independently valuable, and gates everything else.

#### #2 — C2: single source of truth for the config schema

- **Current → target shape:** N hand-authored mirrors (Go struct, 2× TS, UI binding, 2× i18n, 2× e2e snapshots) kept in sync by hand → **a struct-tag-driven schema source that generates the downstream mirrors** (target *named only*; the generator is deliberately not designed here).
- **Why #2 (debt cost vs change cost):** Highest **structural** debt cost — the report-measured 8-file cross-stack tax on *every* field-add (34× / 24× co-change), with **no drift guard** anywhere, so silent divergence is a standing risk (and the mechanism behind the C3-class gaps). But **change cost is high and tooling-heavy**, and part of it (build/CI wiring) is not pure code structure — hence #2, not #1.
- **Blast radius:** the field-add radius (Blast radius A above): 8 files mandatory, `config/client.go` conditional. `[report]`
- **Incremental, reversible path (sketch):** (1) add a **fail-on-mismatch CI drift-check** comparing the mirrors — detectable/enforced parity, valuable on its own; (2) once parity is guarded, convert **one** stack at a time (likeliest first: the two e2e snapshots, already partly type-linked) from hand-authored to generated; (3) leave Go struct as source. Reversible per-stack while generated output stays committed.
- **First prerequisite step:** the **cross-stack drift-check CI job**. You cannot safely replace a hand-mirror with a generated one until a guard proves current byte-parity — and this guard is the missing safety net regardless of whether generation ever follows.

### Considered and rejected

- **C1 — split `model/config.go`:** structurally a candidate (mechanical same-package split, reversible) but **does not address the recorded pain** — the fan-out keys on field *names*, not the Go filename (all three lenses agree). Adds real risk: the CI field-checker is **hardcoded to the file path**, and `Config.IsValid`'s cross-group checks + the monolithic `Sanitize` form an entangled core that resists clean decomposition. A readability play, not debt reduction → out of the top ranking.
- **C4 — collapse the app facade:** deliberate, load-bearing App↔platform boundary (MM-13893); removing it couples ~93 files to platform internals. Reject.
- **C5 — lift env-overlay out of `Store.Set`:** deliberate "env wins at runtime, never persisted" invariant with a first-class `configNoEnv` API; not a clean seam. Reject.
- **R1 (env-vs-persisted precedence) / R2 (async client-config):** report-flagged but their real fix is **redesigning behavior/semantics**, not code structure — explicitly out of this study's scope per the hard boundary. Route to a separate product/behavior analysis.

### Feasibility/cost inputs carried forward (non-candidates)

Not refactors, but they shape the two opportunities above: **N4** (missing `patchConfig` parity subtest) *is* C3's first step; **N2** (Postgres-only config-store CI) bounds the safety net for any store-touching work; **N1/N5/N6/N7** (coverage gaps) raise the value of adding characterization tests before either refactor. **N3** (repo-map RZ4 label) is a doc fix already actioned in Debt #3.

### Method boundary (what this follow-up did NOT do)

No code changed; no target architecture designed beyond *naming* the target shape and the first prerequisite step per candidate. The C3 enforcement-unification and the C2 generator design were explicitly **stopped at the semantics/tooling line** and left to the planning session. The ranking is a proposal, not a decision.
