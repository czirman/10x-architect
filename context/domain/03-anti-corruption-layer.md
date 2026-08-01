---
title: Anti-Corruption Layer — go-opengraph as a leaking dependency
created: 2026-07-12
type: refactor-plan
---

# Anti-Corruption Layer — `go-opengraph` as a leaking dependency

> The product of this document is a **refactor PLAN**, not an implementation. No production code
> was modified. The #1 dependency was **discovered** (by measuring imports per layer), not assumed
> up front. Every `file:line` citation was verified against the `mattermost/` checkout (branch
> `module-4-lesson-5`).

---

## STEP 0 — Project context

**Stack.** Mattermost monorepo: Go server (`server/go.mod`, `go 1.26.3`) + TypeScript/React webapp
(`webapp/channels`, `webapp/platform`). Server-side layers (consistent with the map in
`context/domain/01-domain-distillation.md`):

| Layer | Path | Role |
|---|---|---|
| Domain + wire contract | `server/public/model/` | Entities, `IsValid()`, **the shape of the JSON sent to the client** |
| Application (service) | `server/channels/app/` | Cross-entity logic |
| Persistence | `server/channels/store/` | Data access |
| API | `server/channels/api4/` | HTTP |
| UI | `webapp/` | React |

**Replaceability claims — LIMITATION (reported honestly).** `grep -niE "swap|pluggable|
interchangeab|replace the|drop-in"` over `README.md`, `AGENTS.md`, `server/README.md`,
`webapp/README.md` returned **no** declaration that any component is meant to be swappable. There
is also no PRD and no tech-stack.md (confirming the finding in `01-domain-distillation.md:22-30`).

This does **not** weaken the finding — quite the opposite. We do not have an "intent-vs-code"
divergence here (a document saying "swappable" while the code makes it impossible). We have
something worse: an **accidentally published contract** — a data shape that a third-party library
imposed on the public API and on the database, where **nobody ever decided it should be that way**.
There is no document to cite because there was no decision. The contract came into being through a
leak, not through design.

---

## STEP 1 — IDENTIFYING the leaking dependencies

Measurement: third-party imports inside `server/public/model/*.go` (the contract layer — a library
that lives here is automatically visible to every layer above **and to the client**):

```
10  github.com/pkg/errors
 4  github.com/tinylib/msgp/msgp
 4  github.com/goccy/go-yaml
 3  github.com/vmihailenco/msgpack/v5
 2  github.com/gorilla/websocket
 8  github.com/dyatlov/go-opengraph/...   ← library types in domain fields and signatures
 1  github.com/mattermost/ldap
```

`pkg/errors`, `msgp`, `msgpack`, `go-yaml` are **infrastructure/serialization** libraries — they do
not impose a shape on a domain entity. `go-opengraph` is the only one whose **data structure**
becomes an entity field and a type in domain signatures.

### Every file that "knows" `github.com/dyatlov/go-opengraph` today

(complete list; `grep -rn "dyatlov/go-opengraph" server --include=*.go`, excluding `_test.go`)

| File:line | Layer | What exactly it knows |
|---|---|---|
| `server/go.mod:22` | manifest | `go-opengraph v0.0.0-20220524092352-606d7b1e5f8a` — **pseudo-version, last commit May 2022** |
| `server/public/model/link_metadata.go:17-18` | **contract/domain** | imports `opengraph` + `opengraph/types/image` |
| `server/public/model/link_metadata.go:43,45` | **contract/domain** | `Data any` — comment: "should contain (…) `*opengraph.OpenGraph`" |
| `server/public/model/link_metadata.go:57` | domain | `firstNImages(images []*image.Image, …)` — a domain rule over a library type |
| `server/public/model/link_metadata.go:71` | domain | `TruncateOpenGraph(ogdata *opengraph.OpenGraph) *opengraph.OpenGraph` |
| `server/public/model/link_metadata.go:91` | domain | `FilterSVGImages(images []*image.Image) []*image.Image` |
| `server/public/model/link_metadata.go:161` | domain | `if _, ok := o.Data.(*opengraph.OpenGraph); !ok` — validation via type assertion |
| `server/public/model/link_metadata.go:198` | domain | `og := &opengraph.OpenGraph{}` — reconstructing the library object from DB JSON |
| `server/channels/app/opengraph.go:12-13` | service | imports `opengraph` + alias `ogImage` |
| `server/channels/app/opengraph.go:54-79` | service | `parseOpenGraphMetadata(...) *opengraph.OpenGraph` — `og.ProcessHTML(body)` (`:58`) |
| `server/channels/app/opengraph.go:90-131` | service | `makeOpenGraphURLsAbsolute(og *opengraph.OpenGraph, …)` |
| `server/channels/app/opengraph.go:133-147` | service | `openGraphDataWithProxyAddedToImageURLs(...)` |
| `server/channels/app/opengraph.go:150-157` | service | `filterSVGImagesFromOpenGraph(...)` |
| `server/channels/app/opengraph.go:159-162` | service | `openGraphDecodeHTMLEntities(og *opengraph.OpenGraph)` |
| `server/channels/app/opengraph.go:164-192` | service | `parseOpenGraphFromOEmbed(...)` — **hand-reconstructs** `&opengraph.OpenGraph{…}` (`:170`) and `&ogImage.Image{…}` (`:177`) |
| `server/channels/app/post_metadata.go:19` | service | import |
| `server/channels/app/post_metadata.go:34` | service | `OpenGraph *opengraph.OpenGraph` — field of the cache struct |
| `server/channels/app/post_metadata.go:624-632` | service | `embed.Data.(*opengraph.OpenGraph)` — type-asserts back to the library type |
| `server/channels/app/post_metadata.go:890` | service | `getLinkMetadata(...) (*opengraph.OpenGraph, *model.PostImage, …)` |
| `server/channels/app/post_metadata.go:907,916` | service | `og = model.TruncateOpenGraph(og)` (two call sites) |
| `server/channels/app/post_metadata.go:991` | service | `getLinkMetadataFromOEmbed(...) (*opengraph.OpenGraph, error)` |
| `server/channels/store/storetest/link_metadata_store.go:11-12` | persistence | the store contract is tested against library types |
| **`webapp/platform/types/src/posts.ts:176-191`** | **wire/UI (TS)** | **`OpenGraphMetadata` / `OpenGraphMetadataImage` — the library's shape hand-recreated in a second language** |
| `webapp/channels/src/components/post_view/post_attachment_opengraph/post_attachment_opengraph.tsx:56-75` | UI | `getBestImage()` — image-selection rule over the raw library shape |
| `webapp/channels/src/packages/mattermost-redux/src/reducers/entities/posts.ts`, `.../selectors/entities/posts.ts` | UI (store) | `openGraph: RelationOneToOne<Post, Record<string, OpenGraphMetadata>>` (`types/posts.ts:164`) |

### Dependencies ruled out (counter-examples — this is what a *working* ACL looks like)

| Dependency | Spread | Verdict |
|---|---|---|
| `minio-go` (S3) | only `server/platform/shared/filestore/` (`s3store.go`, `s3_overrides.go`, mocks) | **Properly isolated** behind the `FileBackend` interface. The pattern to imitate. |
| `squirrel` (SQL builder) | only `server/channels/store/` (+ `utils/textgeneration.go`) | Confined to persistence. |
| `luxon` / `moment-timezone` / `date-fns` (webapp) | 13 / 17 / 5 files | Real duplication (three date libraries at once), but **entirely inside the UI** — it does not cross a boundary, does not touch the wire contract or the database. |
| `go-opengraph` | **model + app + store + DB + REST + plugin API + TS types + UI** | **Leaks across every boundary.** |

---

## STEP 2 — CLASSIFICATION and picking #1

| Axis | `go-opengraph` | `minio-go` | dates in webapp |
|---|---|---|---|
| (a) layers / files touched | **7 layers**, 8 Go files + 5+ TS files | 1 layer | 1 layer, ~35 files |
| (b) risk/cost of swapping today | **Very high** — the library type is in the DB, in REST, in the plugin API and in the client's types. A swap = data migration + breaking API change. | Low (adapter) | Medium, local |
| (c) replaceability declared | **None — and that is the charge**: the public contract was set by the library, with no design decision behind it | Not needed — an ACL exists | — |
| (d) maintenance risk | **`v0.0.0-2022…` — pseudo-version, no releases since 2022** | Active | Active |

### Pick: **`github.com/dyatlov/go-opengraph`** — the worst leak.

**Rationale.** It is the only dependency that crosses **all of the following at once**:
1. the domain boundary (library types in signatures inside `model/`),
2. the persistence boundary (the library's JSON sits in the `LinkMetadata.Data` column),
3. the wire boundary (the shape reaches REST and `PostEmbed.Data`, `model/post_embed.go:24`),
4. the client/server boundary (the webapp **hand-recreates the library's schema in TypeScript**),

and on top of that it has had **no release since 2022** (pinned pseudo-version, `go.mod:22`). Net
effect: an unmaintained third-party library **de facto defines Mattermost's client data model**, and
swapping it is today an operation on the database and the public API — not on a single adapter.

---

## STEP 3 — DIAGNOSIS

### 3.1. The library type as a contract field and as a type in domain signatures

`model/link_metadata.go:41-45` — the library is written into the entity's contract *in a comment*,
because the type system cannot hold it here (`Data any`):

```go
// Data is the actual metadata for the link. It should contain data of one of the following types:
// - *model.PostImage if the linked content is an image
// - *opengraph.OpenGraph if the linked content is an HTML document
Data any
```

**Domain** rules (300-character limit, max N images, SVG filter) operate on library types —
`model/link_metadata.go:71-88`:

```go
func TruncateOpenGraph(ogdata *opengraph.OpenGraph) *opengraph.OpenGraph {
	ogdata.Title = truncateText(ogdata.Title)          // Mattermost's rule…
	ogdata.Images = FilterSVGImages(firstNImages(ogdata.Images, LinkMetadataMaxImages))
	ogdata.Article = empty.Article                      // …blanking fields the library should never have brought
```

The line `ogdata.Article = empty.Article` (`:77-85`, 8 fields) is a **textbook symptom of a missing
ACL**: domain code manually erases fields that exist *only because the library has them*. With a
value object of our own, those fields simply would not exist.

Entity validation reduces to a type assertion against the library type (`link_metadata.go:161`):
```go
if _, ok := o.Data.(*opengraph.OpenGraph); !ok {
```

### 3.2. Duplicated reconstruction of the library object (3 places)

| # | Place | Citation |
|---|---|---|
| 1 | `model/link_metadata.go:198` | `og := &opengraph.OpenGraph{}` — rebuilt from JSON **from the database** |
| 2 | `app/opengraph.go:55` | `og := opengraph.NewOpenGraph()` + `og.ProcessHTML(body)` (`:58`) — from **HTML** |
| 3 | `app/opengraph.go:170-183` | hand-assembled `&opengraph.OpenGraph{Type:…, Title:…}` + `&ogImage.Image{…}` — from **oEmbed** |

Three different sources (DB / HTML / oEmbed) → three independent reconstructions of **the same
library type**, each in a different layer.

### 3.3. The client/server leak — a domain rule duplicated across two languages

The webapp **does not receive domain data — it receives the library's raw shape** and runs logic on
it itself. `webapp/platform/types/src/posts.ts:176-191` is a manual transcription of the library's
JSON tags (hence `secure_url`, `site_name` — that is not Mattermost's naming):

```ts
export declare type OpenGraphMetadataImage = {
    secure_url?: string;
    url: string;
    type?: string;
    height?: number;
    width?: number;
};
```

The rule **"secure_url beats url"** lives today on **both sides of the boundary**:

| Side | Citation |
|---|---|
| Server (Go) | `app/opengraph.go:136-143` — `if image.SecureURL != "" { url = image.SecureURL } else { url = image.URL }` |
| Client (TS) | `post_attachment_opengraph.tsx:63` — `const imageUrl = image.secure_url \|\| image.url;` |
| Client (TS), again | `post_attachment_opengraph.tsx:199` — `const src = imageMetadata.secure_url \|\| imageMetadata.url \|\| '';` |

**To be precise:** these are not three parallel copies of one rule. The server copy sits **inside**
the proxy function (`openGraphDataWithProxyAddedToImageURLs`) and only runs **when the image proxy
is enabled** (gate at `app/opengraph.go:69-71`), whereas both client copies run **always**. That
makes the diagnosis worse, not milder: the same domain rule is not only scattered across two
languages, it is also **conditional on one side of the boundary** — behavior depends on server
configuration, and the client computes its own answer regardless.

Meanwhile the rule **"which image is best"** (`getNearestPoint` over dimensions) exists **only on
the client** — `post_attachment_opengraph.tsx:56-75`. The server ships the client a *bag of images*
and makes it choose; the client patches in the missing dimensions from `post.metadata.images`
(`:67-68`), because the library did not supply them.

### 3.4. Scoping the harm correctly (deliberately not overclaiming)

The server is written in Go, so **I do not claim that a server library ends up in the JS bundle** —
that is technically impossible. The harm is different and measurable: **the library's schema and its
rules are duplicated across the Go/TS boundary**, so an unmaintained library from 2022 dictates the
client's data model. Changing the library = changing `posts.ts` = changing React components.

---

## STEP 4 — ACL DESIGN

### 4.0. Binding design constraint: **the ACL takes ownership of the shape, it does not change the bytes**

The JSON shape (`secure_url`, `site_name`, `images[]`) is already a **published contract** in four
places at once: (1) the `LinkMetadata.Data` column in the database, (2) the REST response,
(3) `PostEmbed.Data` visible to plugins (`model/post_embed.go:24`), (4) `posts.ts:176-191`.

Therefore the value object **must serialize byte-for-byte identically** to today's
`opengraph.OpenGraph`. The refactor **does not introduce nicer field names** — that would break the
database, the API and the client, and would make the isolation proof false. The goal is different:
today the shape is defined *accidentally* by a JSON tag in a third-party library; after the refactor
the same shape is defined *deliberately* by a type Mattermost owns.

> This is the axis of the entire plan. An ACL that "improves" field names along the way is not an
> ACL — it is a breaking change.

**The sharpest risk to this assumption — naming it explicitly.** `TruncateOpenGraph` today blanks
eight fields (`Article`, `Book`, `Profile`, `Determiner`, `Locale`, `LocalesAlternate`, `Audios`,
`Videos` — `model/link_metadata.go:77-85`), and the proposed VO **does not have them at all**.
Whether that is byte-identical depends entirely on whether the library's struct tags carry
`omitempty`:

- if they **do** → a zeroed field = absent from the JSON = the VO produces the same bytes;
- if they **do not** → today's wire format carries `"article":{}` / `"videos":null`, and the VO
  would **silently change the bytes** — which would put an asterisk on the proof in 5.1 ("a swap
  does not touch DB/API/UI"), and 5.3 would no longer be purely additive.

**I did not verify this** — the library is not in the local module cache (`find` for
`dyatlov/go-opengraph` returned no sources), and I do not download dependencies as part of static
analysis. That is why **Phase 1 of the plan (STEP 6) carries this as an explicit test goal**: the
golden tests must target those eight fields specifically, not just "freeze the bytes" in general.
This is the spot that will bite during execution — and it must be settled *before* Phase 4, with a
single read of the struct tags.

### 4.1. Value object — the single place that knows the shape

New domain package: `server/public/model/linkpreview/` (**with no library import**).

```go
package linkpreview

// LinkPreview — Mattermost's VO. The JSON tags deliberately mirror today's wire format (see 4.0).
type LinkPreview struct {
	Type        string  `json:"type,omitempty"`
	Title       string  `json:"title,omitempty"`
	Description string  `json:"description,omitempty"`
	SiteName    string  `json:"site_name,omitempty"`
	URL         string  `json:"url,omitempty"`
	Images      []Image `json:"images,omitempty"`
}

type Image struct {
	URL       string `json:"url"`
	SecureURL string `json:"secure_url,omitempty"`
	Type      string `json:"type,omitempty"`
	Width     uint64 `json:"width,omitempty"`
	Height    uint64 `json:"height,omitempty"`
}

// --- domain operations (today scattered across model/ + app/ + UI) ---

// DisplayURL — the "secure wins" rule. Today: app/opengraph.go:136-143 AND
// post_attachment_opengraph.tsx:63 and :199.
func (i Image) DisplayURL() string

// IsSVG — today: model.IsSVGImageURL + FilterSVGImages (link_metadata.go:91-125).
func (i Image) IsSVG() bool

// Truncate — today: model.TruncateOpenGraph (link_metadata.go:71).
// After the refactor there is nothing to "blank" (Article/Book/Profile/…) — the VO has no such fields.
func (p LinkPreview) Truncate(maxImages int) LinkPreview

// WithoutSVGImages — today: model.FilterSVGImages + app.filterSVGImagesFromOpenGraph.
func (p LinkPreview) WithoutSVGImages() LinkPreview

// WithProxiedImages — today: app.openGraphDataWithProxyAddedToImageURLs (opengraph.go:133).
func (p LinkPreview) WithProxiedImages(toProxyURL func(string) string) LinkPreview

// WithAbsoluteURLs — today: app.makeOpenGraphURLsAbsolute (opengraph.go:90).
func (p LinkPreview) WithAbsoluteURLs(base string) LinkPreview

// BestImage — the image-selection rule. TODAY IT EXISTS ONLY ON THE CLIENT
// (post_attachment_opengraph.tsx:56-75, getNearestPoint). See STEP 5.3.
func (p LinkPreview) BestImage(dims []Image, known map[string]PostImageDims) (Image, bool)
```

### 4.2. Port — a narrow domain interface

The port has **two responsibilities**, because the library today does two things: it is a data type
**and** an HTML parser (`og.ProcessHTML(body)`, `app/opengraph.go:58`). The VO alone is not enough —
if `ProcessHTML` stayed in `app/`, `grep dyatlov` would still hit the service layer and the success
criterion from STEP 6 would not be met.

```go
package linkpreview

// LinkPreviewParser — the ONLY entrance to the library's world. Input: bytes. Output: a domain VO.
type LinkPreviewParser interface {
	// ParseHTML replaces app.parseOpenGraphMetadata (opengraph.go:54).
	ParseHTML(body io.Reader, contentType, requestURL string) (LinkPreview, error)

	// ParseOEmbed replaces app.parseOpenGraphFromOEmbed (opengraph.go:164).
	ParseOEmbed(body io.Reader, requestURL string) (LinkPreview, error)
}
```

Nowhere in the port's signature does a library type appear — only `io.Reader`, `string`, and the VO.

### 4.3. Adapter — the only file importing `dyatlov/go-opengraph`

`server/platform/shared/linkpreview/dyatlov/adapter.go`:

```go
package dyatlov

import "github.com/dyatlov/go-opengraph/opengraph"   // ← THE ONLY import in the whole repo

type Adapter struct{}

func (Adapter) ParseHTML(body io.Reader, contentType, requestURL string) (linkpreview.LinkPreview, error) {
	og := opengraph.NewOpenGraph()
	if err := og.ProcessHTML(forceUTF8(body, contentType)); err != nil { … }
	return toDomain(og), nil        // ← library type → VO conversion, in one place
}

func (Adapter) ParseOEmbed(body io.Reader, requestURL string) (linkpreview.LinkPreview, error) {
	// today: hand-assembling &opengraph.OpenGraph{} (opengraph.go:170-183)
	// after: we assemble the VO directly — the library is not needed here at all
}

// toDomain — the only function in the repo that knows what opengraph.OpenGraph looks like.
func toDomain(og *opengraph.OpenGraph) linkpreview.LinkPreview
```

The rest of the code (model, app, store, api4) knows **only the port and the VO**.

Design note: after the refactor `ParseOEmbed` **does not need the library at all** — today it uses it
only because `*opengraph.OpenGraph` was the only type available to speak in. That is itself proof
that the library type had usurped the role of a domain type.

---

## STEP 5 — Isolation proof + before/after

### 5.1. Proof: swapping the library touches only the adapter

Scenario: `dyatlov/go-opengraph` (dead since 2022) → any other OG parser.

| Artifact | Touched by a swap today? | After the refactor |
|---|---|---|
| `go.mod` | yes | yes (unavoidable) |
| **Adapter** (`.../linkpreview/dyatlov/`) | — (does not exist) | **yes — the only file to rewrite** |
| `model/link_metadata.go` | **yes** (import, `Data any`, 3 signatures, type assert, reconstruction) | **no** — operates on the VO |
| `app/opengraph.go` | **yes** (the entire file) | **no** — delegates to the port |
| `app/post_metadata.go` | **yes** (struct field, type assert, 2 signatures) | **no** |
| `store/storetest/link_metadata_store.go` | **yes** | **no** |
| **DB schema / the `Data` column** | **yes** (the library's JSON is written to disk) | **no** — the VO serializes byte-identically (4.0) |
| **REST response / `PostEmbed.Data`** | **yes** (breaking API change) | **no** — the same JSON |
| **`webapp/platform/types/src/posts.ts`** | **yes** (hand-written library schema) | **no** — the contract is Mattermost's from now on |
| **React components** | **yes** | **no** |

Swapping the library stops being a DB migration + breaking API change, and becomes rewriting **one
file** (`toDomain` + `ParseHTML`).

### 5.2. Before / after — the duplicated sites

| Rule | BEFORE (verified) | AFTER |
|---|---|---|
| Reconstructing the OG object | 3 places: `model/link_metadata.go:198`, `app/opengraph.go:55`, `app/opengraph.go:170` | 1: `dyatlov.toDomain()` |
| SVG filter | 3: `model/link_metadata.go:91`, `model/link_metadata.go:83` (inside `Truncate`), `app/opengraph.go:150` | 1: `LinkPreview.WithoutSVGImages()` |
| "secure_url wins" | 3, **across two languages**, and the server copy is **conditional** (only with the image proxy enabled, `app/opengraph.go:69-71,136-143`) while both client copies are unconditional (`post_attachment_opengraph.tsx:63`, `:199`) | 1: `Image.DisplayURL()` (Go), unconditional |
| Blanking library fields (`Article`, `Book`, `Profile`, `Determiner`, `Locale`, …) | `model/link_metadata.go:77-85` — 8 assignments | **0** — the VO has no such fields |
| Choosing the best image | 1, **on the wrong side of the boundary**: `post_attachment_opengraph.tsx:56-75` | 1: `LinkPreview.BestImage()` (server) |

### 5.3. The UI receives domain data, not the raw library object

This is the real test of an ACL. Today the server sends the client a **bag of images in the
library's shape**, and the client runs domain logic on it (`getBestImage`, `secure_url || url`,
patching in dimensions from `post.metadata.images`, `post_attachment_opengraph.tsx:56-75`).

After the refactor the server **decides and sends the result**. The VO computes `BestImage` and
`DisplayURL` on the server, and the wire format gains an **additive** (non-breaking) field, e.g.:

```jsonc
"data": {
  "type": "opengraph", "title": "…", "site_name": "…",
  "images": [ … ],                 // STAYS — backward compatibility (4.0)
  "preview_image": {               // NEW, additive: the server has already chosen
      "url": "…", "width": 800, "height": 418, "format": "png"
  }
}
```

The client stops reimplementing the rule: `getBestImage` disappears, the component reads
`preview_image`. The old `images` field stays for compatibility (older clients, plugins) — that is
the price of the published contract from 4.0, and it has to be paid knowingly.

### 5.4. Resolving the questions that depend on the library's contract

Two decisions hang in the air today, because they depend on whatever the library *happens* to
return. Both belong **in the ACL (VO/adapter), not in the API layer and not in the UI**.

*Source of the resolution (stated honestly):* I resolve them against the **OpenGraph protocol
specification that this library implements** (`og:image:secure_url` = the HTTPS variant of the same
resource; `og:image:width`/`height` are optional) — and **not** against the library's own README,
which I did not read (its sources are not in the local cache, see 4.0). If `go-opengraph` deviates
from the spec, the resolution does not change — only what the adapter has to normalize changes, and
that is precisely the adapter's job.

| Open question | Where the answer comes from today | Where to encode it |
|---|---|---|
| What if the site does not provide the image's `width`/`height`? (the OG spec makes them optional) | the client patches from `post.metadata.images`, and failing that inserts `-1` (`post_attachment_opengraph.tsx:67-68`) | `LinkPreview.BestImage()` — the server already has `PostMetadata.Images` and knows the dimensions; `-1` as "unknown" never leaks to the UI |
| Which URL is authoritative: `url` or `secure_url`? (OG spec: `og:image:secure_url` is the HTTPS variant) | the rule is duplicated 3× across 2 languages (5.2) | `Image.DisplayURL()` — one place; the image proxy (`app/opengraph.go:133-147`) becomes its only consumer |

---

## STEP 6 — Verification and phased plan

### Success criterion (executable)

```bash
grep -rn "dyatlov/go-opengraph" server --include=*.go | grep -v _test
```

**Today:** 8 files across 3 layers (`model/`, `app/`, `store/storetest/`) — see STEP 1.
**After the refactor:** only files under `server/platform/shared/linkpreview/dyatlov/`.

Complementary criterion (the client/server boundary):
```bash
grep -rn "secure_url\|getBestImage" webapp/channels/src --include=*.tsx --include=*.ts | grep -v test
```
**Today:** 3 hits (`post_attachment_opengraph.tsx:56,63,199`). **After:** 0.

### Who knows the dependency today, and who will not

| Stops knowing | Still knows (by design) |
|---|---|
| `server/public/model/link_metadata.go` | `server/platform/shared/linkpreview/dyatlov/adapter.go` |
| `server/channels/app/opengraph.go` | `server/go.mod` |
| `server/channels/app/post_metadata.go` | (the adapter's tests) |
| `server/channels/store/storetest/link_metadata_store.go` | |
| `webapp/platform/types/src/posts.ts` | |
| `webapp/channels/src/components/post_view/post_attachment_opengraph/*` | |

### Phased plan (following the `context/changes/<change-id>/` convention, consistent with `01`/`02`)

| Phase | Scope | Exit criterion |
|---|---|---|
| **1. Characterization** | Golden tests over today's JSON: HTML → JSON, oEmbed → JSON, DB JSON → JSON. **No production code changes.** **Explicit goal: read the `opengraph.OpenGraph` struct tags and settle the question of the eight blanked fields from 4.0** (`omitempty` or not) — the golden test must cover a post after `TruncateOpenGraph`, since that is where those fields get zeroed. | Fixtures freezing the wire-format bytes; **an answer to whether dropping the 8 fields in the VO is additive or breaking** (the safety net for 4.0) |
| **2. VO + port + adapter** | New `linkpreview` package + the `dyatlov` adapter. Old code still untouched. | `toDomain()` passes the Phase 1 golden tests **byte-for-byte** |
| **3. Invert the dependency in `app/`** | `app/opengraph.go` and `app/post_metadata.go` call the port; the `parseOpenGraph*` functions disappear. | `grep dyatlov server/channels/app` → 0 |
| **4. Clean up `model/`** | `TruncateOpenGraph`/`FilterSVGImages`/`firstNImages` → VO methods; `Data any` typed as the VO; deserialization through the adapter. | `grep dyatlov server/public/model` → 0; the success criterion is met on the Go side |
| **5. The client/server boundary** | Additive `preview_image` (5.3); `getBestImage` removed from the UI; `posts.ts` describes Mattermost's contract, not the library's schema. | `grep secure_url\|getBestImage webapp` → 0 |
| **6. Proof of replaceability** | A second adapter (even a stub / "parser B") plugged in behind the port — without touching the DB, the API or the UI. | Swapping the adapter = 1 changed file + 1 line of wiring |

Phases 1-2 are non-invasive and can be merged independently. Phases 3-4 deliver all of the
server-side value. Phase 5 is the only one that touches the wire contract — and it does so
**additively**.

---

## Artifact limitations

- **No requirements document** (PRD/tech-stack) — there is no replaceability declaration to cite
  (STEP 0). The diagnosis rests on the code, not on written-down intent.
- **Static analysis** (grep + reading files) — no code was run, no tests were executed. It was not
  empirically verified that the VO serializes byte-identically; that is a **design assumption**, and
  Phase 1 of the plan is meant to prove it.
- **The library's own sources were not read** — `go-opengraph` is neither vendored nor present in
  the local module cache. The consequence is concrete and flagged in 4.0: I do not know whether the
  eight fields blanked by `TruncateOpenGraph` carry `omitempty`, and byte-identity of the VO hinges
  on that. **This is the single unresolved assumption the isolation proof in 5.1 rests on.**
- **External plugins were not examined** — `PostEmbed.Data` (`model/post_embed.go:24`) is visible to
  plugins; the real reach of the published contract may extend beyond this repo.
- `file:line` citations refer to the state of the `mattermost/` repo on branch `module-4-lesson-5`;
  line numbers may shift after edits.
- The comparison axes in STEP 2 (`minio-go`, `squirrel`, date libraries) were measured coarsely
  (count of importing files), without a full audit of each.
