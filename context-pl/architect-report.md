---
title: Raport architektoniczny — Moduł 4 (ścieżka 10xArchitect)
created: 2026-08-01
type: architecture-summary
---

# Raport architektoniczny — Moduł 4 (ścieżka 10xArchitect)

> Two-pager oparty **wyłącznie** na czterech artefaktach modułu. Każde twierdzenie strukturalne
> (liczby, „tylko tutaj") pochodzi z artefaktu, nie z pamięci o kodzie. Gdzie czegoś brak — napisane
> wprost. Artefakty wejściowe:
> **L2** `context/map/repo-map.md` · **L3** `context/changes/config-surface/research.md` ·
> **L4** `context/changes/config-surface/plan.md` · **L5** `context/domain/01…03` (PL: `context-pl/domena/01…03`).

---

## 1. Opisane projekty

**Wszystkie cztery artefakty powstały na tym samym repozytorium** — monorepo **Mattermost**,
vendorowane w repo roboczym `shaping-claude`. Cel analizy jest przypięty do konkretnego checkoutu
`mattermost/` @ `ee04f28` (L3 czyta go z brancha `master`; L5 z brancha `module-4-lesson-5` — ten sam
checkout). Nie ma więc rozjazdu projektów między wejściami.

- **Stack:** backend Go (`server/`) + frontend TypeScript/React (`webapp/`) + stos e2e
  (`e2e-tests/`: Cypress gaśnie, Playwright rośnie). [L2 §1]
- **Skala (orientacyjnie, z artefaktów):** 284 pliki `server/public/model/*.go`; `config.go` = 5 765
  linii; intensywność prac ~2 600 zmian plików/mies. (kwi–maj 2026); jeden cykl SCC 1 070 modułów
  (~43% `channels`). [L2 §1,§4; L3 Feature overview; L5-01 KROK 0]
- **Gdzie pojawił się w module:** L2 (mapa całości), L3 (flow konfiguracji), L4 (plan refaktoru
  handlerów `api4`), L5 (destylacja domeny rdzenia komunikacyjnego).

## 2. Mapa projektu (L2)

1. **Dwie słabo połączone połowy.** Kręgosłup backendu (`model→store→app→api4`) i stos frontendu
   (`components↔redux↔platform`) prawie się nie współ-zmieniają; spoiwem są `server/model` (kontrakt)
   oraz szew tekstowy `i18n/en.json`, dotykany przy każdym ficzerze. [L2 §1,§3]
2. **Strefa ryzyka #1 = `admin_console`.** Najczęściej edytowany obszar, a **58% (330/571)** tkwi
   w cyklu 1 070 modułów — obszar dotykany najczęściej jest zarazem najmniej bezpieczny. [L2 §4]
3. **Huby „wszyscy dotykają" (config surface).** `config.go`, `config.ts`, `admin_definition.tsx`,
   `client4` — każdy na szczycie listy najczęściej modyfikowanych; potencjalny ripple cross-stack
   mapa oznaczyła jako **inference, nie zmierzoną parę** (Risk Zone 4). [L2 §4]
4. **Największy unknown.** Import graph backendu Go i e2e jest `[unknown]` — dependency-cruiser puszczono
   **tylko na `webapp/`**. Blast radius backendu jest statycznie nieweryfikowalny; opiera się na dyscyplinie
   współ-zmian `model` i recenzentach. [L2 §3,§4,§7]
5. **Entry pointy / fundamenty.** Backendowy hub odpowiedzialności = `app`; czysty fundament frontendu =
   `platform/types` (duży fan-in, **zero** fan-out) — bezpieczny punkt startu. [L2 §2]

## 3. Analiza ficzera (L3)

**Który flow i dlaczego.** Prześledzony został **configuration surface**, zakotwiczony w
`server/public/model/config.go` — wybrany, bo mapa (Research Objective + Risk Zone 4) wskazała go jako
hub #1 modyfikowany przy niemal każdym ficzerze. [L3 Research Question; L2 §4]

**Feature overview.** To 5-warstwowa rura `api4 → app → platform → config.Store → BackingStore{File|DB|Memory}`.
Odczyty serwowane są w całości z cache in-memory (bez uderzenia w backend); zapisy schodzą w dół, a po
zapisie **synchronicznie** rozchodzą się przez emitter do subskrybentów runtime (regeneracja client-config,
broadcast WS `config_changed`, rekonfiguracja loggera i wyszukiwarki) plus fan-out klastra (HA). [L3 Summary]

**Technical debt (najważniejsze ryzyka):**
1. **Duplikacja pipeline'u zapisu — potwierdzona ast-grepem.** `updateConfig` i `patchConfig` mają
   **własne, ręcznie kopiowane listy pól chronionych** (bez wspólnego wrappera): pola strażnicze
   `api4/config.go:155/160/163/167/174` vs `:307/314/318/326/334`. To mechanizm klasy błędu MM-68976.
   **Uwaga (historyczność):** na `ee04f28` `patchConfig` **już chroni wszystkie 5 pól** — defekt załatany
   `c45a675553` (2026-06-15); pozostaje trwałe *ryzyko dryfu* i luka test-parytetu (N4). [L3 C3, Claim
   verification (ast-grep); L4 baseline]
2. **Luka testowa — potwierdzona ast-grepem.** `localGetClientConfig` to jedyny handler local-mode
   z **zerowym** pokryciem (ast-grep = 0, potwierdzone `grep` = 0). Dodatkowo `DatabaseStore` testowany
   tylko na Postgresie (`main_test.go:48`). [L3 Debt #1, #2]
3. **Config surface = szew backend↔frontend.** Mapa nazwała ripple „inference"; git to obala:
   `config.go ↔ config.ts` **34×**, `config.go ↔ admin_definition.tsx` **24×**. To dowód klasy `[git]` —
   świadomie **poza zakresem ast-grep** (nie ast-grep-potwierdzony). Blast radius pola = 8 plików w wielu
   stackach; **field-add nie wymaga migracji** (persystencja field-agnostic). [L3 Debt #3, Blast radius]

## 4. Plan refaktoryzacji (L4)

**Co refaktoryzowane.** Wybrana opcja **#1 (C3)** — de-duplikacja **rejestru strażników**, nie handlerów.
Docelowy kształt: jeden pakietowy **rejestr pól chronionych** w `api4/config.go`, z którego oba handlery
czytają zestaw pól; **table-driven parity test** failuje CI, gdy handler przestanie chronić pole z rejestru.
To **czysty refaktor** — zachowanie każdego endpointu zachowane co do bajta (`update` cicho koeruje, `patch`
zwraca 403). [L4 Overview, Desired End State]

**Czego świadomie NIE robimy.** Nie ujednolicamy stylu egzekwowania (coerce vs 403) — to decyzja kontraktu
API dla product/API ownerów, nie refaktor. Nie łączymy `updateConfig` i `patchConfig` w jeden handler.
Nie ruszamy handlerów local-mode (`config_local.go`). Nie przenosimy rejestru do `model`. Nie tykamy innych
pozycji researchu (C2, N2, współbieżność emittera). [L4 What We're NOT Doing]

**Fazy (jak weryfikowane):**
- **Faza 1 — testy charakteryzujące (brama):** dodać brakujący subtest cloud dla `patchConfig` (N4) +
  przypiąć coerce-vs-403 dla 3 rozjeżdżających się pól. → **auto** (`go test … api4`, `golangci-lint`)
  **+ ręcznie** (reviewer potwierdza, że testy pinują obecne zachowanie; brak zmian w kodzie produkcyjnym).
- **Faza 2 — ekstrakcja rejestru + drift guard:** pakietowy rejestr pól, podpięcie obu handlerów,
  `TestConfigGuardRegistryParity`. → **auto** (testy, lint, `go build`, negative control: usunięcie pola
  z handlera failuje test) **+ ręcznie** (diff zachowania = zero zmian; jedno źródło prawdy dla listy pól).
  [L4 Phase 1/2 → Success Criteria]

## 5. Domena wg DDD (L5)

**Ubiquitous language (rdzeń komunikacyjny).** `Team` → `Channel` → `ChannelMember` → `Post`/`Thread` (RootId),
z nakładką RBAC `Role/Permission/Scheme`. Rdzeń = ta ścieżka; configuration surface — mimo najwyższej
aktywności git — sklasyfikowany jako **generic/supporting**, nie rdzeń domeny. [L5-01 KROK 1, KROK 2]

**Najważniejszy rozjazd model-vs-kod.** Wiedza domenowa żyje w **czasownikach** (`server/channels/app/`),
a nie w **rzeczownikach** (`server/public/model/`). `IsValid()` w `model/` waliduje kształt pól, **nie** prawdę
biznesową — encje są anemiczne. Najgroźniejszy przypadek: reguła #1 poniżej, którą można *poprosić* o pominięcie
flagą boolowską. [L5-01 KROK 4]

**Niezmiennik #1 i jego agregat.** I-1: **„członek kanału musi być aktywnym członkiem Teamu kanału"**.
Dziś egzekwowany w jednej metodzie serwisowej z opt-out (`skipTeamMemberIntegrityCheck`, `app/channel.go:1887`),
nie gwarantowany przez konstrukcję. Docelowy agregat-strażnik: **`ChannelMembership`** (Channel + ChannelMembers),
czyniący niezmiennik prawdziwym *by construction*. [L5-01 KROK 5; L5-02]

**Anti-Corruption Layer.** Przeciekająca zależność #1 = **`github.com/dyatlov/go-opengraph`** (typ biblioteki
jako pole kontraktu `Data any` i w sygnaturach domenowych, kształt ręcznie odtworzony po stronie TS). Zasięg
przecieku: **7 warstw wg tabeli klasyfikacji (L5-03 KROK 2)** — po stronie Go **8 plików w 3 warstwach**
(`model/`, `app/`, `store/storetest/`), plus szew wire/TS i UI. Docelowo: value object + port + **adapter**
(jedyny plik importujący bibliotekę). [L5-03 KROK 1, KROK 4]

## 6. Decyzje, które należą do mnie

AI wygenerowało mapę (L2), zrekonstruowało flow i policzyło dług (L3) oraz **zaproponowało** ranking okazji
refaktorowych — research explicite zaznacza, że ranking to „propozycja dla osobnej sesji planowania, **nie
decyzja**". Rozstrzygnięcia po mojej stronie, zapisane na etapie planowania: **(a)** który flow prześledzić —
config surface, kierując się Research Objective/Risk Zone 4 z mapy; **(b)** którą okazję zaplanować — **#1 (C3)**
zamiast C2, bo niski koszt zmiany (jeden plik) i udokumentowana klasa defektu; **(c)** gdzie postawić granicę
zakresu — nie ujednolicać egzekwowania (coerce vs 403) jako decyzji kontraktu API oraz odesłać R1/R2 jako
redesign zachowania, nie strukturę (L4 „What We're NOT Doing"; L3 candidate audit). *Osobiste uzasadnienie poza
tym, co utrwalają artefakty, nie jest w nich zapisane — do uzupełnienia z pamięci autora.*
