# Artifact 2 — Mattermost `webapp` Static Structure (dependency-cruiser)

Static **dependency-structure** analysis of the frontend (`webapp/`), complementing the
git-history view in [`artifact-1-territory.md`](./artifact-1-territory.md). Where
artifact-1 answers *"where does the work happen?"*, this artifact answers *"how is the
code wired, and what does that wiring cost when you try to change or test it?"*

Companion visual: [`artifact-2-active-areas-cycles.svg`](./artifact-2-active-areas-cycles.svg).

## Scope & method

- **Tool:** `dependency-cruiser@17.4.3` over `webapp/{channels/src, platform/client/src, platform/types/src}`.
- **Focus areas** (the active territory from artifact-1): `channels/src/components/admin_console`,
  `channels/src/packages` (mattermost-redux), `channels/src/utils`, `channels/src/actions`,
  `platform/client/src`, `platform/types/src`.
- **Resolution adaptations** (the stock `.dependency-cruiser.cjs` resolves nothing in this checkout):
  1. Added `['.ts','.tsx','.js','.jsx','.json','.d.ts']` to `enhancedResolveOptions.extensions`
     — the env default excluded TS extensions, so every import was "unresolved".
  2. Mirrored webpack `resolve.modules: ['node_modules','./src']` + the `mattermost-redux` alias
     with a temporary symlink shim (the webapp is not `npm install`-ed; `tsconfig`'s
     `moduleResolution:"bundler"` is not honored by depcruise).
  3. For cross-package edges, symlinked `@mattermost/types` → `platform/types/src` and
     `@mattermost/client` → `platform/client/src` (no built `lib/`). All probe-verified.
- **Edge nature:** all 13,344 edges are **runtime imports** (0 tagged `type-only`; `import type`
  excluded by default), so cascade/cycle figures reflect real load, not compile-only edges.
- Shims were temporary and removed; the only committed output is this file + the SVG.

## 1. Circular dependencies — one giant tangle, not many small loops

- **2,463 modules cruised in `channels/src`; 1,872 circular-violation instances — but these are
  one knot:** a single **strongly-connected component (SCC) of 1,070 modules (≈43% of channels)**.
  Longest single cycle threads **51 files**.
- **Composition of the giant SCC:** components(other) 611, **admin_console 330**, selectors 33,
  **actions 33**, reducers 28, **utils 22**, store(s) 4, types 3.
- **Share of each active area trapped in the SCC:** admin_console **330/571 (58%)**,
  actions **33/55 (60%)**, utils **22/82 (27%)**, mattermost-redux **0/216**.
- **`mattermost-redux` is separable:** 0 members in the giant SCC; it has only small, self-contained
  internal rings (7-node action ring `posts↔channels↔teams↔users↔threads↔emojis↔status`;
  5-node selectors/constants ring; 3-node selectors ring).
- **Barrel cycles vs. structural SCC** — two different problems:
  - *Trivial barrel loops* (`index.ts ↔ component.tsx`): `admin_console.tsx ↔ admin_console/index.ts`,
    `data_grid ↔ data_grid_header/row`, `filter ↔ filter_list`, … → fixable by import hygiene.
  - *Deep structural SCC*: needs real decoupling (dependency inversion / extracted contracts).
- **platform/types:** 4 cycles among entity types (`users↔teams↔channels↔sessions↔groups`);
  **platform/client:** 1 cycle (`websocket_message ↔ websocket_messages`). Low severity.

## 2. Layer boundaries — clean across platform, tangled within `channels`

Intended layering: **`platform/types` (foundation) → `platform/client` → `channels/src` (app)**.

- **All forbidden (upward) directions are exactly zero**, confirmed three independent ways
  (forbidden-rules = 0; direct edge matrix; unresolved-specifier sweep):
  - `platform/types` imports **nothing** outside itself (pure foundation; huge fan-in, zero fan-out).
  - `platform/client` imports **only** types (68 edges); **0** edges into channels.
  - `mattermost-redux` imports only types (309) + client (1); **0** edges back into the app.
- **`channels → platform/client` goes through exactly one file** — the public
  `platform/client/src/index.ts` (no deep imports).
- **The instability is intra-`channels`:** the only "surprising" inbound edges are
  **17 `APP → admin_console`** imports — general-purpose `utils`/`selectors`/`actions` and
  *unrelated* feature modals (trial, analytics, websocket, properties-card) reaching **into**
  admin_console internals. admin_console is **not a leaf**; that's where edits surprise distant code.

**Cross-domain edge matrix (downward = allowed):** `APP→REDUX 1368`, `ADMIN→APP 899`,
`APP→TYPES 737`, `ADMIN→REDUX 395`, `ADMIN→TYPES 393`, `REDUX→TYPES 309`, `CLIENT→TYPES 68`,
`APP→ADMIN 17`, `APP→CLIENT 14`, `ADMIN→CLIENT 4`, `REDUX→CLIENT 1`.

## 3. Testability risks (production modules only)

"Cascade" = modules an import transitively pulls in. Importing **any** giant-SCC module loads
**≈2,240 of 2,478 (90%)** of the graph.

| Area | Prod mods | In giant SCC | Median cascade | Leaf-safe (isolatable) |
|---|---|---|---|---|
| `admin_console` | 571 | **330 (58%)** | **2,240** | 199 (35%) |
| `actions` | 45 | **33 (73%)** | **2,240** | 1 (2%) |
| `utils` | 82 | 22 (27%) | 60 | 41 (50%) |
| `packages` (mattermost-redux) | 207 | 0 | 50 | 71 (34%) |
| `platform/client` | 8 | 0 | 30 | 3 (38%) |
| `platform/types` | 57 | 0 | 2 | 52 (91%) |

**Test strategy by coupling type:**
- **Heavy mocking, unit viable** — bounded reach + direct API-client/action deps: `platform/client/src/client4.ts`
  (cascade 54; mock HTTP via `nock`), action creators that hit the client.
- **Integration > unit** — direct global-state coupling needs a real store: admin_console
  container `index.ts` layer (`connect()`), and all of `mattermost-redux` (dispatch→state, cascade ≤202, 0 in SCC).
- **Modification → e2e** — SCC runtime orchestrators: `actions/websocket_actions.ts` (88 direct imports;
  37 actions + client + state), `actions/global_actions.tsx`, plugin registry, store init; plus
  config-driven UI `admin_console/admin_definition.tsx` (87 imports, #5 most-modified file). Rationale is
  **non-locality** — a change inside the 1,070-node cycle cannot be reasoned about locally.

**Biggest surprise — "pure" utils that detonate the graph:** 14 of 22 SCC-utils have *zero* direct
action/API/state coupling yet cascade to 2,240 (e.g. `utils/url.tsx`, `utils/emoticons.tsx`,
`utils/markdown/renderer.tsx`, `utils/syntax_highlighting.tsx`). They look unit-testable but aren't,
because their import chain loops back into the app (`url.tsx → markdown/renderer.tsx`, itself in the cycle).

**Most suspicious modules:** `actions/websocket_actions.ts`, `utils/utils.tsx` (the #1 cycle hub,
~1,692 cycle instances), `admin_console/admin_definition.tsx`, `actions/global_actions.tsx`,
`utils/post_utils.ts`, the admin container `index.ts` family, and the 14 "pure" SCC utils.

## 4. Companion graph

`artifact-2-active-areas-cycles.svg` — an 8-node, area-collapsed view
(`--include-only` the focus areas + the `components`/`selectors` the cycle routes through;
`--collapse` to area level; `no-circular` rule → cyclic edges **orange**). It answers one question:
*which active areas form a circular tangle vs. a clean lower spine?*
- **Orange tangle:** `admin_console ↔ components ↔ actions ↔ utils ↔ selectors`.
- **Clean spine:** `mattermost-redux → {client, types}`, `client → types`, `types` = pure sink.

## Key takeaways

1. The frontend's structural risk is **one 1,070-module SCC** anchored in `admin_console` (the #1
   most-active area) and stitched by `actions` + `utils/utils.tsx`. The area teams edit most is the
   area least safe to edit.
2. **Platform layering is respected**: `types`/`client`/`mattermost-redux` are clean, one-directional,
   separable lower layers — good proving grounds for the decoupling pattern.
3. **The surprises are intra-`channels`**: non-admin code importing into admin_console internals, and
   "pure" utils silently trapped in the cycle.
4. **Testing follows coupling type**: mock the client (unit), use a real store (integration for redux +
   admin containers), and gate SCC orchestrators + `admin_definition` behind e2e.

## What to check next

- Cut `utils/url.tsx → utils/markdown/renderer.tsx` (and the markdown cluster's back-edge into the app);
  re-run SCC — utils that drop out convert from "heavy-mock" to unit-testable.
- Relocate misplaced `admin_console_*` helpers (in `utils`/`selectors`) back into admin_console; extract
  the shared assets/utils that trial/analytics/websocket code reaches in for.
- Tag the e2e-mandatory set (`websocket_actions`, `global_actions`, plugin registry, store init,
  `admin_definition`) as a CI gate — matches artifact-1's rising Playwright activity.
- Optional next visual: `--focus '^channels/src/utils/utils\.tsx$'` to show that single hub's blast radius.
