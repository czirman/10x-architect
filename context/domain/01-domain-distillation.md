---
title: Domain distillation — Mattermost communication core
created: 2026-07-11
type: domain-distillation
---

# Domain distillation — Mattermost communication core

> The product of this document is a **domain map**, not code. Entity, aggregate, and rule
> names were **discovered** from code and source documents, not assumed up front. Every
> claim is anchored by a `file:line` citation that was actually verified.

---

## STEP 0 — Project context (discovery)

**What the product is.** Mattermost is an "open core, self-hosted collaboration platform that
offers chat, workflow automation, voice calling, screen sharing, and AI integration", written
in Go + React, running as a single binary on Linux + PostgreSQL (`mattermost/README.md:3`).
Monthly releases, MIT license (`README.md:3`).

**Requirements documents — LIMITATION.** The repository has **no PRD / vision / tech-stack.md**
(a `find` for `*prd*`, `*vision*`, `*tech-stack*` returned no product document). The only
source narratives are `README.md` (marketing) and `CHANGELOG.md` (an empty stub pointing to
online docs, `CHANGELOG.md:1-4`). Consequently, **I distill the product goal and success
criteria from the contract-layer code** (`server/public/model/`) and from the previously
produced repo map (`context/map/repo-map.md`), and **not** from a requirements document. This
is the strongest limitation of the whole artifact: business rules are reconstructed from code
behavior, so a "declared invariant" = what the code enforces today, not what someone wrote down
as intent.

**Stack and where business logic lives** (verified from the directory layout):

| Layer | Path | Role |
|---|---|---|
| Contract / domain model | `server/public/model/*.go` (284 files) | Entity structs + `IsValid()` — **structural validation**, not business invariants |
| Application logic (service) | `server/channels/app/*.go` | **Cross-entity invariants live here** (membership, threads, groups) |
| Persistence | `server/channels/store/` | Data-access contract, DM/GM uniqueness constraints |
| API | `server/channels/api4/` | HTTP entry point, authorization |
| UI | `webapp/` (TypeScript, React) | Admin console, post composer |

**Distillation scope (deliberate narrowing).** 284 model files are not a distillation target.
I go deep only on the **communication core**: `Team → Channel → ChannelMember → Post`
+ `User` + `Role/Permission`. This is unambiguously the product core ("chat", `README.md:3`)
and where invariants live. The configuration surface (`config.go`) — hub #1 per the git map
(`context/map/repo-map.md:106,146`) — I classify as a **supporting/generic subdomain** and only
bridge to the earlier work in `context/changes/config-surface/`. The narrowing is **not** an
assumption of names — I still discover entities from code; I only declare which subtree I
examined in depth.

---

## STEP 1 — Ubiquitous Language (communication core)

Terms extracted from code + README. Definition / source citation / where it lives in code.

| Term | Definition (discovered) | Concept source | Life in code |
|---|---|---|---|
| **Team** | Top-level space grouping channels and users; type `O` (open) or `I` (invite). | "collaboration platform" `README.md:3` | `type Team struct` `model/team.go:26`; types `TeamOpen/TeamInvite` `team.go:15-16`; `IsValid` `team.go:129` |
| **Channel** | A conversation place inside a Team; six types. | "chat" `README.md:3` | `type Channel struct` `model/channel.go:83`; `type ChannelType` `channel.go:25` |
| **ChannelType** | `O` open, `P` private, `D` direct (DM), `G` group (GM), `BO`/`BP` board. | — | `model/channel.go:28-33` |
| **Direct Message (DM)** | A `D`-type channel between exactly two users; name = `userA__userB`. | — | `model/channel.go:30`; name encoding `channel.go:348-351`; creation `app/channel.go:466` |
| **Group Message (GM)** | A `G`-type channel for 3–8 users. | — | `model/channel.go:31`; limits `ChannelGroupMinUsers=3`/`MaxUsers=8` `channel.go:37-38` |
| **ChannelMember** | A user's membership in a channel; scheme roles + NotifyProps. | — | `type ChannelMember struct` `model/channel_member.go:54`; `IsValid` `channel_member.go:129` |
| **Scheme role (Guest/User/Admin)** | Member's role in the channel permission scheme. | — | `SchemeGuest/SchemeUser/SchemeAdmin` `model/channel_member.go:66-68` |
| **Post** | A message in a channel; optionally a reply in a thread (`RootId`). | "chat" `README.md:3` | `type Post struct` `model/post.go:127`; `RootId` `post.go:136`; `IsValid` `post.go:479` |
| **Thread / RootId** | Thread: a reply post points its `RootId` at a root post; max 1 level. | — | `post.go:136`; enforced in `app/post.go:299-306` |
| **User** | A user account; may be a bot or a remote user (`IsRemote`). | — | `type User struct` `model/user.go` (1160 lines); `IsBot`, `IsRemote()` used at `app/channel.go:471`, `app/post.go:246` |
| **Role / Permission / Scheme** | RBAC model: roles hold permissions; schemes map roles to levels (team/channel). | "workflow automation" `README.md:3` | `model/role.go` (1267 l.); `model/permission.go` (2772 l.); migrations `app/permissions_migrations.go` |
| **GroupConstrained** | Flag: channel/team membership should be synchronized with a linked group (LDAP/SAML). | — | field `channel.go:100`; `IsGroupConstrained()` enforced at `app/channel.go:1802` |
| **SharedChannel** | A channel shared between remote instances (federation). | — | `app/shared_channel.go`; record created at `app/channel.go:513` |
| **Configuration** | Server configuration contract — a supporting subdomain (see STEP 2). | map: hub #1 `repo-map.md:146` | `model/config.go` (NO core business invariant) |

---

## STEP 2 — Subdomain classification (Core / Supporting / Generic)

Core = what constitutes the product's meaning and advantage ("chat" + conversation access
control). The rationale references the vision in `README.md:3` (no formal success criteria — see
STEP 0).

| Area / concept | Category | Rationale |
|---|---|---|
| **Channel + ChannelMember** (membership, types, threads) | **Core** | The essence of "chat"; the whole product value rests on it. The sharpest cross-entity invariants live here (`app/channel.go:1888`, `app/post.go:299`). |
| **Post / Thread** | **Core** | The carrier of the conversation itself. Thread integrity = correctness of the core experience. |
| **Team** | **Core** | The organizational boundary of channels and membership; determines the membership invariant (`app/channel.go:1888`). |
| **Role / Permission / Scheme (RBAC)** | **Core** | Conversation access control is part of the "self-hosted / enterprise" advantage (`README.md:3`, DevSecOps use-case `README.md:11`). |
| **GroupConstrained / group synchronization (LDAP/SAML)** | **Supporting** | Supports enterprise, but it's identity integration, not the essence of chat. |
| **SharedChannel / federation** | **Supporting** | Extends the reach of the core; it is not its essence. |
| **Notifications / NotifyProps** | **Supporting** | Supports the conversation, but interchangeable, not differentiating. |
| **Configuration surface** (`config.go`, `admin_definition.tsx`) | **Generic / Supporting** | The highest-churn hub (`repo-map.md:106,146`), but it's configuration management — a generically solved problem, not a domain advantage. **Bridge to** `context/changes/config-surface/`: that work studied the configuration *flow*; from a DDD perspective it is a technical layer surrounding the core. |
| **Audit / Compliance** | **Generic** | A regulatory requirement, a generic pattern. `model/audit.go`, `compliance.go`. |
| **Cluster / Cloud / analytics** | **Generic** | Operational infrastructure. |

**STEP 2 conclusion:** the core is `Team → Channel → ChannelMember → Post` with an RBAC overlay.
The configuration surface, despite the highest git activity, is **not the domain core** — it is
a generic technical layer. This is the "activity ≠ domain significance" divergence explicitly
noted in the map (`repo-map.md:146`).

---

## STEP 3 — Aggregate candidates and their invariants

> Key distinction (confirmed in code): `IsValid()` in `model/` is a **structural declaration**
> (ID format, enum membership, field lengths) — **not** enforcement of a business invariant.
> Cross-entity invariants live in `server/channels/app/`.
> Status below: **enforces** / **declares** / **ignores**.

### Candidate A — Channel as the root of the membership aggregate (Channel + ChannelMembers)

| Invariant | Source citation | Enforcement status |
|---|---|---|
| A channel member may only be an **active member of the channel's Team** | `app/channel.go:1888-1903` (`GetMember(channel.TeamId, user.Id)`, `teamMember.DeleteAt > 0` → error) | **Enforced in the service layer, opt-out** — gated by the `skipTeamMemberIntegrityCheck bool` parameter (`channel.go:1887`). Usage verification: the only `skip=true` is `app/syncables.go:69` (group synchronization — the Team member was just added, the check is redundant); direct callers pass `false` (`channel.go:2653`, `team.go:721`). So **this is not a bypassed backdoor but a redundancy optimization** — yet the invariant is still not guaranteed *by construction* of the aggregate, only by call-site discipline. |
| A **Direct** channel has **exactly 2** members | no declaration in `model.Channel.IsValid`; enforced by the **signature** `createDirectChannelWithUser(user, otherUser)` `app/channel.go:466` + name encoding `userA__userB` `channel.go:348-351` | **Ignored at the aggregate level** — "2" follows from the function's argument count and the store, not from an entity invariant. |
| A **Group** channel has **3–8** members | `app/channel.go:561` (`len(userIDs) > ChannelGroupMaxUsers \|\| < ChannelGroupMinUsers`) | **Enforced** — but only at creation, in a service method, not in the entity. |
| A `group_constrained` channel → membership = linked group | flag `channel.go:100`; scattered enforcement: `app/channel.go:1802` (add), `app/channel.go:2915` (remove), `app/access_control.go:2098`, sync `app/group.go:409` | **Declares the flag, enforces via a scattered process** — consistency depends on the sync job, not the aggregate. |

### Candidate B — Post as the root of the thread aggregate

| Invariant | Source citation | Enforcement status |
|---|---|---|
| The thread's root post must be in the **same channel** as the reply | `app/post.go:299-301` (`!parentPostList.IsChannelId(post.ChannelId)` → error) | **Enforced in the app layer** — `model.Post.IsValid` only checks the `RootId` format (`post.go:500`). |
| The thread is **flat** (max 1 level — you cannot reply to a reply) | `app/post.go:304-306` (`if rootPost.RootId != ""` → error) | **Enforced in the app layer** — entirely unknown to the model. |

### Candidate C — Team as the root of the organizational membership aggregate

| Invariant | Source citation | Enforcement status |
|---|---|---|
| A `group_constrained` Team → membership = linked group | `app/team.go:686,762`; `app/access_control.go:2123` | **Enforced via a scattered process**, analogous to the channel. |
| Team type ∈ {`O`,`I`} | `model/team.go:174` | **Declares** (structurally, in `IsValid`). |

---

## STEP 4 — MODEL vs CODE divergences (the most valuable part)

> "Model" = what the entity declares about itself in `server/public/model/*.go`.
> "Code" = where the rule is actually (or not at all) enforced.

| # | Domain rule | MODEL says | CODE does | Evidence |
|---|---|---|---|---|
| 1 | A channel member must be a Team member | `Channel.IsValid` / `ChannelMember.IsValid` say **nothing** about Team — only ID format and NotifyProps | A check in a single service method, opt-out via the `skipTeamMemberIntegrityCheck` flag (the only `skip=true`: sync in `syncables.go:69`, redundantly) | `model/channel_member.go:129-147` (absent) vs `app/channel.go:1887-1903` + `syncables.go:69` |
| 2 | A reply and the thread root are in the same channel | `Post.IsValid` only checks `IsValidId(RootId)` | The service fetches the root and compares the channel | `model/post.go:500` vs `app/post.go:299-301` |
| 3 | Flat thread (no reply to a reply) | **silent** | The service rejects `rootPost.RootId != ""` | `model/post.go` (absent) vs `app/post.go:304-306` |
| 4 | DM = exactly 2 members | **silent** — `IsValid` does not count members (the `channel.go:348-351` block rejects *non-DM* channels whose name collides with the `__` pattern; it does not count DM members) | Enforced by the 2-argument signature `createDirectChannelWithUser(user, otherUser)` + store | `model/channel.go:311-386` (no counting) vs `app/channel.go:466` |
| 5 | GM = 3–8 members | **silent** (the constants `channel.go:37-38` exist, but `IsValid` does not use them) | Checked only at `createGroupChannel` | `model/channel.go:311` (absent) vs `app/channel.go:561` |
| 6 | `group_constrained` → membership == group | Declares **only** the flag + that it is meaningful for O/P (`channel.go:379-382`) | Consistency maintained by a scattered process + sync job (≥5 places) | `model/channel.go:380` vs `app/channel.go:1802,2915`, `access_control.go:2098`, `group.go:409` |

**The crux of the divergence:** the domain knowledge about *what a valid channel/thread is*
exists and is rich — but it lives in the **service verbs** (`app/`), not in the **domain nouns**
(`model/`). The entities are anemic: `IsValid()` guards field shape, not business truth. The
most dangerous case is #1 — a core invariant that can be *asked* to be skipped by passing `true`.

---

## STEP 5 — Refactor ranking (value × risk)

Value = how core the invariant is. Risk = how weakly it is enforced today
(ease of violation).

| Rank | Aggregate / invariant | Value | Risk | Rationale |
|---|---|---|---|---|
| **#1** | **Channel + ChannelMembers — "a channel member must be an active member of the Team"** | **Very high** (core: a channel belongs to a Team; a violation = access to conversations outside one's permissions) | **High (structural)** | The invariant is **not guaranteed by construction** — enforced in a single service method, opt-out via a flag (`app/channel.go:1887`); the aggregate does not guard it. Actual "bypass" risk is low (the only `skip=true` is redundant — `syncables.go:69`), but consistency depends on call-site discipline, not the entity. |
| #2 | Post/Thread — same-channel + thread flatness | High (conversation integrity) | Medium | Enforced, but scattered in `app/post.go` and entirely absent from the `Post` entity; easy to bypass with a new post-creation path. |
| #3 | GroupConstrained — membership == group | Medium (enterprise, supporting subdomain) | High | Consistency depends on ≥5 scattered places + a sync job; no single guardian. |
| #4 | GM 3–8 / DM = 2 | Medium | Low | Enforced structurally (signature/store); harder to violate accidentally. |

### #1 to refactor: the **Channel + ChannelMembers** aggregate around the Team-membership invariant

**Why #1:** it is the only candidate combining *highest value* (a core access-security rule)
× *highest risk* (enforcement in one place, deliberately toggleable by a boolean, without
aggregate/store support). The refactor would move the invariant from a toggleable service method
into the Channel aggregate root (an `AddMember`-style method that refuses to add a non-Team
member), so that the invariant is true **by construction** rather than maintained by service
call-site discipline. This sets up the next step in the chain directly
(`m4l5-2-invariant-aggregate-refactor`).

---

## Artifact limitations

- **No requirements document** (PRD/vision) — business rules reconstructed from the behavior
  of the `app/` layer code, not from intent written down by the team (STEP 0). A "declared
  invariant" = what the code enforces today.
- **Scope narrowed** to the communication core; the remaining 278 `model/` files (cloud,
  cluster, compliance, oauth, ...) were not analyzed in depth.
- `file:line` citations refer to the state of the `mattermost/` repo in this checkout (branch
  `module-4-lesson-5`); line numbers may shift after edits.
- **The store layer was not examined in depth** — it was not verified whether `store/sqlstore/`
  guarantees the membership invariant (e.g. via an FK). The DDD thesis ("the entity itself does
  not guard it") holds regardless, but a potential database-level guarantee remains unverified.
- Static analysis (grep + read) — no code or tests were run; runtime behavior, performance, and
  test coverage of the invariants were not examined.
