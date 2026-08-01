---
title: Invariant → aggregate refactor — ChannelMembership guardian
created: 2026-07-11
type: refactor-plan
---

# Invariant → aggregate refactor: the `ChannelMembership` guardian aggregate

> **This document is a PLAN, not an implementation.** No production code was modified.
> Every `file:line` below was opened and read in this session. Paths are relative to
> `mattermost/` (checkout on branch `module-4-lesson-5`); line numbers may drift after edits.
>
> Predecessor: `context/domain/01-domain-distillation.md`. That document ranked candidates.
> **I re-derived the invariant list and re-verified every citation independently** — the
> conclusion converges, but the diagnosis below is sharper than 01's and corrects it in two
> places (marked ⚠️).

---

## STEP 0 — Context discovered

**Requirements documents: none exist.** There is no PRD, vision, or `tech-stack.md` in this
repository. The only product narrative is `README.md:3` — an *"open core, self-hosted
collaboration platform that offers chat, workflow automation, voice calling, screen sharing and
AI integration"*. There are no written "business logic" or "success criteria" sections.

**Consequence for this plan (the key limitation):** business rules are reconstructed from *code
behavior*, not from written intent. "The domain requires X" below always means "the code
enforces X in at least one place", never "the team wrote down X". Where I infer product intent
(e.g. "a team is an access boundary"), I say so explicitly.

**Stack and layers where business logic lives** (verified from the tree):

| Layer | Path | What actually lives there |
|---|---|---|
| API (HTTP) | `server/channels/api4/` | Routing, param parsing, permission checks. Handlers are **not** thin — they carry authorization. |
| Application / service | `server/channels/app/` | **This is where the cross-entity invariants live.** Business rules are verbs on `*App`. |
| Domain model | `server/public/model/*.go` | Structs + `IsValid()`. ⚠️ `IsValid()` is **structural validation** (ID format, enum, lengths) — it holds no cross-entity business rule. The entities are **anemic**. |
| Persistence | `server/channels/store/sqlstore/` | SQL, plus an ops-facing integrity checker (`integrity.go`). |
| UI | `webapp/` (React/TS) | Renders what the API allows. |

The relevant architectural fact: **there is no aggregate layer.** A rule is either a check
inside an `*App` method, or it does not exist at runtime.

---

## STEP 1 — Business invariants identified

Rules that must always hold in this domain, extracted from code (no requirements doc exists to
extract from). Each has a verified citation.

| # | Invariant (as the code implies it) | Where the rule is expressed |
|---|---|---|
| **I-1** | **A channel member must be an active member of the channel's Team.** | `app/channel.go:1888-1903` — loads `Team().GetMember(channel.TeamId, user.Id)`; rejects on not-found and on `teamMember.DeleteAt > 0`. Gated by the parameter `skipTeamMemberIntegrityCheck` (`app/channel.go:1887`). |
| **I-2** | **A reply lives in the same channel as its thread root.** | `app/post.go:300` — `!parentPostList.IsChannelId(post.ChannelId)` → error. |
| **I-3** | **A thread is flat** (you cannot reply to a reply). | `app/post.go:304-306` — `if rootPost.RootId != ""` → error. |
| **I-4** | **A Group Message has 3–8 members.** | `app/channel.go:562` — `len(userIDs) > model.ChannelGroupMaxUsers \|\| < ChannelGroupMinUsers` → error. Constants at `public/model/channel.go:37-38`. |
| **I-5** | **A Direct Message has exactly 2 members.** | Not asserted anywhere. It follows from the *arity of a function signature*: `createDirectChannelWithUser(rctx, user, otherUser, ...)` `app/channel.go:466`. |
| **I-6** | **A group-constrained channel's membership equals its linked group.** | `app/channel.go:1802` — `if channel.IsGroupConstrained()` → `FilterNonGroupChannelMembers`. Maintained by a background sync process, not by one guard. |

**The shape of the whole list:** every one of these lives in `server/channels/app/` as a check
inside a service verb. **Not one of them is expressed on the entity it constrains.** That is the
structural finding — the domain knowledge is real and rich, but it sits in the verbs, not the
nouns.

---

## STEP 2 — Classification and the pick

Three axes, per the brief: **(a)** how core to the product's meaning, **(b)** how smeared across
layers, **(c)** whether it is *enforced* / merely *declared* / *violable*.

| # | (a) Core-ness | (b) Spread | (c) Enforcement status |
|---|---|---|---|
| **I-1** Channel member ⊆ Team member | **Highest.** A Team is the product's access boundary; a channel belongs to exactly one team. If this breaks, a user reads conversations of a team they are not in. This is an **access-control** rule, not a tidiness rule. | **Worst: 2 aggregates, 3 files, both directions.** Add-side in `app/channel.go`, remove-side in `app/team.go`, group-sync opt-out in `app/syncables.go`, and *not covered* by `store/sqlstore/integrity.go`. | **Violable.** Enforced at add-time only, and that check is **switchable off by a boolean parameter**. The remove-side is a procedural cleanup with no transaction and no owner. **No DB constraint.** |
| **I-2** reply in root's channel | High (thread integrity) | Low — one place, `app/post.go`. | Enforced (unconditionally). |
| **I-3** flat thread | High (thread integrity) | Low — one place, `app/post.go`. | Enforced (unconditionally). |
| **I-4** GM 3–8 | Medium | Low — one place, at creation. | Enforced at creation. Nothing re-checks after members are added later. |
| **I-5** DM = 2 | Medium | Low | **Only declared, structurally** — guaranteed by a 2-argument signature. Fragile in principle, but hard to violate by accident. |
| **I-6** group-constrained ≡ group | Medium (supporting subdomain — identity integration, not chat) | High — ≥3 places + a sync job. | Eventually-consistent by design. A *process*, not an invariant. Refactoring it means redesigning sync — out of proportion here. |

### Pick: **I-1 — "a channel member must be an active member of the channel's Team"**

It is the only candidate that is **simultaneously the most core and the most weakly enforced**.
I-2/I-3 are more core-adjacent but are *already properly enforced* (one place, no opt-out) —
they would be a cosmetic refactor. I-6 is badly smeared but sits in a **supporting** subdomain
and is *designed* to be eventually consistent. I-1 is the one where high value and weak
enforcement actually coincide.

**What raises I-1 from "consistency drift" to "security defect"** — and this is the finding
that decides the plan — is that **the read path never re-checks team membership**:

```
app/authorization.go:466  HasPermissionToReadChannel(userID, channel)
app/authorization.go:467    → HasPermissionToChannel(userID, channel.Id, PermissionReadChannelContent)
app/authorization.go:337-347  → reads ChannelMembers roles → RolesGrantPermission → return true
```

`HasPermissionToChannel` (`app/authorization.go:327-347`) decides purely from the user's
**ChannelMembers** rows. Team membership is consulted **only as a fallback for users who are
*not* members**, and only for open channels (`app/authorization.go:471-472`). So for a
**private** channel, a row in `ChannelMembers` is, by itself, a permanent grant of read access.

**Therefore: an orphaned `ChannelMember` row is not a stale record. It is a standing grant of
read/write access to a private conversation in a team the user has left.** That is what the
aggregate must make unrepresentable.

---

## STEP 3 — Diagnosis of I-1

### 3.1 Where the rule lives today (all layers)

**Add-side — enforced, but switchable off.**

`app/channel.go:1887-1903`:
```go
func (a *App) AddUserToChannel(rctx request.CTX, user *model.User, channel *model.Channel, skipTeamMemberIntegrityCheck bool) (*model.ChannelMember, *model.AppError) {
	if !skipTeamMemberIntegrityCheck {
		teamMember, nErr := a.Srv().Store().Team().GetMember(rctx, channel.TeamId, user.Id)
		...
		if teamMember.DeleteAt > 0 {
			return nil, model.NewAppError("AddUserToChannel", "api.channel.add_user.to.channel.failed.deleted.app_error", ...)
		}
	}
	newMember, err := a.addUserToChannel(rctx, user, channel)   // :1905
	...
}
```

The invariant is a **parameter**. It is promoted into a public options struct — so any caller,
including plugins, expresses it as data:

`app/channel.go:1927-1935`:
```go
type ChannelMemberOpts struct {
	UserRequestorID string
	PostRootID      string
	// SkipTeamMemberIntegrityCheck is used to indicate whether it should be checked
	// that a user has already been removed from that team or not.
	SkipTeamMemberIntegrityCheck bool
}
```
…and `AddChannelMember` (`app/channel.go:1938`) forwards it verbatim at `app/channel.go:1966`.

**Verified production call sites** (`_test.go` excluded):

| Call site | Skip? | Verdict |
|---|---|---|
| `app/channel.go:2653` (`JoinChannel`) | `false` | Checked. |
| `app/team.go:721` | `false` | Checked. |
| `app/syncables.go:69` | **`true`** | The **only** `true` in production. The team member was just created by the same sync, so the check is redundant — **not** a malicious backdoor. |

⚠️ **Correction to doc 01's framing:** the private `addUserToChannel` (`app/channel.go:1787`) is
**not** an independent bypass path. Its only production caller is `app/channel.go:1905`, i.e.
inside the checked public method, after the check. I verified this by grep. The real holes are
the two below, not a rogue call path.

**Hole 1 — the invariant is opt-out by design.** Today's single `true` is benign. But the rule's
*truth* rests on **call-site discipline**, not on construction. Nothing stops the next caller —
or a plugin — from passing `true` and writing an orphan row. A domain invariant that takes a
`bool` to disable it is not an invariant; it is a **default**.

**Remove-side — a procedural cleanup, not an enforcement, and not atomic.**

`app/team.go:1363` `LeaveTeam` is the only thing keeping I-1 true when a user leaves a team. It
performs, in sequence and **with no transaction**:

1. `Channel().GetChannels(team.Id, user.Id, ...)` — read the user's channels in the team.
2. a `for` loop calling `a.removeChannelMembership(rctx, user.Id, channel.Id, "LeaveTeam")` per
   channel (DMs/GMs skipped — correct, they have no team).
3. `a.ch.srv.teamService.RemoveTeamMember(rctx, teamMember)` — drop the team membership.
4. `a.postProcessTeamMemberLeave(...)` (`app/team.go:1318`) — cache invalidation, sidebar,
   preferences.

`removeChannelMembership` itself (`app/channel.go:2886-2894`) is **two independent store writes**
with no transaction:
```go
if err := a.Srv().Store().Channel().RemoveMember(rctx, channelID, userID); err != nil { ... }
if err := a.Srv().Store().Thread().DeleteMembershipsForChannel(userID, channelID); err != nil { ... }
```

**Hole 2 — N+1 writes, no transaction, no owner.** Removing a user from a team with 40 channels
is 80+ separate writes with no atomic boundary. The *ordering* is defensively chosen (channels
first, then the team member), so a mid-loop crash leaves the user still on the team — the safe
direction. But the sequence is not atomic, and **nothing owns the pair**.

**A concrete way I-1 breaks — labeled as a scenario, not observed behavior.** I cannot prove the
absence of a lock from static reading, so I state this as the failure mode the design permits,
not as a bug I reproduced:

> Request A (`LeaveTeam`) reads the user's channel list at step 1 and begins removing memberships.
> Concurrently, request B (`AddChannelMember` on a private channel in that team) runs its
> add-time check — the team membership **still exists**, so the check passes — and writes a new
> `ChannelMembers` row. Request A then completes step 3 and deletes the team membership. The row
> written by B is now an orphan. Per §2, that row is a **standing read grant on a private channel
> for a non-member of the team.** Nothing later revokes it.

**Hole 3 — the cleanup's *input* is a query whose "not found" is swallowed into "nothing to do".**
The first step of `LeaveTeam` (`app/team.go:1371-1381`):

```go
if channelList, nErr = a.Srv().Store().Channel().GetChannels(team.Id, user.Id, ...); nErr != nil {
	var nfErr *store.ErrNotFound
	if errors.As(nErr, &nfErr) {
		channelList = model.ChannelList{}   // ← ErrNotFound reinterpreted as "user has no channels"
	} else {
		return model.NewAppError("LeaveTeam", ...)
	}
}
```

Today this is **almost certainly benign** — the store returns `ErrNotFound` for an empty result
set, so "not found" really does mean "no channels". But note what it structurally is: **the sole
guard of I-1's remove-side treats a failed lookup as an empty lookup, and then proceeds to delete
the team membership anyway.** If that query ever returns `ErrNotFound` for any reason other than
genuine emptiness, every one of the user's channel memberships is orphaned *silently and
deterministically* — no error, no log, no detector (see below). The cleanup would report success
having cleaned nothing. This is a load-bearing assumption held together by a store-layer
convention, and the aggregate design removes it: after the refactor the deletion is a set-based
statement in the same transaction, so "no rows matched" and "the lookup failed" can no longer be
confused.

**Batch callers log-and-continue (affects I-6, not I-1).** `DeleteGroupConstrainedTeamMemberships`
(`app/syncables.go:161-183`) loops over users to evict and calls `a.RemoveUserFromTeam` — which
correctly routes through `LeaveTeam`, so channel cleanup *does* happen per user. But on failure it
appends to a `multierror` and **`continue`s** (`syncables.go:174-178`). Per-user state stays
consistent, so **I-1 survives** — but a user who *should* have been evicted from a
group-constrained team silently remains, and the job reports partial success. That is a fail-fast
violation against **I-6**, and I record it here rather than fold it into I-1's diagnosis, because
conflating them would overstate the case I am making.

**Persistence layer — enforces nothing, and cannot even *see* the violation.**

There is **no foreign key** from `ChannelMembers` to `TeamMembers` (there could not be a simple
one — the relationship is transitive through `Channels`). Worse, the ops-facing integrity
checker does not cover it. Verified in `store/sqlstore/integrity.go`:

| Check present | Line |
|---|---|
| `Channels` → `ChannelMembers` | `integrity.go:103-105` |
| `Users` → `ChannelMembers` | `integrity.go:302-304` |
| `Teams` → `TeamMembers` | `integrity.go:265-267` |
| **`Teams` → `ChannelMembers`** | **absent** |

So a violation of I-1 is **not detectable by the tool built to detect exactly this class of
problem.** There is no reconciliation job and no alarm. The state, once wrong, stays wrong
silently.

### 3.2 Diagnosis summary

| Layer | Does it enforce I-1? |
|---|---|
| UI (`webapp/`) | No — and it doesn't need to. ✅ *This is not a "client is the only guard" case.* |
| API (`api4/`) | **No.** `addChannelMember` (`api4/channel.go:2289+`) parses `user_ids`/`post_root_id` and checks *permissions*; it never mentions the team invariant. |
| App (`app/`) | **Partially.** Add-side: yes, but opt-out. Remove-side: a non-atomic cleanup in a different file, on a different aggregate. |
| Domain model (`model/`) | **No.** `ChannelMember.IsValid()` knows nothing about Teams. The rule is invisible to the type. |
| Persistence | **No.** No FK, no integrity check, no reconciliation. |

⚠️ **A second correction to the expected narrative — on fail-fast, be precise.** The brief expects
"errors swallowed instead of stopping the operation". On I-1's *main* paths that is **not** what I
found: `AddUserToChannel` and the `LeaveTeam` removal loop both propagate their errors and abort.
The primary defect is **non-atomicity plus the absence of an owner**, not log-and-continue. There
are exactly two genuine swallows, and I scope them honestly rather than inflate them:
- **Hole 3** — `ErrNotFound` reinterpreted as "no channels" in the one query the cleanup depends
  on (`app/team.go:1375-1377`). Latent, not currently firing.
- **`app/syncables.go:174-178`** — `continue`-on-error in the group-sync eviction batch. Real, but
  it degrades **I-6**, not I-1.

**The one-sentence diagnosis:** *I-1 is a core access-control invariant that no object owns — a
toggleable precondition on one aggregate, plus a non-atomic, unserialized cleanup on another,
with no backstop in the database, no detector in ops tooling, and a permission read path that
treats the resulting orphan row as a valid grant.*

---

## STEP 4 — Design: the `ChannelMembership` guardian aggregate

**Design constraint — must fit Go + Mattermost's App/Store layering.** I am not proposing a Java
DDD transplant. Concretely: a new domain package holding the rule as a *type*, a store method
that loads and saves it *as a unit and in one transaction*, and `*App` methods that become thin
delegates. This survives the existing plugin API and `api4` untouched at the signature level.

### 4.1 The aggregate root

New package `server/channels/app/membership/` (domain, no store/HTTP imports — dependency-free,
therefore unit-testable without a DB):

```go
package membership

// TeamMembership is the guarded fact: this user's standing in this team, right now.
type TeamMembership struct {
	teamID   string
	userID   string
	deleteAt int64
}

func (tm TeamMembership) IsActive() bool { return tm.deleteAt == 0 }

// ChannelMembership is the AGGREGATE ROOT for invariant I-1.
// It is constructed only from a channel + the actor's team standing, so a
// ChannelMembership that violates I-1 CANNOT BE CONSTRUCTED.
type ChannelMembership struct {
	channelID string
	teamID    string
	userID    string
	roles     string
}
```

### 4.2 Named domain errors (fail-fast, no silent state change)

```go
type ErrNotTeamMember struct{ UserID, TeamID string }   // never was a member
type ErrTeamMembershipRevoked struct{ UserID, TeamID string } // was, DeleteAt > 0
type ErrOrphanedMembership struct{ UserID, ChannelID string } // detected at load
```
Each implements `error`. **Nothing in this package logs.** Each illegal operation *returns*.

### 4.3 The domain method with its preconditions

```go
// Grant is the ONLY way to produce a ChannelMembership.
// Preconditions (I-1) — no parameter can disable them:
//   P1: channel.TeamId == teamMembership.teamID   (right team)
//   P2: teamMembership.IsActive()                 (active, not soft-deleted)
func Grant(channel *model.Channel, tm TeamMembership, roles string) (*ChannelMembership, error) {
	if channel.TeamId != tm.teamID {
		return nil, &ErrNotTeamMember{UserID: tm.userID, TeamID: channel.TeamId}
	}
	if !tm.IsActive() {
		return nil, &ErrTeamMembershipRevoked{UserID: tm.userID, TeamID: tm.teamID}
	}
	return &ChannelMembership{channelID: channel.Id, teamID: tm.teamID, userID: tm.userID, roles: roles}, nil
}
```

**The load-bearing change:** the `skipTeamMemberIntegrityCheck` boolean **has no counterpart
here, and deliberately so.** The sync case (`app/syncables.go:69`), which passes `true` today
purely to avoid a redundant read, is served instead by *handing in the `TeamMembership` it just
created* — it satisfies the precondition with the value it already holds, rather than asking to
skip it. **The optimization survives; the opt-out does not.** That is the whole point of the
refactor: I-1 stops being a default and becomes a constructor.

DM/GM channels (`ChannelType` `D`/`G`) have no `TeamId`; they route to a separate
`GrantDirect(...)` constructor that carries I-4/I-5 instead — so a teamless channel can never
silently fall through the I-1 check.

### 4.4 Repository — loads and saves the aggregate as a unit, in ONE transaction

Today `LeaveTeam` issues N+1 un-bounded writes across two stores. The invariant needs atomicity,
so it gets a transaction. **The primitive already exists in this codebase** — the store uses
`GetMaster().Begin()` + `defer finalizeTransactionX(transaction, &err)`, e.g.
`store/sqlstore/channel_store.go:634-639`. The new method follows that exact pattern:

```go
// store/store.go (interface)
type ChannelStore interface {
	...
	// GrantMembership applies the invariant and writes, inside ONE transaction,
	// holding a row lock on the actor's TeamMembers row. Either a legal membership
	// is written, or a named domain error is returned and nothing is written.
	GrantMembership(rctx request.CTX, channelID, userID string, rule membership.GrantFunc) (*model.ChannelMember, error)

	// RevokeAllForTeamMember removes the team membership AND every channel
	// membership in that team, atomically, taking the SAME row lock.
	// Replaces the LeaveTeam loop.
	RevokeAllForTeamMember(rctx request.CTX, teamID, userID string) error
}
```

Both methods lock the same `TeamMembers` row. **That shared lock — not the aggregate type — is
what makes I-1 true under concurrency.**

`RevokeAllForTeamMember` runs, inside a single transaction:
1. `SELECT … FROM TeamMembers WHERE TeamId=? AND UserId=? FOR UPDATE` — **take the row lock first**
   (see below);
2. `DELETE FROM ChannelMembers` for every non-DM/GM channel of that team (**one set-based
   statement**, replacing the N-iteration loop);
3. the corresponding `ThreadMemberships` delete (today the un-transacted second write in
   `app/channel.go:2887-2892`);
4. the `TeamMembers` update/delete.

Either all of it lands or none of it does. **This closes Hole 2** — the crash/partial-state hole.

#### ⚠️ Atomicity on the revoke side alone does NOT close the race — the add side must lock too

This is the subtlest part of the design and the easiest thing to get wrong. Wrapping *only*
`RevokeAllForTeamMember` in a transaction leaves the §3.1 TOCTOU scenario **fully intact**. At
the default isolation level of both supported databases (`READ COMMITTED`), the two transactions
do not conflict on any row, so nothing serializes them:

```
Revoke TX:  DELETE FROM ChannelMembers WHERE …   -- the Add's row does not exist yet → deletes nothing
            DELETE FROM TeamMembers     WHERE …
            COMMIT
Add TX:     read TeamMembers → still active (read before Revoke committed)
            INSERT INTO ChannelMembers …          -- commits AFTER the DELETE already ran
            COMMIT
                                                  → orphan row survives. Invariant broken.
```

The `DELETE` cannot remove a row that had not been inserted yet, and the `INSERT` was authorized
by a read of a team membership that was about to vanish. **Atomicity is not mutual exclusion.**

Therefore the **add path must also be a single transaction, and `GrantMembership` must take
`SELECT … FOR UPDATE` on the `TeamMembers` row** — the same row `RevokeAllForTeamMember` locks in
its step 1. That row becomes the **serialization point** for the invariant: Add and Revoke now
contend on it, and whichever runs second sees the other's committed truth. Concretely:

- Revoke commits first → Add's `FOR UPDATE` read blocks, then returns the *revoked* membership →
  `Grant` returns `ErrTeamMembershipRevoked` → **no row is written.**
- Add commits first → Revoke's `DELETE FROM ChannelMembers` blocks, then runs *after* the insert
  is visible → **the new row is deleted with the rest.**

Either interleaving preserves I-1. (`SERIALIZABLE` isolation would also work, but it would impose
a retry-on-conflict burden on this very hot path; a single row lock is the cheaper and more
targeted choice.)

**This is why the aggregate needs a repository and not just a constructor:** the rule is only as
true as the boundary the two writes share. `membership.Grant` makes the invariant *unrepresentable
in memory*; the row lock is what makes it *unrepresentable in the database*. Both halves are load-
bearing — shipping Phase 1 without Phase 2's lock would produce a design that reads correct and
still races.

##### Premise of the lock (verified) — and the one path where it does not hold

`SELECT … FOR UPDATE` only serializes if the row **still exists** after removal. Verified: the
normal leave path **soft-deletes**. `teamService.RemoveTeamMember` (`app/teams/teams.go:248-251`)
does:

```go
teamMember.Roles = ""
teamMember.DeleteAt = model.GetMillis()
if _, nErr := ts.store.UpdateMember(rctx, teamMember); nErr != nil { return nErr }
```

The `TeamMembers` row **persists with `DeleteAt > 0`** — which is exactly what today's check at
`app/channel.go:1897` already keys on. So the row is there to be locked, and the design holds. ✅

**But there is a hard-delete path.** `SqlTeamStore.RemoveMembers` (`store/sqlstore/team_store.go:
1262-1266`) and `RemoveAllMembersByTeam` (`team_store.go:1285-1288`) issue a real
`DELETE FROM TeamMembers` (used by permanent team/user deletion, not by leave). On those paths the
row vanishes, so **there is nothing left for a concurrent `Grant` to block on** — and a `FOR UPDATE`
against a deleted row does not wait. For permanent deletion the aggregate must therefore serialize
on a durable row instead: lock the parent **`Teams`** row, or take a `pg_advisory_xact_lock` keyed
on `(teamID, userID)`. I flag this rather than hand-wave it, because it is the same class of
mistake as the one this section exists to correct: *a lock that has nothing to hold is not a lock.*

### 4.5 Thin API / thin App

`*App` methods keep their signatures (plugin API compatibility) but become **parse → aggregate →
map error**:

```go
func (a *App) AddChannelMember(rctx request.CTX, userID string, channel *model.Channel, opts ChannelMemberOpts) (*model.ChannelMember, *model.AppError) {
	// GrantMembership owns the whole critical section: it opens ONE transaction,
	// loads the team standing FOR UPDATE, applies the rule, and writes — or writes nothing.
	// The rule cannot be evaluated on a snapshot that a concurrent LeaveTeam can invalidate.
	saved, err := a.Srv().Store().Channel().GrantMembership(rctx, channel.Id, userID, membership.Grant)
	if err != nil {
		return nil, toAppError(err)   // ← named domain error, fail-fast, nothing written
	}

	a.postAddMemberSideEffects(rctx, saved, opts)   // websockets, sidebar — AFTER the fact commits
	return saved, nil
}
```

Inside the store, following the existing `channel_store.go:634-639` pattern:

```go
func (s SqlChannelStore) GrantMembership(rctx request.CTX, channelID, userID string, rule membership.GrantFunc) (cm *model.ChannelMember, err error) {
	transaction, err := s.GetMaster().Begin()
	if err != nil { return nil, errors.Wrap(err, "begin_transaction") }
	defer finalizeTransactionX(transaction, &err)

	channel, err := s.getChannelT(transaction, channelID)
	if err != nil { return nil, err }

	// FOR UPDATE — the serialization point shared with RevokeAllForTeamMember.
	teamStanding, err := s.getTeamMembershipForUpdateT(transaction, channel.TeamId, userID)
	if err != nil { return nil, err }

	granted, err := rule(channel, teamStanding, defaultRolesFor(channel))   // ← the invariant
	if err != nil { return nil, err }                                       // ← rollback, no write

	saved, err := s.saveMemberT(transaction, granted)
	if err != nil { return nil, err }
	return saved, transaction.Commit()
}
```

The domain rule (`membership.Grant`) is **injected as a function**, so the aggregate package keeps
zero store dependencies and stays unit-testable without a database — while the store supplies the
transactional boundary the rule needs to actually hold.

`toAppError` maps domain errors → HTTP once, in one place:

| Domain error | AppError id | HTTP |
|---|---|---|
| `ErrNotTeamMember` | `app.team.get_member.missing.app_error` | 404 (preserves today's status — no API contract break) |
| `ErrTeamMembershipRevoked` | `api.channel.add_user.to.channel.failed.deleted.app_error` | 400 (preserves today's status) |
| `ErrOrphanedMembership` | `app.channel.membership.orphaned.app_error` | 500 |

**Enforcement locus does not move from client to server — it was already on the server.** It
moves from *scattered service verbs* into *one constructor + one transaction*. That is the
correct statement of this refactor, and I want it stated accurately rather than dramatised.

---

## STEP 5 — Before/after, phases, tests

### 5.1 Before → after, per site where the rule lives today

| Site (verified) | Before | After |
|---|---|---|
| `app/channel.go:1887-1903` | `AddUserToChannel(..., skipTeamMemberIntegrityCheck bool)` — the invariant is a parameter | Signature keeps the `bool` (plugin API), but it is **ignored and marked deprecated**; the check is unconditional inside `membership.Grant`. Deleted in the final phase. |
| `app/channel.go:1927-1935` | `ChannelMemberOpts.SkipTeamMemberIntegrityCheck` | Field deprecated → removed. The sync optimization is preserved by **passing the `TeamMembership`**, not by skipping. |
| `app/channel.go:1966` | forwards `opts.SkipTeamMemberIntegrityCheck` | forwards nothing; calls `membership.Grant`. |
| `app/syncables.go:69` | `SkipTeamMemberIntegrityCheck: true` | Passes the `TeamMembership` it just created → precondition satisfied, **zero extra reads**, no opt-out. |
| `app/team.go:1363` `LeaveTeam` — the loop | Read channels → N× `removeChannelMembership` → `RemoveTeamMember`, **no transaction** | One call: `store.Channel().RevokeAllForTeamMember(teamID, userID)`. Atomic. Loop deleted. |
| `app/team.go:1375-1377` (Hole 3) | `ErrNotFound` on the channel lookup → silently treated as "no channels", team member deleted anyway | Gone. The cleanup is a set-based `DELETE` in the same transaction — "zero rows matched" and "the lookup failed" can no longer be confused. |
| `app/channel.go:2886-2894` `removeChannelMembership` | 2 un-transacted store writes | Both writes folded into the aggregate's transaction. The helper survives only for the *single-channel* leave path. |
| `app/syncables.go:174-178` | `continue`-on-error while evicting group-constrained team members | **Out of scope — I-6, not I-1.** Recorded, not planned. Flagged so it is not mistaken for fixed. |
| `app/authorization.go:327-347` | Grants read from `ChannelMembers` alone | **Unchanged — and that becomes correct, but only once Phase 2 ships.** Trusting `ChannelMembers` is sound *iff* an orphan row cannot exist, which requires the shared row lock (§4.4), not merely the aggregate type. *The refactor's payoff is that it makes the existing hot read path safe **without adding a read to it** — the cost is paid once on write, not on every permission check.* |
| `store/sqlstore/integrity.go` | No `Teams → ChannelMembers` check | **New check added** — so any pre-existing orphan (from before the refactor) becomes visible. |
| `model/channel_member.go` `IsValid()` | Structural only | Unchanged. Structural validation stays structural; business rules go to the aggregate, not into `IsValid()`. |

### 5.2 Phased plan

The project has a **real test discipline** — `_test.go` files sit beside virtually every `app/`
and `store/` file, with an established `TestHelper` harness (`channels/api4/apitestlib.go`,
`channels/app/…/helper_test.go`) and a Go runner (`go test ./...` / `make test`). So phases 0–3
are genuinely **test-first**; there is a runner to be red against.

| Phase | Work | Test-first? |
|---|---|---|
| **0 — Expose the truth** | Add the `Teams → ChannelMembers` check to `store/sqlstore/integrity.go`. Ship it alone. **Does the violation already exist in production data?** Answer that before refactoring — it sizes the migration. | **Yes** — write the failing check test first, seeding a hand-made orphan. |
| **1 — The rule, in isolation** | Create `app/membership/` (aggregate, `Grant`, named errors). No caller changes. Pure unit tests, no DB. | **Yes.** Table-driven; this is where §5.3 lives. |
| **2 — Atomicity + mutual exclusion** | `GrantMembership` + `RevokeAllForTeamMember` in the store, using `GetMaster().Begin()` + `finalizeTransactionX` per `channel_store.go:634-639`, **both taking `SELECT … FOR UPDATE` on the same `TeamMembers` row**. Rewrite `LeaveTeam` to call it. **The lock ships with this phase or the refactor does not hold — see §4.4.** | **Yes** — a rollback test *and* the concurrency test (§5.3 #11). |
| **3 — Route the callers** | `AddChannelMember` / `AddUserToChannel` delegate to `GrantMembership`. The `bool` and the opts field are **ignored + deprecated**, not yet deleted — keeps the plugin API green. Rewrite `syncables.go:69` to pass the `TeamMembership`. | **Yes** — existing `app`/`api4` suites must stay green; they are the regression net. |
| **4 — Backfill** | Migration/job reconciling orphans that Phase 0 found. **Deletion of an access grant is not reversible — this phase needs a human decision on policy, not a default.** | Idempotency test. |
| **5 — Remove the opt-out** | Delete `skipTeamMemberIntegrityCheck` and `ChannelMemberOpts.SkipTeamMemberIntegrityCheck`. **Breaking change to the plugin-facing API — must ride a major release.** | Compile-time. |

Phases 0–3 are independently shippable and each is a net improvement if the chain stops there.
Phase 5 is the only one that breaks a contract.

### 5.3 Test cases for I-1

**Legal (must succeed):**
1. Active team member added to a public channel of that team → membership granted.
2. Active team member added to a private channel of that team → granted.
3. Group-sync path (`syncables`): team member created, then channel membership granted in the
   same flow, **without any extra `Team().GetMember` read** → granted. *(Guards the performance
   reason the skip flag existed — the optimization must survive.)*
4. DM/GM (`D`/`G`, no `TeamId`) → routed to `GrantDirect`, granted, I-1 not applied.

**Illegal (must return a named error and write nothing):**
5. User who was never a team member → `ErrNotTeamMember`. **Assert no `ChannelMembers` row was
   written** — not merely that an error came back.
6. User whose team membership is soft-deleted (`DeleteAt > 0`) → `ErrTeamMembershipRevoked`.
7. Channel from team A + team membership in team B → `ErrNotTeamMember` (the P1 mismatch — this
   is the case *today's code cannot even express*, because it never compares the two team IDs).
8. **Regression test for the removed opt-out:** no code path, given any input, produces a
   `ChannelMembership` whose `teamID` ≠ the channel's `TeamId`. With `Grant` as the only
   constructor, this is enforced by the type system; the test documents the intent.

**Atomicity (Phase 2):**
9. `RevokeAllForTeamMember` with a forced failure on the final `TeamMembers` write → **all**
   `ChannelMembers` rows still present (full rollback). No partial state.
10. `LeaveTeam` on a user in N channels → after commit, zero `ChannelMembers` rows in that team,
    zero `ThreadMemberships`, team membership gone. One transaction.
11. **The §3.1 TOCTOU scenario, as an explicit concurrency test:** run `LeaveTeam` and
    `AddChannelMember` for the same user against a real DB, both orderings, repeated under load.
    Assert the terminal state is **never** "no team membership + a surviving channel membership" —
    only the two legal outcomes of §4.4 (add rejected with `ErrTeamMembershipRevoked`, or the added
    row swept by the revoke).
    - Expected **red on today's code** — that is what makes it worth writing.
    - Expected **still red after Phase 1 alone** (the aggregate type cannot stop a race), and
      **green only once Phase 2 ships the shared `FOR UPDATE` lock.** This test is the executable
      proof of §4.4; if it passes without the lock, the test is not actually racing and should be
      distrusted before the design is.

### 5.4 New load-bearing names to register

The repo has **no `docs/reference/contract-surfaces.md`** (verified — `/10x-init` has not been
run here, only `context/foundation/README.md` exists). So there is nowhere to register these
yet. If the registry is created, these are the names this refactor makes load-bearing:

| Name | Kind | Why load-bearing |
|---|---|---|
| `membership.ChannelMembership` | Aggregate root | The single guardian of I-1. |
| `membership.TeamMembership` | Value object | The precondition input; carries the active/revoked distinction. |
| `membership.Grant()` | Constructor | **The only** way to create a channel membership. |
| `membership.ErrNotTeamMember` / `ErrTeamMembershipRevoked` / `ErrOrphanedMembership` | Domain errors | The fail-fast vocabulary; API status mapping depends on them. |
| `ChannelStore.GrantMembership()` / `.RevokeAllForTeamMember()` | Repository contract | The atomic boundary — both take the **same** `TeamMembers` row lock. |
| `checkTeamsChannelMembersIntegrity()` | Ops contract | The detector that does not exist today. |
| ~~`skipTeamMemberIntegrityCheck`~~ / ~~`ChannelMemberOpts.SkipTeamMemberIntegrityCheck`~~ | **Retired** | Plugin-facing; removal is a breaking change (Phase 5). |

---

## Limitations of this artifact

- **No requirements document exists** — I-1…I-6 are reconstructed from code behavior. The claim
  "a team is an access boundary" is my inference from `README.md:3` + the authorization code, not
  a quoted product decision. *If the team's actual intent is that leaving a team should preserve
  channel access, this entire plan is solving the wrong problem — that is the one assumption a
  human must confirm before Phase 1.*
- **The TOCTOU scenario in §3.1 is a design-permitted failure mode, not an observed bug.** I read
  statically; I did not run the server, reproduce a race, or inspect production data. Absence of a
  lock cannot be proven by reading. Phase 0 exists precisely to replace this inference with data,
  and test §5.3 #11 is written to make the race either reproduce or falsify itself.
- **I searched for a *deterministic* orphan path and did not find one.** I traced every production
  writer of `TeamMembers` (`app/team.go:1415` via `LeaveTeam`; `app/channel.go:2948`, the guest
  last-channel eviction; `app/syncables.go:174` via `RemoveUserFromTeam`) — all route through the
  channel cleanup. So I-1's breakage today is **latent** (race + opt-out + Hole 3), not a standing
  bug I can point at. **The refactor's value is that it makes a class of defect unrepresentable, not
  that it fixes a known incident** — and the plan should be sold that way, not oversold.
- **The isolation-level claim in §4.4 is reasoned, not measured.** I assert that `READ COMMITTED`
  does not serialize the two paths; I did not run the interleaving against Postgres/MySQL to
  confirm it. It follows from standard semantics, but Phase 2's test is what should establish it.
  The *soft-delete premise* the lock rests on, by contrast, **is** verified
  (`app/teams/teams.go:248-251`) — as is the hard-delete exception (`team_store.go:1262-1266`).
- **No code was executed** — no tests run, no build. Every citation was read; none was executed.
- The **plugin API surface** (`public/plugin/api.go:590`) exposes `AddUserToChannel`; I confirmed
  the signature but did not audit plugin-side consumers of the opts struct. Phase 5's blast radius
  is therefore estimated, not measured.
- Scope is I-1 only. I-2…I-6 are diagnosed but deliberately not planned.
