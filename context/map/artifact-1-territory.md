# Artifact 1 — Mattermost Territory Map

Git-history analysis of the **mattermost** repository: where the real, hands-on
activity lives, how it changed over the year, how areas couple, and what ties the
whole repo together.

## Scope & method

- **Repo:** `mattermost/` (monorepo: `server/`, `webapp/`, `e2e-tests/`, `tools/`, `api/`)
- **Window:** 2025-06-20 → 2026-06-19 (HEAD `ee04f28`)
- **Volume:** 1,872 non-merge commits, 7,347 unique files touched, 20,196 change-records
- **Signal:** count of change-records per path (`git log --name-only`)
- **Noise filtered out** (4,720 records dropped) for the "where is the work" rankings:
  i18n translation bundles (`*/i18n/*.json`), lockfiles (`package-lock.json`,
  `go.sum`, `go.mod`), snapshots (`*.snap`), mockery `mocks/`, generated store
  layers (`retry/timer/opentracinglayer.go`), `*.pb.go`, `Makefile`, `*.yml/.yaml`,
  `package.json`/`tsconfig`, `.env*`, image/font assets, `CHANGELOG`.
- Edge months **25-06** (Jun 20–30) and **26-06** (Jun 1–19) are partial.

---

## 1. Most active modules (folders)

Top-level (`server/` vs `webapp/`) and even depth-3 were too coarse, so the
frontend monolith `webapp/channels/src` was drilled one layer into `components/`.

| # | Module | Changes |
|---|---|---|
| 1 | `webapp/channels/src/components/admin_console` | 1,429 |
| 2 | `server/channels/app` | 1,383 |
| 3 | `e2e-tests/cypress/tests` | 1,191 |
| 4 | `server/channels/api4` | 795 |
| 5 | `server/public/model` | 714 |
| 6 | `server/channels/store` | 708 |
| 7 | `e2e-tests/playwright/specs` | 437 |
| 8 | `server/channels/db` | 437 |
| 9 | `server/cmd/mmctl` | 304 |
| 10 | `webapp/channels/src/components/post_view` | 273 |

**Read:** activity concentrates on the **admin-console UI**, the
**app→api4→store** server spine, and **e2e test suites**. The post composer/viewer
(`post_view`, `advanced_text_editor`) is the most-churned end-user surface.

## 2. Most modified files

| # | File | Changes |
|---|---|---|
| 1 | `server/public/model/config.go` | 70 |
| 2 | `server/channels/store/store.go` | 66 |
| 3 | `webapp/platform/types/src/config.ts` | 59 |
| 4 | `webapp/platform/client/src/client4.ts` | 58 |
| 5 | `webapp/channels/src/components/admin_console/admin_definition.tsx` | 55 |
| 6 | `e2e-tests/playwright/lib/src/server/default_config.ts` | 53 |
| 7 | `webapp/channels/src/utils/constants.tsx` | 50 |
| 8 | `server/channels/app/post.go` | 48 |
| 9 | `server/public/model/client4.go` | 44 |
| 10 | `server/channels/app/channel.go` | 42 |

**Caveat:** the top is dominated by the **config surface** (`config.go`,
`config.ts`, `admin_definition.tsx`, `client4.*`) — hand-written source, but "everyone
touches them" hubs. Stripping that surface, the real feature hotspots are
**posts, channels, and users** in `app`/`api4`, with test files churning alongside.

---

## 3. Work-pressure trend (since last year)

Monthly filtered file-changes (real code, noise removed):

| Month | 25-06* | 25-07 | 25-08 | 25-09 | 25-10 | 25-11 | 25-12 | 26-01 | 26-02 | 26-03 | 26-04 | 26-05 | 26-06* |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Commits | 81 | 181 | 100 | 206 | 162 | 116 | 87 | 120 | 133 | 165 | 199 | 208 | 114 |
| File-changes | 348 | 1401 | 992 | 1547 | 1092 | 1655 | 624 | 1742 | 1396 | 1589 | 2603 | 2672 | 2535 |

- **Intensity roughly doubled.** A busy month in 2025 was ~1,000–1,650 changes;
  the last full quarter (Apr–May 2026) sits at ~2,600.
- Winter lull (low **Dec 2025**), then a steady, accelerating climb through 2026.
- **The heat relocated to the frontend:** `webapp/components` surged
  (591 in Jan, 892 partial June); `admin_console` went from ~50–100/mo to
  255–312/mo.
- **Test-stack migration visible:** `e2e/playwright` climbs steadily
  (101→93→136→162→210) while `e2e/cypress` goes bursty/fading — Cypress → Playwright.
- `server/db` was a **one-time big bang** (281 in Jul 2025, then near-silence).
- Backend core (`app`/`store`/`model`) re-intensified in spring 2026.

---

## 4. Folder couplings (co-change in the same commit)

**Top pairs**

| Pair | Commits |
|---|---|
| webapp/components + webapp/src(other) | 217 |
| server/app + server/model | 193 |
| server/api4 + server/app | 165 |

**Top triples**

| Triple | Commits |
|---|---|
| server/api4 + server/app + server/model | 107 |
| webapp/components + webapp/platform + webapp/src(other) | 98 |
| server/app + server/model + server/store | 78 |

Two dense clusters dominate and barely mix: a **backend stack**
(`model ↔ store ↔ app ↔ api4`) and a **frontend stack**
(`components ↔ src ↔ platform`). `server/model` is the universal backend connector
(shared contract surface). Conclusions for the top-3 ranked modules:

- **admin_console** — couples almost exclusively within the frontend; its only
  real backend tie is `server/model` (every admin setting needs a config field).
  = "frontend UI + config schema."
- **server/app** — the backend hub (model 193, api4 165, store 111); feature work
  travels as a tight model→store→app→api4 bundle.
- **e2e/cypress** — strikingly *uncoupled* (best partner only 54). Tests are
  committed in isolation (separate test PRs / test-after), not bundled with the
  code they test.

---

## 5. The "common denominator" file

Reading **spread** (distinct areas co-touched) together with **commits** (frequency):

| File | Areas | Commits | What it is |
|---|---|---|---|
| **webapp/channels/src/i18n/en.json** | 112 | 204 | Frontend translation bundle |
| **server/i18n/en.json** | 102 | 184 | Backend translation bundle |
| server/public/model/config.go | 123 | 70 | Config schema struct |
| server/Makefile | 125 | 121 | Build orchestration |
| .github/workflows/server-ci.yml | 186 | 38 | CI definition |

**Answer:** the repo's #1 cross-cutting hub is **`webapp/channels/src/i18n/en.json`**
(204 commits / 112 areas), with **`server/i18n/en.json`** (184 / 102) as its twin.
Any feature, anywhere in the tree, that adds user-facing text must edit one of
these — they are the seam stitching every area together, independent of folder
structure. (This is exactly why they were filtered out of the effort rankings:
they measure *where strings live*, not where engineering happens.) Config
(`config.go`) and build/CI files are secondary denominators of the same kind.

---

## Key takeaways

1. The real hands-on territory is the **admin-console UI** + the
   **app/api4/store backend spine** + **e2e tests**.
2. Work intensity **~2× vs a year ago**, concentrating on the frontend (admin
   console) and Playwright e2e tests in 2026.
3. Backend and frontend are **two weakly-connected coupling clusters**; `server/model`
   bridges the backend; e2e tests are committed in isolation.
4. The true repo-wide **common denominator is the i18n `en.json` translation files**,
   followed by the config schema and build/CI plumbing.
