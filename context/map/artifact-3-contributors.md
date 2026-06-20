# Artifact 3 — Mattermost Key Contributors by Area (last 12 months)

Who to talk to for each high-contact area identified from
[`artifact-1-territory.md`](./artifact-1-territory.md) (churn) and
[`artifact-2-structure.md`](./artifact-2-structure.md) (coupling/risk). Answers
*"who can offer support in this area?"*

## Scope & method

- **Repo / window:** `mattermost/` monorepo, 2025-06-20 → 2026-06-20, non-merge commits only.
- **Signal:** per-area = distinct commits touching the area's paths; per-person topic
  classification = file-touch counts bucketed into sub-areas (`git log --name-only`).
- **Bots / automation excluded:** `cursor[bot]`, `dependabot[bot]`, `unified-ci-app[bot]`,
  `Weblate (bot)` / `Hosted Weblate`, `mm-prodsec-bot`, `R Oyanagi` (translate.mattermost.com).
- **AI agents:** 267 commits carry AI co-author trailers (Claude / Cursor Agent). These are
  *human-authored, AI-assisted* — a named human is the author and is accountable, so the human
  is counted. Only fully autonomous bot-account commits (e.g. `cursor[bot]`) were dropped, since
  those have no noticeable human driver.

---

## Area → key contributors (commits, bots removed)

### 1. Admin Console — `webapp/channels/src/components/admin_console`
| Contributor | Commits |
|---|---|
| Pablo Vélez | 25 |
| Ibrahim Serdar Acikgoz | 21 |
| Harrison Healey | 16 |
| Maria A Nunez | 14 |
| Vicktor (Victor-Nyagudi) | 11 |
| Jesse Hallam | 11 |
| Saturnino Abril (sabril) | 10 |

**Go-to:** **Pablo Vélez** & **Serdar Acikgoz** (most active hands-on), **Harrison Healey**
(frontend lead — 609 admin_console file-touches, owns the surrounding component graph).

### 2. Backend spine — `server/{public/model, channels/store, channels/app, channels/api4}`
| Contributor | Commits |
|---|---|
| Jesse Hallam | 66 |
| Ben Schumacher | 50 |
| Harshil Sharma | 40 |
| Pablo Vélez | 39 |
| Ibrahim Serdar Acikgoz | 37 |
| Doug Lauder (wiggin77) | 27 |
| Alejandro García Montoro | 27 |

**Go-to:** **Jesse Hallam** (cross-cutting backend / contract `model`), **Ben Schumacher**
(`app` core + `mmctl`), **Alejandro García Montoro** (enterprise/platform internals).

### 3. SCC orchestrators — `webapp/channels/src/actions` + `utils/utils.tsx`
| Contributor | Commits |
|---|---|
| Harrison Healey | 12 |
| Devin Binnie | 8 |
| Scott Bishel | 5 |
| Ibrahim Serdar Acikgoz | 4 |
| Nick Misasi / Jesse Hallam / Harshil Sharma / Doug Lauder | 3 |

**Go-to:** **Harrison Healey** & **Devin Binnie** — the only two with repeated hands in the
1,070-module cycle hub (websocket_actions / global_actions / utils.tsx). Critical: this is the
area least safe to edit, so contact before touching.

### 4. Config surface — `config.go` / `config.ts` / `admin_definition.tsx` / `client4.*`
| Contributor | Commits |
|---|---|
| Jesse Hallam | 17 |
| Pablo Vélez | 14 |
| Ben Schumacher | 14 |
| Harshil Sharma | 12 |
| Ben Cooke | 11 |
| Maria A Nunez | 10 |
| Ibrahim Serdar Acikgoz / Alejandro García Montoro | 10 |

**Go-to:** **Jesse Hallam** (config schema authority), **Ben Schumacher** (backend config),
**Pablo Vélez** (`admin_definition.tsx` UI binding).

### 5. e2e + i18n — `e2e-tests/{playwright,cypress}` + `i18n/en.json`
| Contributor | Commits |
|---|---|
| Pablo Vélez | 54 |
| Saturnino Abril (sabril) | 43 |
| Jesse Hallam | 30 |
| Harrison Healey | 28 |
| Maria A Nunez | 24 |
| Harshil Sharma | 24 |
| Ibrahim Serdar Acikgoz | 22 |

**Go-to:** **Saturnino Abril** (the e2e/test specialist, leading Cypress→Playwright migration),
**Pablo Vélez** (highest e2e-spec author).

---

## Per-person profile (topic classification → support offered)

Activity bucketed by file-touch count; **bold** = where this person is a primary contact.

| Person | Dominant topics | Best to ask about |
|---|---|---|
| **Jesse Hallam** | srv:app/store/api4/enterprise/model, web:components, tools | **Backend architecture, config/contract (`model`), cross-cutting** |
| **Ben Schumacher** | srv:app (440), mmctl, api4, store, model | **Server `app` core, `mmctl` CLI** |
| **Harrison Healey** | web:components (2005), admin_console (609), post_view, utils, e2e | **Frontend lead: webapp, admin_console, SCC orchestrators, post composer** |
| **Pablo Vélez** | web:components/admin_console, e2e:playwright, srv:app/store/model | **Admin console end-to-end, e2e specs** |
| **Saturnino Abril (sabril)** | e2e:cypress (784) + playwright (326), web:components/admin_console, ci | **e2e test suites + Cypress→Playwright migration** |
| **Ibrahim Serdar Acikgoz** | web:components/admin_console, srv:app/store/model | Admin console + backend feature work |
| **Harshil Sharma** | web:components/admin_console/post_view, srv:app/store/api4/model | Post view, admin console, backend CRUD |
| **Maria A Nunez** | web:components/admin_console, srv:app/api4/model, mm-redux | Admin console + backend + redux state |
| **Devin Binnie** | web:components/utils/mm-redux, srv:app/store/model | **Frontend `utils`/redux (cycle-adjacent), full-stack** |
| **Nick Misasi** | web:components/admin_console, srv:app/model/api4 (+ .agents/.cursor) | Full-stack features, AI-tooling config |
| **Doug Lauder (wiggin77)** | srv:platform/app/store/model | Server platform internals |
| **Alejandro García Montoro** | srv:app/enterprise/store/platform/model/mmctl | Backend + enterprise + platform |

## Key takeaways

1. **Two clear leads:** **Jesse Hallam** anchors the backend (and config/contract), **Harrison
   Healey** anchors the frontend (and the dangerous SCC orchestrators).
2. **Admin console** has the deepest bench — Pablo Vélez, Serdar, Maria, Harshil, Harrison — so
   support is easy to find there despite its risk.
3. **Highest-risk, thinnest bench: the SCC cycle hub (Area 3).** Only Harrison Healey and Devin
   Binnie have recurring involvement — single points of knowledge; engage them early.
4. **Tests are specialist-owned:** Saturnino Abril + Pablo Vélez carry e2e and the framework
   migration; route test/CI questions to them.
