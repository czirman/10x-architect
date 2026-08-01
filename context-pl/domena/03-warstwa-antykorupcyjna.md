---
title: Anti-Corruption Layer — go-opengraph jako przeciekająca zależność
created: 2026-07-12
type: refactor-plan
---

# Anti-Corruption Layer — `go-opengraph` jako przeciekająca zależność

> Produkt tego dokumentu to **PLAN refaktoru**, nie implementacja. Kod produkcyjny nie został
> zmodyfikowany. Zależność #1 została **odkryta** (pomiar importów po warstwach), a nie założona
> z góry. Każdy cytat `plik:linia` był weryfikowany w checkoucie `mattermost/` (branch
> `module-4-lesson-5`).

---

## KROK 0 — Kontekst projektu

**Stack.** Monorepo Mattermost: serwer w Go (`server/go.mod`, `go 1.26.3`) + webapp w
TypeScript/React (`webapp/channels`, `webapp/platform`). Warstwy po stronie serwera (zgodne z
mapą z `context/domain/01-domain-distillation.md`):

| Warstwa | Ścieżka | Rola |
|---|---|---|
| Kontrakt domenowy + wire | `server/public/model/` | Encje, `IsValid()`, **kształt JSON-a wysyłanego do klienta** |
| Aplikacja (serwis) | `server/channels/app/` | Logika międzyencyjna |
| Persystencja | `server/channels/store/` | Dostęp do danych |
| API | `server/channels/api4/` | HTTP |
| UI | `webapp/` | React |

**Deklaracje o wymienialności — LIMITACJA (uczciwy raport).** `grep -niE "swap|pluggable|
interchangeab|replace the|drop-in"` po `README.md`, `AGENTS.md`, `server/README.md`,
`webapp/README.md` nie zwrócił **żadnej** deklaracji, że którykolwiek komponent ma być
wymienialny. Nie ma też PRD ani tech-stack.md (potwierdzenie ustalenia z
`01-domain-distillation.md:22-30`).

To **nie osłabia** znaleziska — przeciwnie. Nie mamy tu rozjazdu „intencja-vs-kod" (dokument mówi
„wymienialne", kod na to nie pozwala). Mamy coś gorszego: **przypadkowo opublikowany kontrakt** —
kształt danych, który biblioteka zewnętrzna narzuciła publicznemu API i bazie danych, a **nikt
nigdy nie podjął decyzji, że tak ma być**. Nie ma dokumentu do zacytowania, bo nie ma decyzji.
Kontrakt powstał przez wyciek, nie przez projekt.

---

## KROK 1 — IDENTYFIKACJA przeciekających zależności

Pomiar: importy third-party wewnątrz `server/public/model/*.go` (warstwa kontraktu — biblioteka,
która tu jest, jest automatycznie widoczna dla wszystkich warstw powyżej **i dla klienta**):

```
10  github.com/pkg/errors
 4  github.com/tinylib/msgp/msgp
 4  github.com/goccy/go-yaml
 3  github.com/vmihailenco/msgpack/v5
 2  github.com/gorilla/websocket
 8  github.com/dyatlov/go-opengraph/...   ← typy biblioteki w polach i sygnaturach domenowych
 1  github.com/mattermost/ldap
```

`pkg/errors`, `msgp`, `msgpack`, `go-yaml` to biblioteki **infrastrukturalne/serializacyjne** —
nie narzucają kształtu bytu domenowego. `go-opengraph` jest jedyną, której **struktura danych**
staje się polem encji i typem w sygnaturach domenowych.

### Wszystkie pliki, które dziś „znają" `github.com/dyatlov/go-opengraph`

(pełna lista; `grep -rn "dyatlov/go-opengraph" server --include=*.go`, bez `_test.go`)

| Plik:linia | Warstwa | Co dokładnie wie |
|---|---|---|
| `server/go.mod:22` | manifest | `go-opengraph v0.0.0-20220524092352-606d7b1e5f8a` — **pseudo-wersja, ostatni commit z maja 2022** |
| `server/public/model/link_metadata.go:17-18` | **kontrakt/domena** | import `opengraph` + `opengraph/types/image` |
| `server/public/model/link_metadata.go:43,45` | **kontrakt/domena** | `Data any` — komentarz: „should contain (…) `*opengraph.OpenGraph`" |
| `server/public/model/link_metadata.go:57` | domena | `firstNImages(images []*image.Image, …)` — reguła domenowa na typie biblioteki |
| `server/public/model/link_metadata.go:71` | domena | `TruncateOpenGraph(ogdata *opengraph.OpenGraph) *opengraph.OpenGraph` |
| `server/public/model/link_metadata.go:91` | domena | `FilterSVGImages(images []*image.Image) []*image.Image` |
| `server/public/model/link_metadata.go:161` | domena | `if _, ok := o.Data.(*opengraph.OpenGraph); !ok` — walidacja przez type-assert |
| `server/public/model/link_metadata.go:198` | domena | `og := &opengraph.OpenGraph{}` — rekonstrukcja obiektu biblioteki z JSON-a z bazy |
| `server/channels/app/opengraph.go:12-13` | serwis | import `opengraph` + alias `ogImage` |
| `server/channels/app/opengraph.go:54-79` | serwis | `parseOpenGraphMetadata(...) *opengraph.OpenGraph` — `og.ProcessHTML(body)` (`:58`) |
| `server/channels/app/opengraph.go:90-131` | serwis | `makeOpenGraphURLsAbsolute(og *opengraph.OpenGraph, …)` |
| `server/channels/app/opengraph.go:133-147` | serwis | `openGraphDataWithProxyAddedToImageURLs(...)` |
| `server/channels/app/opengraph.go:150-157` | serwis | `filterSVGImagesFromOpenGraph(...)` |
| `server/channels/app/opengraph.go:159-162` | serwis | `openGraphDecodeHTMLEntities(og *opengraph.OpenGraph)` |
| `server/channels/app/opengraph.go:164-192` | serwis | `parseOpenGraphFromOEmbed(...)` — **ręczna rekonstrukcja** `&opengraph.OpenGraph{…}` (`:170`) i `&ogImage.Image{…}` (`:177`) |
| `server/channels/app/post_metadata.go:19` | serwis | import |
| `server/channels/app/post_metadata.go:34` | serwis | `OpenGraph *opengraph.OpenGraph` — pole struktury cache'a |
| `server/channels/app/post_metadata.go:624-632` | serwis | `embed.Data.(*opengraph.OpenGraph)` — type-assert z powrotem na typ biblioteki |
| `server/channels/app/post_metadata.go:890` | serwis | `getLinkMetadata(...) (*opengraph.OpenGraph, *model.PostImage, …)` |
| `server/channels/app/post_metadata.go:907,916` | serwis | `og = model.TruncateOpenGraph(og)` (dwa wywołania) |
| `server/channels/app/post_metadata.go:991` | serwis | `getLinkMetadataFromOEmbed(...) (*opengraph.OpenGraph, error)` |
| `server/channels/store/storetest/link_metadata_store.go:11-12` | persystencja | kontrakt store'a testowany na typach biblioteki |
| **`webapp/platform/types/src/posts.ts:176-191`** | **wire/UI (TS)** | **`OpenGraphMetadata` / `OpenGraphMetadataImage` — ręcznie odtworzony kształt biblioteki w drugim języku** |
| `webapp/channels/src/components/post_view/post_attachment_opengraph/post_attachment_opengraph.tsx:56-75` | UI | `getBestImage()` — reguła wyboru obrazka na surowym kształcie biblioteki |
| `webapp/channels/src/packages/mattermost-redux/src/reducers/entities/posts.ts`, `.../selectors/entities/posts.ts` | UI (store) | `openGraph: RelationOneToOne<Post, Record<string, OpenGraphMetadata>>` (`types/posts.ts:164`) |

### Zależności odrzucone (kontrprzykłady — tak wygląda ACL, który *działa*)

| Zależność | Rozrzut | Werdykt |
|---|---|---|
| `minio-go` (S3) | wyłącznie `server/platform/shared/filestore/` (`s3store.go`, `s3_overrides.go`, mocks) | **Poprawnie odizolowana** za interfejsem `FileBackend`. Wzorzec do naśladowania. |
| `squirrel` (builder SQL) | wyłącznie `server/channels/store/` (+ `utils/textgeneration.go`) | Zamknięta w persystencji. |
| `luxon` / `moment-timezone` / `date-fns` (webapp) | 13 / 17 / 5 plików | Duplikacja realna (3 biblioteki dat naraz), ale **cała wewnątrz UI** — nie przecieka przez granicę, nie dotyka kontraktu wire ani bazy. |
| `go-opengraph` | **model + app + store + baza + REST + plugin API + TS types + UI** | **Przeciek przez wszystkie granice.** |

---

## KROK 2 — KLASYFIKACJA i wybór #1

| Oś oceny | `go-opengraph` | `minio-go` | daty w webapp |
|---|---|---|---|
| (a) liczba warstw / plików | **7 warstw**, 8 plików Go + 5+ plików TS | 1 warstwa | 1 warstwa, ~35 plików |
| (b) ryzyko/koszt wymiany dziś | **Bardzo wysoki** — typ biblioteki jest w bazie, w REST, w plugin API i w typach klienta. Wymiana = migracja danych + breaking change API. | Niski (adapter) | Średni, lokalny |
| (c) deklaracja wymienialności | **Brak — i to jest zarzut**: kontrakt publiczny został ustalony przez bibliotekę, bez decyzji projektowej | Nie trzeba — ACL istnieje | — |
| (d) ryzyko utrzymaniowe | **`v0.0.0-2022…` — pseudo-wersja, brak wydań od 2022** | Aktywna | Aktywne |

### Wybór: **`github.com/dyatlov/go-opengraph`** — najgorszy przeciek.

**Uzasadnienie.** To jedyna zależność, która przekracza **jednocześnie**:
1. granicę domeny (typy biblioteki w sygnaturach w `model/`),
2. granicę persystencji (JSON biblioteki leży w kolumnie `LinkMetadata.Data`),
3. granicę wire (kształt trafia do REST i do `PostEmbed.Data`, `model/post_embed.go:24`),
4. granicę klient/serwer (webapp **ręcznie odtwarza schemat biblioteki w TypeScripcie**),

a przy tym jest **niewydawana od 2022** (pinned pseudo-version, `go.mod:22`). Efekt: nieutrzymywana
biblioteka trzeciej strony **de facto definiuje model danych klienta Mattermosta**, a jej wymiana
jest dziś operacją na bazie danych i publicznym API — nie na jednym adapterze.

---

## KROK 3 — DIAGNOZA

### 3.1. Typ biblioteki jako pole kontraktu i jako typ w sygnaturach domenowych

`model/link_metadata.go:41-45` — biblioteka jest wpisana w kontrakt encji *komentarzem*, bo
system typów jej tu nie utrzyma (`Data any`):

```go
// Data is the actual metadata for the link. It should contain data of one of the following types:
// - *model.PostImage if the linked content is an image
// - *opengraph.OpenGraph if the linked content is an HTML document
Data any
```

Reguły **domenowe** (limit 300 znaków, max N obrazków, filtr SVG) operują na typach biblioteki —
`model/link_metadata.go:71-88`:

```go
func TruncateOpenGraph(ogdata *opengraph.OpenGraph) *opengraph.OpenGraph {
	ogdata.Title = truncateText(ogdata.Title)          // reguła Mattermosta…
	ogdata.Images = FilterSVGImages(firstNImages(ogdata.Images, LinkMetadataMaxImages))
	ogdata.Article = empty.Article                      // …wygaszanie pól, których biblioteka nie powinna była przynieść
```

Zdanie `ogdata.Article = empty.Article` (`:77-85`, 8 pól) to **czysty objaw braku ACL**: kod
domenowy ręcznie kasuje pola, które istnieją *tylko dlatego, że biblioteka je ma*. Gdyby istniał
własny value object, tych pól po prostu by nie było.

Walidacja encji sprowadza się do type-assertu na typ biblioteki (`link_metadata.go:161`):
```go
if _, ok := o.Data.(*opengraph.OpenGraph); !ok {
```

### 3.2. Zduplikowana rekonstrukcja obiektu biblioteki (3 miejsca)

| # | Miejsce | Cytat |
|---|---|---|
| 1 | `model/link_metadata.go:198` | `og := &opengraph.OpenGraph{}` — odtworzenie z JSON-a **z bazy** |
| 2 | `app/opengraph.go:55` | `og := opengraph.NewOpenGraph()` + `og.ProcessHTML(body)` (`:58`) — z **HTML-a** |
| 3 | `app/opengraph.go:170-183` | ręczne złożenie `&opengraph.OpenGraph{Type:…, Title:…}` + `&ogImage.Image{…}` — z **oEmbed** |

Trzy różne źródła (baza / HTML / oEmbed) → trzy niezależne rekonstrukcje **tego samego typu
biblioteki**, każda w innej warstwie.

### 3.3. Przeciek przez granicę klient/serwer — reguła domenowa zduplikowana w dwóch językach

Webapp **nie dostaje danych domenowych — dostaje surowy kształt biblioteki** i sam wykonuje na nim
logikę. `webapp/platform/types/src/posts.ts:176-191` to ręczna transkrypcja tagów JSON biblioteki
(stąd `secure_url`, `site_name` — to nie jest nazewnictwo Mattermosta):

```ts
export declare type OpenGraphMetadataImage = {
    secure_url?: string;
    url: string;
    type?: string;
    height?: number;
    width?: number;
};
```

Reguła **„secure_url wygrywa nad url"** żyje dziś po **obu stronach granicy**:

| Strona | Cytat |
|---|---|
| Serwer (Go) | `app/opengraph.go:136-143` — `if image.SecureURL != "" { url = image.SecureURL } else { url = image.URL }` |
| Klient (TS) | `post_attachment_opengraph.tsx:63` — `const imageUrl = image.secure_url \|\| image.url;` |
| Klient (TS), ponownie | `post_attachment_opengraph.tsx:199` — `const src = imageMetadata.secure_url \|\| imageMetadata.url \|\| '';` |

**Precyzyjnie:** to nie są trzy równoległe kopie jednej reguły. Kopia serwerowa siedzi **wewnątrz**
funkcji proxy (`openGraphDataWithProxyAddedToImageURLs`) i wykonuje się **tylko gdy image proxy jest
włączone** (bramka `app/opengraph.go:69-71`), a obie kopie klienckie działają **zawsze**. To
pogarsza diagnozę, nie łagodzi jej: ta sama reguła domenowa jest nie tylko rozsiana po dwóch
językach, ale i **warunkowa po jednej stronie granicy** — czyli zachowanie zależy od konfiguracji
serwera, a klient i tak liczy swoje.

A reguła **„który obrazek jest najlepszy"** (`getNearestPoint` po wymiarach) istnieje **wyłącznie
po stronie klienta** — `post_attachment_opengraph.tsx:56-75`. Serwer wysyła klientowi *worek
obrazków* i każe mu wybrać; klient dokleja brakujące wymiary z `post.metadata.images` (`:67-68`),
bo biblioteka ich nie dostarczyła.

### 3.4. Sprostowanie zakresu szkody (świadomie nie przesadzam)

Serwer jest w Go, więc **nie twierdzę, że biblioteka serwerowa trafia do bundla JS** — to
technicznie niemożliwe. Szkoda jest inna i mierzalna: **schemat i reguły biblioteki są
zduplikowane przez granicę Go/TS**, więc nieutrzymywana biblioteka z 2022 r. dyktuje model danych
klienta. Zmiana biblioteki = zmiana `posts.ts` = zmiana komponentów React.

---

## KROK 4 — PROJEKT ACL

### 4.0. Wiążące ograniczenie projektowe: **ACL przejmuje własność kształtu, nie zmienia bajtów**

Kształt JSON (`secure_url`, `site_name`, `images[]`) jest już **opublikowanym kontraktem** w
czterech miejscach naraz: (1) kolumna `LinkMetadata.Data` w bazie, (2) odpowiedź REST,
(3) `PostEmbed.Data` widoczne dla pluginów (`model/post_embed.go:24`), (4) `posts.ts:176-191`.

Dlatego value object **musi serializować się bajt-w-bajt identycznie** jak dzisiejszy
`opengraph.OpenGraph`. Refaktor **nie wprowadza ładniejszych nazw pól** — bo to złamałoby bazę,
API i klienta, i uczyniłoby dowód izolacji fałszywym. Cel jest inny: dziś kształt definiuje
*przypadkiem* tag JSON w bibliotece trzeciej strony; po refaktorze ten sam kształt definiuje
*świadomie* typ należący do Mattermosta.

> To jest oś całego planu. ACL, który przy okazji „poprawia" nazwy pól, nie jest ACL-em — jest
> breaking changem.

**Najostrzejsze ryzyko tego założenia — nazywam je wprost.** `TruncateOpenGraph` wygasza dziś
osiem pól (`Article`, `Book`, `Profile`, `Determiner`, `Locale`, `LocalesAlternate`, `Audios`,
`Videos` — `model/link_metadata.go:77-85`), a projektowany VO **nie ma ich w ogóle**. Czy to jest
bajt-identyczne, zależy wyłącznie od tego, czy struct tagi biblioteki mają `omitempty`:

- jeśli **mają** → pole wyzerowane = nieobecne w JSON = VO produkuje ten sam bajt;
- jeśli **nie mają** → dzisiejszy wire format niesie `"article":{}` / `"videos":null`, a VO
  **po cichu zmieniłby bajty** — i wtedy dowód z 5.1 („wymiana nie dotyka bazy/API/UI") ma gwiazdkę,
  a 5.3 nie jest już czysto addytywne.

**Nie zweryfikowałem tego** — biblioteki nie ma w lokalnym module cache (`find` po
`dyatlov/go-opengraph` nie zwrócił źródeł), a nie pobieram zależności w ramach analizy statycznej.
Dlatego **Faza 1 planu (KROK 6) ma to jako jawny cel testowy**: golden testy muszą celować
konkretnie w te osiem pól, a nie ogólnie „zamrozić bajty". To jest miejsce, które ugryzie przy
wykonaniu — i musi zostać rozstrzygnięte *przed* Fazą 4, jednym odczytem struct tagów.

### 4.1. Value object — jedyne miejsce wiedzy o kształcie

Nowy pakiet domenowy: `server/public/model/linkpreview/` (**bez importu biblioteki**).

```go
package linkpreview

// LinkPreview — VO Mattermosta. Tagi JSON celowo powielają dzisiejszy wire format (patrz 4.0).
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

// --- operacje domenowe (dziś rozsiane po model/ + app/ + UI) ---

// DisplayURL — reguła „secure wygrywa". Dziś: app/opengraph.go:136-143 ORAZ
// post_attachment_opengraph.tsx:63 i :199.
func (i Image) DisplayURL() string

// IsSVG — dziś: model.IsSVGImageURL + FilterSVGImages (link_metadata.go:91-125).
func (i Image) IsSVG() bool

// Truncate — dziś: model.TruncateOpenGraph (link_metadata.go:71).
// Po refaktorze nie ma czego „wygaszać" (Article/Book/Profile/…) — VO tych pól nie ma.
func (p LinkPreview) Truncate(maxImages int) LinkPreview

// WithoutSVGImages — dziś: model.FilterSVGImages + app.filterSVGImagesFromOpenGraph.
func (p LinkPreview) WithoutSVGImages() LinkPreview

// WithProxiedImages — dziś: app.openGraphDataWithProxyAddedToImageURLs (opengraph.go:133).
func (p LinkPreview) WithProxiedImages(toProxyURL func(string) string) LinkPreview

// WithAbsoluteURLs — dziś: app.makeOpenGraphURLsAbsolute (opengraph.go:90).
func (p LinkPreview) WithAbsoluteURLs(base string) LinkPreview

// BestImage — reguła wyboru obrazka. DZIŚ ISTNIEJE TYLKO W KLIENCIE
// (post_attachment_opengraph.tsx:56-75, getNearestPoint). Patrz KROK 5.3.
func (p LinkPreview) BestImage(dims []Image, known map[string]PostImageDims) (Image, bool)
```

### 4.2. Port — wąski interfejs domenowy

Port ma **dwie odpowiedzialności**, bo biblioteka dziś robi dwie rzeczy: jest typem danych **i**
parserem HTML (`og.ProcessHTML(body)`, `app/opengraph.go:58`). Sam VO nie wystarczy — gdyby
`ProcessHTML` zostało w `app/`, `grep dyatlov` nadal trafiałby w warstwę serwisu i kryterium
sukcesu z KROKU 6 byłoby niespełnione.

```go
package linkpreview

// LinkPreviewParser — JEDYNE wejście do świata biblioteki. Wejście: bajty. Wyjście: VO domenowy.
type LinkPreviewParser interface {
	// ParseHTML zastępuje app.parseOpenGraphMetadata (opengraph.go:54).
	ParseHTML(body io.Reader, contentType, requestURL string) (LinkPreview, error)

	// ParseOEmbed zastępuje app.parseOpenGraphFromOEmbed (opengraph.go:164).
	ParseOEmbed(body io.Reader, requestURL string) (LinkPreview, error)
}
```

Nigdzie w sygnaturze portu nie występuje typ biblioteki — tylko `io.Reader`, `string`, VO.

### 4.3. Adapter — jedyny plik importujący `dyatlov/go-opengraph`

`server/platform/shared/linkpreview/dyatlov/adapter.go`:

```go
package dyatlov

import "github.com/dyatlov/go-opengraph/opengraph"   // ← JEDYNY import w całym repo

type Adapter struct{}

func (Adapter) ParseHTML(body io.Reader, contentType, requestURL string) (linkpreview.LinkPreview, error) {
	og := opengraph.NewOpenGraph()
	if err := og.ProcessHTML(forceUTF8(body, contentType)); err != nil { … }
	return toDomain(og), nil        // ← konwersja typ biblioteki → VO, jedno miejsce
}

func (Adapter) ParseOEmbed(body io.Reader, requestURL string) (linkpreview.LinkPreview, error) {
	// dziś: ręczne składanie &opengraph.OpenGraph{} (opengraph.go:170-183)
	// po:   składamy od razu VO — biblioteka w ogóle nie jest tu potrzebna
}

// toDomain — jedyna funkcja w repo, która wie, jak wygląda opengraph.OpenGraph.
func toDomain(og *opengraph.OpenGraph) linkpreview.LinkPreview
```

Reszta kodu (model, app, store, api4) zna **wyłącznie port i VO**.

Uwaga projektowa: `ParseOEmbed` po refaktorze **nie potrzebuje biblioteki w ogóle** — dziś używa
jej tylko dlatego, że `*opengraph.OpenGraph` był jedynym typem, jakim można było mówić. To sam w
sobie dowód, że typ biblioteki pełnił rolę uzurpowanego typu domenowego.

---

## KROK 5 — Dowód izolacji + before/after

### 5.1. Dowód: wymiana biblioteki dotyka wyłącznie adaptera

Scenariusz: `dyatlov/go-opengraph` (martwy od 2022) → dowolny inny parser OG.

| Artefakt | Dziś dotknięty wymianą? | Po refaktorze |
|---|---|---|
| `go.mod` | tak | tak (nieuniknione) |
| **Adapter** (`.../linkpreview/dyatlov/`) | — (nie istnieje) | **tak — jedyny plik do przepisania** |
| `model/link_metadata.go` | **tak** (import, `Data any`, 3 sygnatury, type-assert, rekonstrukcja) | **nie** — operuje na VO |
| `app/opengraph.go` | **tak** (cały plik) | **nie** — deleguje do portu |
| `app/post_metadata.go` | **tak** (pole struktury, type-assert, 2 sygnatury) | **nie** |
| `store/storetest/link_metadata_store.go` | **tak** | **nie** |
| **Schemat bazy / kolumna `Data`** | **tak** (JSON biblioteki jest zapisany na dysku) | **nie** — VO serializuje się bajt-identycznie (4.0) |
| **Odpowiedź REST / `PostEmbed.Data`** | **tak** (breaking change API) | **nie** — ten sam JSON |
| **`webapp/platform/types/src/posts.ts`** | **tak** (ręczny schemat biblioteki) | **nie** — kontrakt jest odtąd Mattermosta |
| **Komponenty React** | **tak** | **nie** |

Wymiana biblioteki przestaje być migracją bazy + breaking changem API, a staje się przepisaniem
**jednego pliku** (`toDomain` + `ParseHTML`).

### 5.2. Before / after — miejsca zduplikowane

| Reguła | BEFORE (zweryfikowane) | AFTER |
|---|---|---|
| Rekonstrukcja obiektu OG | 3 miejsca: `model/link_metadata.go:198`, `app/opengraph.go:55`, `app/opengraph.go:170` | 1: `dyatlov.toDomain()` |
| Filtr SVG | 3: `model/link_metadata.go:91`, `model/link_metadata.go:83` (w `Truncate`), `app/opengraph.go:150` | 1: `LinkPreview.WithoutSVGImages()` |
| „secure_url wygrywa" | 3, **w dwóch językach**, przy czym kopia serwerowa jest **warunkowa** (tylko przy włączonym image proxy, `app/opengraph.go:69-71,136-143`), a obie klienckie bezwarunkowe (`post_attachment_opengraph.tsx:63`, `:199`) | 1: `Image.DisplayURL()` (Go), bezwarunkowo |
| Wygaszanie pól biblioteki (`Article`, `Book`, `Profile`, `Determiner`, `Locale`, …) | `model/link_metadata.go:77-85` — 8 przypisań | **0** — VO nie ma tych pól |
| Wybór najlepszego obrazka | 1, **po złej stronie granicy**: `post_attachment_opengraph.tsx:56-75` | 1: `LinkPreview.BestImage()` (serwer) |

### 5.3. Warstwa UI dostaje dane domenowe, nie surowy obiekt biblioteki

To jest właściwy test ACL-a. Dziś serwer wysyła klientowi **worek obrazków w kształcie biblioteki**,
a klient wykonuje na nim logikę domenową (`getBestImage`, `secure_url || url`, doklejanie wymiarów
z `post.metadata.images`, `post_attachment_opengraph.tsx:56-75`).

Po refaktorze serwer **rozstrzyga i wysyła wynik**. VO wyznacza `BestImage` i `DisplayURL` po
stronie serwera, a wire format zyskuje **addytywne** (niełamiące) pole, np.:

```jsonc
"data": {
  "type": "opengraph", "title": "…", "site_name": "…",
  "images": [ … ],                 // ZOSTAJE — kompatybilność wsteczna (4.0)
  "preview_image": {               // NOWE, addytywne: serwer już wybrał
      "url": "…", "width": 800, "height": 418, "format": "png"
  }
}
```

Klient przestaje reimplementować regułę: `getBestImage` znika, komponent czyta `preview_image`.
Stare pole `images` zostaje dla kompatybilności (starsze klienty, pluginy) — to jest cena
opublikowanego kontraktu z 4.0 i trzeba ją zapłacić świadomie.

### 5.4. Rozstrzygnięcie pytań zależnych od kontraktu biblioteki

Dwie decyzje wiszą dziś w powietrzu, bo zależą od tego, co biblioteka *akurat* zwróci. Obie należy
zakodować **w ACL (VO/adapter), nie w warstwie API ani w UI**.

*Źródło rozstrzygnięcia (uczciwie):* rozstrzygam je w oparciu o **specyfikację protokołu OpenGraph,
którą ta biblioteka implementuje** (`og:image:secure_url` = wariant HTTPS tego samego zasobu;
`og:image:width`/`height` są opcjonalne) — a **nie** o README samej biblioteki, którego nie
odczytałem (źródeł nie ma w lokalnym cache, patrz 4.0). Gdyby `go-opengraph` odbiegała od
specyfikacji, rozstrzygnięcie się nie zmienia — zmienia się tylko to, co adapter musi znormalizować,
i to jest dokładnie jego zadanie.

| Pytanie otwarte | Skąd dziś odpowiedź | Gdzie zakodować |
|---|---|---|
| Co, gdy strona nie poda `width`/`height` obrazka? (spec OG czyni je opcjonalnymi) | klient dokleja z `post.metadata.images`, a jak nie ma — wstawia `-1` (`post_attachment_opengraph.tsx:67-68`) | `LinkPreview.BestImage()` — serwer, mając już `PostMetadata.Images`, zna wymiary; `-1` jako „nieznane" nie wycieka do UI |
| Który URL jest autorytatywny: `url` czy `secure_url`? (spec OG: `og:image:secure_url` to wariant HTTPS) | reguła powielona 3× w 2 językach (5.2) | `Image.DisplayURL()` — jedno miejsce; proxy obrazków (`app/opengraph.go:133-147`) staje się jego jedynym konsumentem |

---

## KROK 6 — Weryfikacja i plan faz

### Kryterium sukcesu (wykonywalne)

```bash
grep -rn "dyatlov/go-opengraph" server --include=*.go | grep -v _test
```

**Dziś:** 8 plików w 3 warstwach (`model/`, `app/`, `store/storetest/`) — patrz KROK 1.
**Po refaktorze:** wyłącznie pliki w `server/platform/shared/linkpreview/dyatlov/`.

Kryterium uzupełniające (granica klient/serwer):
```bash
grep -rn "secure_url\|getBestImage" webapp/channels/src --include=*.tsx --include=*.ts | grep -v test
```
**Dziś:** 3 trafienia (`post_attachment_opengraph.tsx:56,63,199`). **Po:** 0.

### Kto dziś zna zależność, a kto nie będzie

| Przestaje wiedzieć | Nadal wie (i tak ma być) |
|---|---|
| `server/public/model/link_metadata.go` | `server/platform/shared/linkpreview/dyatlov/adapter.go` |
| `server/channels/app/opengraph.go` | `server/go.mod` |
| `server/channels/app/post_metadata.go` | (testy adaptera) |
| `server/channels/store/storetest/link_metadata_store.go` | |
| `webapp/platform/types/src/posts.ts` | |
| `webapp/channels/src/components/post_view/post_attachment_opengraph/*` | |

### Plan faz (konwencja `context/changes/<change-id>/`, zgodna z `01`/`02`)

| Faza | Zakres | Kryterium wyjścia |
|---|---|---|
| **1. Charakteryzacja** | Testy złotego wzorca (golden tests) na dzisiejszy JSON: HTML → JSON, oEmbed → JSON, JSON z bazy → JSON. **Bez zmian w kodzie produkcyjnym.** **Jawny cel: odczytać struct tagi `opengraph.OpenGraph` i rozstrzygnąć sprawę ośmiu wygaszanych pól z 4.0** (`omitempty` czy nie) — golden test musi pokrywać post po `TruncateOpenGraph`, bo tam te pola są zerowane. | Fixture'y zamrażające bajty wire formatu; **odpowiedź, czy usunięcie 8 pól w VO jest addytywne, czy łamiące** (bezpiecznik dla 4.0) |
| **2. VO + port + adapter** | Nowy pakiet `linkpreview` + adapter `dyatlov`. Stary kod jeszcze nietknięty. | `toDomain()` przechodzi golden testy z fazy 1 **bajt-w-bajt** |
| **3. Odwrócenie zależności w `app/`** | `app/opengraph.go` i `app/post_metadata.go` wołają port; `parseOpenGraph*` znikają. | `grep dyatlov server/channels/app` → 0 |
| **4. Oczyszczenie `model/`** | `TruncateOpenGraph`/`FilterSVGImages`/`firstNImages` → metody VO; `Data any` typowane VO; deserializacja przez adapter. | `grep dyatlov server/public/model` → 0; kryterium sukcesu spełnione po stronie Go |
| **5. Granica klient/serwer** | Addytywne `preview_image` (5.3); `getBestImage` usunięty z UI; `posts.ts` opisuje kontrakt Mattermosta, nie schemat biblioteki. | `grep secure_url\|getBestImage webapp` → 0 |
| **6. Dowód wymienialności** | Drugi adapter (choćby atrapa/„parser B") wpięty za port — bez dotykania bazy, API i UI. | Podmiana adaptera = 1 zmieniony plik + 1 linia wiring-u |

Fazy 1-2 są nieinwazyjne i można je scalić niezależnie. Fazy 3-4 dają całą wartość po stronie
serwera. Faza 5 jest jedyną, która dotyka kontraktu wire — i robi to **addytywnie**.

---

## Ograniczenia artefaktu

- **Brak dokumentu wymagań** (PRD/tech-stack) — nie ma deklaracji wymienialności do zacytowania
  (KROK 0). Diagnoza opiera się na kodzie, nie na spisanej intencji.
- **Analiza statyczna** (grep + odczyt plików) — kodu nie uruchamiano, testów nie odpalano. Nie
  zweryfikowano empirycznie, że VO serializuje się bajt-identycznie; to **założenie projektowe**,
  którego dowodem ma być faza 1 planu.
- **Nie odczytano źródeł samej biblioteki** — `go-opengraph` nie jest zwendorowana ani obecna w
  lokalnym module cache. Konsekwencja jest konkretna i wskazana w 4.0: nie wiem, czy osiem pól
  wygaszanych przez `TruncateOpenGraph` ma `omitempty`, a od tego zależy, czy VO jest bajt-identyczny.
  **To jedyne nierozstrzygnięte założenie, na którym stoi dowód izolacji z 5.1.**
- **Nie sprawdzono pluginów zewnętrznych** — `PostEmbed.Data` (`model/post_embed.go:24`) jest
  widoczne dla pluginów; realny zasięg opublikowanego kontraktu może być większy niż to repo.
- Cytaty `plik:linia` odnoszą się do stanu repo `mattermost/` na branchu `module-4-lesson-5`;
  numery linii mogą się przesunąć po edycjach.
- Osi porównawcze w KROKU 2 (`minio-go`, `squirrel`, biblioteki dat) zmierzono zgrubnie (liczba
  plików importujących), bez pełnego audytu każdej z nich.
