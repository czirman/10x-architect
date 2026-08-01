---
title: Refaktor niezmiennik → agregat — strażnik ChannelMembership
created: 2026-07-11
type: refactor-plan
---

# Refaktor niezmiennik → agregat: agregat-strażnik `ChannelMembership`

> **Ten dokument jest PLANEM, nie implementacją.** Nie zmodyfikowano żadnego kodu produkcyjnego.
> Każdy cytat `plik:linia` poniżej został w tej sesji faktycznie otwarty i przeczytany. Ścieżki są
> względne do `mattermost/` (checkout na gałęzi `module-4-lesson-5`); numery linii mogą się
> przesunąć po edycjach.
>
> Poprzednik: `context-pl/domena/01-destylacja-domeny.md`. Tamten dokument uszeregował
> kandydatów. **Listę niezmienników wyprowadziłem od nowa i zweryfikowałem każdy cytat
> niezależnie** — wniosek jest zbieżny, ale diagnoza poniżej jest ostrzejsza niż w 01 i **koryguje
> go w dwóch miejscach** (oznaczone ⚠️).
>
> Wersja angielska (kanoniczna): `context/domain/02-invariant-aggregate-refactor.md`.

---

## KROK 0 — Odkryty kontekst

**Dokumenty wymagań: nie istnieją.** W tym repozytorium nie ma PRD, wizji ani `tech-stack.md`.
Jedyną narracją produktową jest `README.md:3` — *„open core, self-hosted collaboration platform
that offers chat, workflow automation, voice calling, screen sharing and AI integration"*. Nie ma
spisanych sekcji „business logic" ani „success criteria".

**Konsekwencja dla tego planu (kluczowe ograniczenie):** reguły biznesowe są odtworzone z
*zachowania kodu*, nie z zapisanej intencji. „Domena wymaga X" oznacza poniżej zawsze „kod
egzekwuje X w co najmniej jednym miejscu", nigdy „zespół zapisał X". Tam, gdzie wnioskuję o
intencji produktowej (np. „Team jest granicą dostępu"), mówię o tym wprost.

**Stack i warstwy, w których żyje logika biznesowa** (zweryfikowane z drzewa katalogów):

| Warstwa | Ścieżka | Co tam faktycznie żyje |
|---|---|---|
| API (HTTP) | `server/channels/api4/` | Routing, parsowanie parametrów, sprawdzanie uprawnień. Handlery **nie są** cienkie — niosą autoryzację. |
| Aplikacja / serwis | `server/channels/app/` | **Tu żyją niezmienniki międzybytowe.** Reguły biznesowe są czasownikami na `*App`. |
| Model domenowy | `server/public/model/*.go` | Struktury + `IsValid()`. ⚠️ `IsValid()` to **walidacja strukturalna** (format ID, enum, długości) — nie zawiera żadnej reguły międzybytowej. Encje są **anemiczne**. |
| Persystencja | `server/channels/store/sqlstore/` | SQL plus opsowy checker integralności (`integrity.go`). |
| UI | `webapp/` (React/TS) | Renderuje to, na co pozwala API. |

Istotny fakt architektoniczny: **nie ma warstwy agregatów.** Reguła jest albo `if`-em wewnątrz
metody `*App`, albo nie istnieje w runtime.

---

## KROK 1 — Zidentyfikowane niezmienniki biznesowe

Reguły, które w tej domenie muszą być zawsze prawdziwe, wyciągnięte z kodu (nie ma dokumentu
wymagań, z którego można by je wyciągnąć). Każda z zweryfikowanym cytatem.

| # | Niezmiennik (tak, jak implikuje go kod) | Gdzie reguła jest wyrażona |
|---|---|---|
| **I-1** | **Członek kanału musi być aktywnym członkiem Teamu, do którego należy kanał.** | `app/channel.go:1888-1903` — ładuje `Team().GetMember(channel.TeamId, user.Id)`; odrzuca przy braku członkostwa i przy `teamMember.DeleteAt > 0`. Bramkowane parametrem `skipTeamMemberIntegrityCheck` (`app/channel.go:1887`). |
| **I-2** | **Odpowiedź żyje w tym samym kanale, co korzeń wątku.** | `app/post.go:300` — `!parentPostList.IsChannelId(post.ChannelId)` → błąd. |
| **I-3** | **Wątek jest płaski** (nie można odpowiedzieć na odpowiedź). | `app/post.go:304-306` — `if rootPost.RootId != ""` → błąd. |
| **I-4** | **Group Message ma 3–8 członków.** | `app/channel.go:562` — `len(userIDs) > model.ChannelGroupMaxUsers \|\| < ChannelGroupMinUsers` → błąd. Stałe w `public/model/channel.go:37-38`. |
| **I-5** | **Direct Message ma dokładnie 2 członków.** | Nigdzie nie jest asertowane. Wynika z *arności sygnatury funkcji*: `createDirectChannelWithUser(rctx, user, otherUser, ...)` `app/channel.go:466`. |
| **I-6** | **Członkostwo kanału group-constrained równa się jego powiązanej grupie.** | `app/channel.go:1802` — `if channel.IsGroupConstrained()` → `FilterNonGroupChannelMembers`. Utrzymywane przez proces synchronizacji w tle, nie przez jednego strażnika. |

**Kształt całej listy:** każdy z tych niezmienników żyje w `server/channels/app/` jako `if`
wewnątrz czasownika serwisowego. **Ani jeden nie jest wyrażony na encji, którą ogranicza.** To
jest ustalenie strukturalne — wiedza domenowa jest realna i bogata, ale siedzi w **czasownikach**,
nie w **rzeczownikach**.

---

## KROK 2 — Klasyfikacja i wybór

Trzy osie, zgodnie z zadaniem: **(a)** jak rdzeniowy dla sensu produktu, **(b)** jak rozsmarowany
po warstwach, **(c)** czy jest *egzekwowany* / tylko *deklarowany* / *naruszalny*.

| # | (a) Rdzeniowość | (b) Rozsmarowanie | (c) Status egzekwowania |
|---|---|---|---|
| **I-1** Członek kanału ⊆ członek Teamu | **Najwyższa.** Team jest granicą dostępu w produkcie; kanał należy do dokładnie jednego teamu. Jeśli to pęknie, użytkownik czyta rozmowy teamu, w którym go nie ma. To reguła **kontroli dostępu**, nie reguła porządkowa. | **Najgorsze: 2 agregaty, 3 pliki, oba kierunki.** Strona dodawania w `app/channel.go`, strona usuwania w `app/team.go`, furtka group-sync w `app/syncables.go`, i *brak pokrycia* w `store/sqlstore/integrity.go`. | **Naruszalny.** Egzekwowany tylko przy dodawaniu, a ten check **da się wyłączyć parametrem boolean**. Strona usuwania to proceduralne sprzątanie bez transakcji i bez właściciela. **Brak ograniczenia w bazie.** |
| **I-2** odpowiedź w kanale korzenia | Wysoka (integralność wątku) | Niskie — jedno miejsce, `app/post.go`. | Egzekwowany (bezwarunkowo). |
| **I-3** płaski wątek | Wysoka (integralność wątku) | Niskie — jedno miejsce, `app/post.go`. | Egzekwowany (bezwarunkowo). |
| **I-4** GM 3–8 | Średnia | Niskie — jedno miejsce, przy tworzeniu. | Egzekwowany przy tworzeniu. Nic nie sprawdza ponownie po późniejszym dodaniu członków. |
| **I-5** DM = 2 | Średnia | Niskie | **Tylko deklarowany, strukturalnie** — gwarantowany 2-argumentową sygnaturą. Z zasady kruchy, ale trudny do naruszenia przypadkiem. |
| **I-6** group-constrained ≡ grupa | Średnia (poddomena wspierająca — integracja tożsamości, nie czat) | Wysokie — ≥3 miejsca + job synchronizujący. | Ostatecznie spójny **z projektu**. To *proces*, nie niezmiennik. Refaktor oznaczałby przeprojektowanie synchronizacji — nieproporcjonalnie. |

### Wybór: **I-1 — „członek kanału musi być aktywnym członkiem Teamu"**

To jedyny kandydat, który jest **jednocześnie najbardziej rdzeniowy i najsłabiej egzekwowany**.
I-2/I-3 są bliskie rdzeniowi, ale są *już poprawnie egzekwowane* (jedno miejsce, brak furtki) —
ich refaktor byłby kosmetyczny. I-6 jest fatalnie rozsmarowany, ale siedzi w poddomenie
**wspierającej** i jest *zaprojektowany* jako ostatecznie spójny. To przy I-1 wysoka wartość i
słabe egzekwowanie faktycznie się pokrywają.

**Co podnosi I-1 z „dryfu spójności" do „usterki bezpieczeństwa"** — i to jest ustalenie
rozstrzygające o całym planie — to fakt, że **ścieżka odczytu nigdy nie sprawdza ponownie
członkostwa w Teamie**:

```
app/authorization.go:466  HasPermissionToReadChannel(userID, channel)
app/authorization.go:467    → HasPermissionToChannel(userID, channel.Id, PermissionReadChannelContent)
app/authorization.go:337-347  → czyta role z ChannelMembers → RolesGrantPermission → return true
```

`HasPermissionToChannel` (`app/authorization.go:327-347`) decyduje **wyłącznie** na podstawie
wierszy użytkownika w **`ChannelMembers`**. Członkostwo w Teamie jest brane pod uwagę **tylko jako
fallback dla użytkowników, którzy *nie* są członkami**, i tylko dla kanałów otwartych
(`app/authorization.go:471-472`). Zatem dla kanału **prywatnego** wiersz w `ChannelMembers` jest —
sam z siebie — trwałym nadaniem prawa odczytu.

**Wniosek: osierocony wiersz `ChannelMember` to nie jest nieaktualny rekord. To stale obowiązujące
nadanie prawa odczytu i zapisu do prywatnej rozmowy w teamie, z którego użytkownik już wyszedł.**
I to właśnie agregat ma uczynić **niereprezentowalnym**.

---

## KROK 3 — Diagnoza I-1

### 3.1 Gdzie dziś żyje reguła (wszystkie warstwy)

**Strona dodawania — egzekwowana, ale wyłączalna.**

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

Niezmiennik jest **parametrem**. Co więcej, jest awansowany do publicznej struktury opcji — więc
każdy wołający, łącznie z pluginami, wyraża go jako **dane**:

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
…a `AddChannelMember` (`app/channel.go:1938`) przekazuje go dosłownie dalej w `app/channel.go:1966`.

**Zweryfikowane produkcyjne miejsca wywołania** (bez `_test.go`):

| Miejsce wywołania | Skip? | Werdykt |
|---|---|---|
| `app/channel.go:2653` (`JoinChannel`) | `false` | Sprawdzane. |
| `app/team.go:721` | `false` | Sprawdzane. |
| `app/syncables.go:69` | **`true`** | **Jedyne** `true` w produkcji. Członek teamu został właśnie utworzony przez tę samą synchronizację, więc check jest redundantny — **nie** jest to złośliwa furtka. |

⚠️ **Korekta ujęcia z dokumentu 01:** prywatna `addUserToChannel` (`app/channel.go:1787`) **nie
jest** niezależną ścieżką obejścia. Jej jedynym produkcyjnym wołającym jest `app/channel.go:1905`,
czyli wnętrze sprawdzanej metody publicznej, **po** checku. Zweryfikowałem to gerpem. Prawdziwe
dziury są dwie poniżej, a nie „dzika ścieżka wywołania".

**Dziura 1 — niezmiennik jest z założenia opt-out.** Dzisiejsze jedno `true` jest niegroźne. Ale
*prawdziwość* reguły opiera się na **dyscyplinie miejsc wywołania**, nie na konstrukcji. Nic nie
powstrzyma następnego wołającego — albo pluginu — przed podaniem `true` i zapisaniem osieroconego
wiersza. **Niezmiennik domenowy, który przyjmuje `bool` do swojego wyłączenia, nie jest
niezmiennikiem — jest wartością domyślną.**

**Strona usuwania — proceduralne sprzątanie, nie egzekwowanie, i bez atomowości.**

`app/team.go:1363` `LeaveTeam` to jedyna rzecz utrzymująca prawdziwość I-1, gdy użytkownik
opuszcza team. Wykonuje po kolei i **bez żadnej transakcji**:

1. `Channel().GetChannels(team.Id, user.Id, ...)` — odczyt kanałów użytkownika w tym teamie.
2. pętlę `for` wołającą `a.removeChannelMembership(rctx, user.Id, channel.Id, "LeaveTeam")` per
   kanał (DM/GM pomijane — słusznie, nie mają teamu).
3. `a.ch.srv.teamService.RemoveTeamMember(rctx, teamMember)` — usunięcie członkostwa w teamie.
4. `a.postProcessTeamMemberLeave(...)` (`app/team.go:1318`) — inwalidacja cache, sidebar,
   preferencje.

Samo `removeChannelMembership` (`app/channel.go:2886-2894`) to **dwa niezależne zapisy do store'a**
bez transakcji:
```go
if err := a.Srv().Store().Channel().RemoveMember(rctx, channelID, userID); err != nil { ... }
if err := a.Srv().Store().Thread().DeleteMembershipsForChannel(userID, channelID); err != nil { ... }
```

**Dziura 2 — N+1 zapisów, brak transakcji, brak właściciela.** Usunięcie użytkownika z teamu o 40
kanałach to 80+ osobnych zapisów bez żadnej granicy atomowej. *Kolejność* jest dobrana obronnie
(najpierw kanały, potem członek teamu), więc crash w środku pętli zostawia użytkownika nadal w
teamie — to bezpieczny kierunek. Ale sekwencja nie jest atomowa i **nikt nie jest właścicielem tej
pary**.

**Konkretny sposób, w jaki I-1 pęka — oznaczony jako scenariusz, nie jako zaobserwowane
zachowanie.** Nie da się dowieść braku blokady samym czytaniem kodu, więc podaję to jako tryb
awarii, na który projekt pozwala, a nie jako buga, którego odtworzyłem:

> Żądanie A (`LeaveTeam`) czyta listę kanałów użytkownika w kroku 1 i zaczyna usuwać członkostwa.
> Równolegle żądanie B (`AddChannelMember` na prywatnym kanale w tym teamie) wykonuje swój check
> przy dodawaniu — członkostwo w teamie **wciąż istnieje**, więc check przechodzi — i zapisuje nowy
> wiersz w `ChannelMembers`. Żądanie A kończy krok 3 i usuwa członkostwo w teamie. Wiersz zapisany
> przez B jest teraz osierocony. Zgodnie z KROKIEM 2 ten wiersz jest **stale obowiązującym prawem
> odczytu prywatnego kanału dla osoby spoza teamu.** Nic go później nie odbiera.

**Dziura 3 — *wejściem* sprzątania jest zapytanie, którego „nie znaleziono" jest połykane jako
„nie ma nic do roboty".** Pierwszy krok `LeaveTeam` (`app/team.go:1371-1381`):

```go
if channelList, nErr = a.Srv().Store().Channel().GetChannels(team.Id, user.Id, ...); nErr != nil {
	var nfErr *store.ErrNotFound
	if errors.As(nErr, &nfErr) {
		channelList = model.ChannelList{}   // ← ErrNotFound reinterpretowany jako „user nie ma kanałów"
	} else {
		return model.NewAppError("LeaveTeam", ...)
	}
}
```

Dziś jest to **niemal na pewno niegroźne** — store zwraca `ErrNotFound` dla pustego zbioru wyników,
więc „nie znaleziono" naprawdę znaczy „brak kanałów". Ale zwróć uwagę, czym to jest strukturalnie:
**jedyny strażnik strony usuwania I-1 traktuje nieudany odczyt jako pusty odczyt, po czym i tak
usuwa członkostwo w teamie.** Gdyby to zapytanie kiedykolwiek zwróciło `ErrNotFound` z jakiegokolwiek
innego powodu niż faktyczna pustka, **wszystkie** członkostwa kanałowe użytkownika zostaną osierocone
*cicho i deterministycznie* — bez błędu, bez logu, bez detektora (patrz niżej). Sprzątanie zaraportuje
sukces, nie posprzątawszy niczego. To założenie nośne trzymane w kupie konwencją warstwy store, a
projekt agregatu je usuwa: po refaktorze usuwanie jest zbiorczym `DELETE` w tej samej transakcji,
więc „zero pasujących wierszy" i „odczyt się nie powiódł" nie da się już pomylić.

**Wołający wsadowi logują-i-jadą dalej (dotyczy I-6, nie I-1).**
`DeleteGroupConstrainedTeamMemberships` (`app/syncables.go:161-183`) iteruje po użytkownikach do
usunięcia i woła `a.RemoveUserFromTeam` — które poprawnie przechodzi przez `LeaveTeam`, więc
sprzątanie kanałów *jednak* się dzieje per użytkownik. Ale przy błędzie dopisuje do `multierror` i
robi **`continue`** (`syncables.go:174-178`). Stan per-użytkownik pozostaje spójny, więc **I-1
przeżywa** — ale użytkownik, który *powinien* zostać usunięty z teamu group-constrained, cicho w nim
zostaje, a job raportuje częściowy sukces. To naruszenie fail-fast wobec **I-6** i notuję je tutaj,
zamiast wtapiać w diagnozę I-1, bo zlanie ich razem **przeszacowałoby tezę, którą stawiam**.

**Warstwa persystencji — nie egzekwuje niczego i nawet nie *widzi* naruszenia.**

**Nie ma klucza obcego** z `ChannelMembers` do `TeamMembers` (prostego być nie może — relacja jest
przechodnia przez `Channels`). Gorzej: opsowy checker integralności tego nie pokrywa. Zweryfikowane
w `store/sqlstore/integrity.go`:

| Obecny check | Linia |
|---|---|
| `Channels` → `ChannelMembers` | `integrity.go:103-105` |
| `Users` → `ChannelMembers` | `integrity.go:302-304` |
| `Teams` → `TeamMembers` | `integrity.go:265-267` |
| **`Teams` → `ChannelMembers`** | **BRAK** |

Zatem naruszenie I-1 **nie jest wykrywalne narzędziem zbudowanym dokładnie do wykrywania tej klasy
problemu.** Nie ma joba uzgadniającego ani alarmu. Stan raz zepsuty **pozostaje zepsuty w ciszy**.

### 3.2 Podsumowanie diagnozy

| Warstwa | Czy egzekwuje I-1? |
|---|---|
| UI (`webapp/`) | Nie — i nie musi. ✅ *To NIE jest przypadek „klient jedynym strażnikiem".* |
| API (`api4/`) | **Nie.** `addChannelMember` (`api4/channel.go:2289+`) parsuje `user_ids`/`post_root_id` i sprawdza *uprawnienia*; nigdy nie wspomina o niezmienniku teamu. |
| App (`app/`) | **Częściowo.** Dodawanie: tak, ale z furtką. Usuwanie: nieatomowe sprzątanie w innym pliku, na innym agregacie. |
| Model domenowy (`model/`) | **Nie.** `ChannelMember.IsValid()` nic nie wie o Teamach. Reguła jest niewidzialna dla typu. |
| Persystencja | **Nie.** Brak FK, brak checku integralności, brak uzgadniania. |

⚠️ **Druga korekta oczekiwanej narracji — o fail-fast, precyzyjnie.** Zadanie zakłada „błąd połykany
zamiast zatrzymywać operację". Na *głównych* ścieżkach I-1 **nie znalazłem tego**: `AddUserToChannel`
i pętla usuwająca w `LeaveTeam` obie propagują błędy i przerywają. Podstawową usterką jest **brak
atomowości plus brak właściciela**, a nie log-and-continue. Są dokładnie dwa prawdziwe połknięcia i
zakreślam je uczciwie, zamiast je nadmuchiwać:
- **Dziura 3** — `ErrNotFound` reinterpretowany jako „brak kanałów" w jedynym zapytaniu, od którego
  zależy sprzątanie (`app/team.go:1375-1377`). Latentne, obecnie nie odpala.
- **`app/syncables.go:174-178`** — `continue`-przy-błędzie w wsadzie eksmisji group-sync. Realne, ale
  degraduje **I-6**, nie I-1.

**Diagnoza w jednym zdaniu:** *I-1 to rdzeniowy niezmiennik kontroli dostępu, którego nie posiada
żaden obiekt — wyłączalny warunek wstępny na jednym agregacie plus nieatomowe, nieserializowane
sprzątanie na drugim, bez zabezpieczenia w bazie, bez detektora w narzędziach opsowych i ze ścieżką
odczytu uprawnień, która traktuje powstały osierocony wiersz jako ważne nadanie.*

---

## KROK 4 — Projekt: agregat-strażnik `ChannelMembership`

**Ograniczenie projektowe — musi pasować do Go i warstwowania App/Store Mattermosta.** Nie proponuję
przeszczepu Java DDD. Konkretnie: nowy pakiet domenowy trzymający regułę jako *typ*, metoda store'a,
która ładuje i zapisuje ją *jako całość i w jednej transakcji*, oraz metody `*App`, które stają się
cienkimi delegatami. To przeżywa istniejące plugin API i `api4` nietknięte na poziomie sygnatur.

### 4.1 Korzeń agregatu

Nowy pakiet `server/channels/app/membership/` (domena, bez importów store/HTTP — bezzależnościowy, a
więc testowalny jednostkowo bez bazy):

```go
package membership

// TeamMembership to strzeżony fakt: status tego użytkownika w tym teamie, tu i teraz.
type TeamMembership struct {
	teamID   string
	userID   string
	deleteAt int64
}

func (tm TeamMembership) IsActive() bool { return tm.deleteAt == 0 }

// ChannelMembership to KORZEŃ AGREGATU dla niezmiennika I-1.
// Konstruowany wyłącznie z kanału + statusu aktora w teamie, więc
// ChannelMembership naruszający I-1 NIE DA SIĘ SKONSTRUOWAĆ.
type ChannelMembership struct {
	channelID string
	teamID    string
	userID    string
	roles     string
}
```

### 4.2 Nazwane błędy domenowe (fail-fast, żadnej cichej zmiany stanu)

```go
type ErrNotTeamMember struct{ UserID, TeamID string }         // nigdy nie był członkiem
type ErrTeamMembershipRevoked struct{ UserID, TeamID string } // był, DeleteAt > 0
type ErrOrphanedMembership struct{ UserID, ChannelID string } // wykryte przy ładowaniu
```
Każdy implementuje `error`. **Nic w tym pakiecie nie loguje.** Każda nielegalna operacja *zwraca*.

### 4.3 Metoda domenowa z warunkami wstępnymi

```go
// Grant to JEDYNY sposób wyprodukowania ChannelMembership.
// Warunki wstępne (I-1) — żaden parametr ich nie wyłącza:
//   P1: channel.TeamId == teamMembership.teamID   (właściwy team)
//   P2: teamMembership.IsActive()                 (aktywny, nie soft-deleted)
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

**Zmiana nośna:** boolean `skipTeamMemberIntegrityCheck` **nie ma tu odpowiednika — i to celowo.**
Przypadek synchronizacji (`app/syncables.go:69`), który dziś podaje `true` wyłącznie po to, by uniknąć
redundantnego odczytu, jest obsłużony inaczej: *przekazuje `TeamMembership`, które właśnie utworzył* —
spełnia warunek wstępny wartością, którą już trzyma w ręku, zamiast prosić o jego pominięcie.
**Optymalizacja przeżywa; furtka nie.** I o to w tym refaktorze chodzi: I-1 przestaje być wartością
domyślną, a staje się konstruktorem.

Kanały DM/GM (`ChannelType` `D`/`G`) nie mają `TeamId`; trafiają do osobnego konstruktora
`GrantDirect(...)`, który niesie zamiast tego I-4/I-5 — więc kanał bez teamu nigdy nie przeleci cicho
przez check I-1.

### 4.4 Repozytorium — ładuje i zapisuje agregat jako całość, w JEDNEJ transakcji

Dziś `LeaveTeam` wykonuje N+1 nieograniczonych zapisów przez dwa store'y. Niezmiennik potrzebuje
atomowości, więc dostaje transakcję. **Prymityw już istnieje w tej bazie kodu** — store używa
`GetMaster().Begin()` + `defer finalizeTransactionX(transaction, &err)`, np.
`store/sqlstore/channel_store.go:634-639`. Nowa metoda idzie dokładnie tym wzorcem:

```go
// store/store.go (interfejs)
type ChannelStore interface {
	...
	// GrantMembership stosuje niezmiennik i zapisuje, wewnątrz JEDNEJ transakcji,
	// trzymając blokadę wiersza na TeamMembers aktora. Albo powstaje legalne
	// członkostwo, albo zwracany jest nazwany błąd domenowy i NIC nie jest zapisane.
	GrantMembership(rctx request.CTX, channelID, userID string, rule membership.GrantFunc) (*model.ChannelMember, error)

	// RevokeAllForTeamMember usuwa członkostwo w teamie ORAZ każde członkostwo
	// kanałowe w tym teamie, atomowo, biorąc TĘ SAMĄ blokadę wiersza.
	// Zastępuje pętlę z LeaveTeam.
	RevokeAllForTeamMember(rctx request.CTX, teamID, userID string) error
}
```

Obie metody blokują **ten sam** wiersz `TeamMembers`. **To ta współdzielona blokada — a nie typ
agregatu — czyni I-1 prawdziwym przy współbieżności.**

`RevokeAllForTeamMember` wykonuje, wewnątrz jednej transakcji:
1. `SELECT … FROM TeamMembers WHERE TeamId=? AND UserId=? FOR UPDATE` — **najpierw bierze blokadę
   wiersza** (patrz niżej);
2. `DELETE FROM ChannelMembers` dla każdego kanału tego teamu spoza DM/GM (**jedna instrukcja
   zbiorcza**, zastępująca pętlę N-iteracyjną);
3. odpowiadające usunięcie `ThreadMemberships` (dziś nieotransakcjonowany drugi zapis w
   `app/channel.go:2887-2892`);
4. update/delete na `TeamMembers`.

Albo wszystko wchodzi, albo nic. **To zamyka Dziurę 2** — dziurę crash/stan-częściowy.

#### ⚠️ Sama atomowość po stronie usuwania NIE zamyka wyścigu — strona dodawania też musi blokować

To najsubtelniejsza część projektu i najłatwiejsza do zepsucia. Opakowanie *tylko*
`RevokeAllForTeamMember` w transakcję pozostawia scenariusz TOCTOU z §3.1 **całkowicie nietkniętym**.
Na domyślnym poziomie izolacji obu wspieranych baz (`READ COMMITTED`) te dwie transakcje nie kolidują
na żadnym wierszu, więc nic ich nie serializuje:

```
Revoke TX:  DELETE FROM ChannelMembers WHERE …   -- wiersz Add jeszcze nie istnieje → nic nie usuwa
            DELETE FROM TeamMembers     WHERE …
            COMMIT
Add TX:     odczyt TeamMembers → wciąż aktywne (odczyt przed commitem Revoke)
            INSERT INTO ChannelMembers …          -- commituje PO tym, jak DELETE już przeszedł
            COMMIT
                                                  → osierocony wiersz przeżywa. Niezmiennik złamany.
```

`DELETE` nie może usunąć wiersza, który jeszcze nie został wstawiony, a `INSERT` został autoryzowany
odczytem członkostwa w teamie, które właśnie miało zniknąć. **Atomowość to nie jest wzajemne
wykluczanie.**

Dlatego **ścieżka dodawania też musi być jedną transakcją, a `GrantMembership` musi wziąć
`SELECT … FOR UPDATE` na wierszu `TeamMembers`** — tym samym, który `RevokeAllForTeamMember` blokuje
w swoim kroku 1. Ten wiersz staje się **punktem serializacji** niezmiennika: Add i Revoke rywalizują
o niego, a ta operacja, która idzie druga, widzi zacommitowaną prawdę tej pierwszej. Konkretnie:

- Revoke commituje pierwszy → odczyt `FOR UPDATE` w Add blokuje się, po czym zwraca *odwołane*
  członkostwo → `Grant` zwraca `ErrTeamMembershipRevoked` → **żaden wiersz nie zostaje zapisany.**
- Add commituje pierwszy → `DELETE FROM ChannelMembers` w Revoke blokuje się, po czym wykonuje się
  *po* tym, jak insert jest widoczny → **nowy wiersz zostaje usunięty razem z resztą.**

Oba przeplecenia zachowują I-1. (Izolacja `SERIALIZABLE` też by zadziałała, ale nałożyłaby ciężar
retry-on-conflict na tę bardzo gorącą ścieżkę; pojedyncza blokada wiersza jest tańsza i celniejsza.)

**Dlatego agregat potrzebuje repozytorium, a nie tylko konstruktora:** reguła jest tak prawdziwa, jak
granica, którą oba zapisy dzielą. `membership.Grant` czyni niezmiennik *niereprezentowalnym w
pamięci*; blokada wiersza czyni go *niereprezentowalnym w bazie*. **Obie połowy są nośne** — wypuszczenie
Fazy 1 bez blokady z Fazy 2 dałoby projekt, który *czyta się* poprawnie i **dalej się ściga**.

##### Przesłanka blokady (zweryfikowana) — i jedyna ścieżka, na której nie zachodzi

`SELECT … FOR UPDATE` serializuje tylko wtedy, gdy wiersz **nadal istnieje** po usunięciu.
Zweryfikowane: normalna ścieżka opuszczenia teamu robi **soft-delete**.
`teamService.RemoveTeamMember` (`app/teams/teams.go:248-251`) wykonuje:

```go
teamMember.Roles = ""
teamMember.DeleteAt = model.GetMillis()
if _, nErr := ts.store.UpdateMember(rctx, teamMember); nErr != nil { return nErr }
```

Wiersz `TeamMembers` **przetrwa z `DeleteAt > 0`** — czyli dokładnie tym, na czym dzisiejszy check w
`app/channel.go:1897` już się opiera. Zatem wiersz jest, jest co blokować, i projekt się trzyma. ✅

**Ale istnieje ścieżka hard-delete.** `SqlTeamStore.RemoveMembers` (`store/sqlstore/team_store.go:1262-1266`)
oraz `RemoveAllMembersByTeam` (`team_store.go:1285-1288`) wykonują prawdziwe `DELETE FROM TeamMembers`
(używane przy trwałym usuwaniu teamu/użytkownika, nie przy opuszczaniu). Na tych ścieżkach wiersz
znika, więc **nie zostaje nic, na czym równoległy `Grant` mógłby się zablokować** — a `FOR UPDATE` na
usuniętym wierszu nie czeka. Dla trwałego usuwania agregat musi więc serializować na wierszu trwałym:
zablokować rodzicielski wiersz **`Teams`** albo wziąć `pg_advisory_xact_lock` kluczowany na
`(teamID, userID)`. Sygnalizuję to, zamiast machnąć ręką, bo jest to **ta sama klasa błędu**, którą ta
sekcja koryguje: *blokada, która nie ma czego trzymać, nie jest blokadą.*

### 4.5 Cienkie API / cienkie App

Metody `*App` zachowują sygnatury (zgodność plugin API), ale stają się **parse → agregat → mapowanie
błędu**:

```go
func (a *App) AddChannelMember(rctx request.CTX, userID string, channel *model.Channel, opts ChannelMemberOpts) (*model.ChannelMember, *model.AppError) {
	// GrantMembership jest właścicielem całej sekcji krytycznej: otwiera JEDNĄ transakcję,
	// ładuje status w teamie FOR UPDATE, stosuje regułę i zapisuje — albo nie zapisuje nic.
	// Reguły nie da się ocenić na snapshocie, który równoległy LeaveTeam może unieważnić.
	saved, err := a.Srv().Store().Channel().GrantMembership(rctx, channel.Id, userID, membership.Grant)
	if err != nil {
		return nil, toAppError(err)   // ← nazwany błąd domenowy, fail-fast, nic nie zapisane
	}

	a.postAddMemberSideEffects(rctx, saved, opts)   // websockety, sidebar — PO zacommitowaniu faktu
	return saved, nil
}
```

Wewnątrz store'a, zgodnie z istniejącym wzorcem `channel_store.go:634-639`:

```go
func (s SqlChannelStore) GrantMembership(rctx request.CTX, channelID, userID string, rule membership.GrantFunc) (cm *model.ChannelMember, err error) {
	transaction, err := s.GetMaster().Begin()
	if err != nil { return nil, errors.Wrap(err, "begin_transaction") }
	defer finalizeTransactionX(transaction, &err)

	channel, err := s.getChannelT(transaction, channelID)
	if err != nil { return nil, err }

	// FOR UPDATE — punkt serializacji współdzielony z RevokeAllForTeamMember.
	teamStanding, err := s.getTeamMembershipForUpdateT(transaction, channel.TeamId, userID)
	if err != nil { return nil, err }

	granted, err := rule(channel, teamStanding, defaultRolesFor(channel))   // ← niezmiennik
	if err != nil { return nil, err }                                       // ← rollback, brak zapisu

	saved, err := s.saveMemberT(transaction, granted)
	if err != nil { return nil, err }
	return saved, transaction.Commit()
}
```

Reguła domenowa (`membership.Grant`) jest **wstrzykiwana jako funkcja**, dzięki czemu pakiet agregatu
ma zero zależności od store'a i pozostaje testowalny jednostkowo bez bazy — a store dostarcza granicę
transakcyjną, której reguła potrzebuje, by naprawdę obowiązywać.

`toAppError` mapuje błędy domenowe → HTTP raz, w jednym miejscu:

| Błąd domenowy | AppError id | HTTP |
|---|---|---|
| `ErrNotTeamMember` | `app.team.get_member.missing.app_error` | 404 (zachowuje dzisiejszy status — brak złamania kontraktu API) |
| `ErrTeamMembershipRevoked` | `api.channel.add_user.to.channel.failed.deleted.app_error` | 400 (zachowuje dzisiejszy status) |
| `ErrOrphanedMembership` | `app.channel.membership.orphaned.app_error` | 500 |

**Miejsce egzekwowania NIE przenosi się z klienta na serwer — ono już było na serwerze.** Przenosi się
z *rozproszonych czasowników serwisowych* do *jednego konstruktora + jednej transakcji*. To jest
poprawne sformułowanie tego refaktoru i chcę je mieć postawione dokładnie, a nie udramatyzowane.

---

## KROK 5 — Before/after, plan, testy

### 5.1 Before → after, dla każdego dzisiejszego miejsca reguły

| Miejsce (zweryfikowane) | Before | After |
|---|---|---|
| `app/channel.go:1887-1903` | `AddUserToChannel(..., skipTeamMemberIntegrityCheck bool)` — niezmiennik jest parametrem | Sygnatura zachowuje `bool` (plugin API), ale jest on **ignorowany i oznaczony deprecated**; check jest bezwarunkowy wewnątrz `membership.Grant`. Usunięty w ostatniej fazie. |
| `app/channel.go:1927-1935` | `ChannelMemberOpts.SkipTeamMemberIntegrityCheck` | Pole deprecated → usunięte. Optymalizacja synchronizacji zachowana przez **przekazanie `TeamMembership`**, nie przez pomijanie. |
| `app/channel.go:1966` | przekazuje `opts.SkipTeamMemberIntegrityCheck` | nie przekazuje nic; woła `membership.Grant`. |
| `app/syncables.go:69` | `SkipTeamMemberIntegrityCheck: true` | Przekazuje `TeamMembership`, które właśnie utworzył → warunek wstępny spełniony, **zero dodatkowych odczytów**, brak furtki. |
| `app/team.go:1363` `LeaveTeam` — pętla | Odczyt kanałów → N× `removeChannelMembership` → `RemoveTeamMember`, **bez transakcji** | Jedno wywołanie: `store.Channel().RevokeAllForTeamMember(teamID, userID)`. Atomowe. Pętla usunięta. |
| `app/team.go:1375-1377` (Dziura 3) | `ErrNotFound` przy odczycie kanałów → po cichu traktowane jako „brak kanałów", członek teamu i tak usunięty | Znika. Sprzątanie to zbiorczy `DELETE` w tej samej transakcji — „zero pasujących wierszy" i „odczyt się nie powiódł" nie da się już pomylić. |
| `app/channel.go:2886-2894` `removeChannelMembership` | 2 nieotransakcjonowane zapisy | Oba zapisy wciągnięte do transakcji agregatu. Helper przeżywa tylko dla ścieżki opuszczenia *pojedynczego kanału*. |
| `app/syncables.go:174-178` | `continue`-przy-błędzie przy eksmisji członków teamu group-constrained | **Poza zakresem — I-6, nie I-1.** Odnotowane, nie zaplanowane. Oznaczone, by nie zostało wzięte za naprawione. |
| `app/authorization.go:327-347` | Przyznaje odczyt wyłącznie z `ChannelMembers` | **Bez zmian — i staje się to poprawne, ale dopiero gdy wejdzie Faza 2.** Ufanie `ChannelMembers` jest słuszne *wtedy i tylko wtedy*, gdy osierocony wiersz nie może istnieć, co wymaga współdzielonej blokady wiersza (§4.4), a nie samego typu agregatu. *Zyskiem refaktoru jest to, że czyni istniejącą gorącą ścieżkę odczytu bezpieczną **bez dokładania do niej odczytu** — koszt płacony jest raz przy zapisie, nie przy każdym sprawdzeniu uprawnień.* |
| `store/sqlstore/integrity.go` | Brak checku `Teams → ChannelMembers` | **Dodany nowy check** — dzięki czemu każdy wcześniej istniejący sierota (sprzed refaktoru) staje się widoczny. |
| `model/channel_member.go` `IsValid()` | Tylko strukturalna | Bez zmian. Walidacja strukturalna zostaje strukturalna; reguły biznesowe idą do agregatu, a **nie** do `IsValid()`. |

### 5.2 Plan fazowy

Projekt ma **realną dyscyplinę testową** — pliki `_test.go` stoją obok praktycznie każdego pliku
`app/` i `store/`, jest ugruntowany harness `TestHelper` (`channels/api4/apitestlib.go`,
`channels/app/…/helper_test.go`) i runner Go (`go test ./...` / `make test`). Dlatego fazy 0–3 są
**naprawdę test-first**; jest runner, względem którego można być na czerwono.

| Faza | Praca | Test-first? |
|---|---|---|
| **0 — Ujawnij prawdę** | Dodaj check `Teams → ChannelMembers` do `store/sqlstore/integrity.go`. Wypuść osobno. **Czy naruszenie już istnieje w danych produkcyjnych?** Odpowiedz na to PRZED refaktorem — to wymiaruje migrację. | **Tak** — najpierw failujący test checku, z ręcznie zasianym sierotą. |
| **1 — Reguła, w izolacji** | Stwórz `app/membership/` (agregat, `Grant`, nazwane błędy). Zero zmian u wołających. Czyste testy jednostkowe, bez bazy. | **Tak.** Table-driven; tu żyje §5.3. |
| **2 — Atomowość + wzajemne wykluczanie** | `GrantMembership` + `RevokeAllForTeamMember` w store, na `GetMaster().Begin()` + `finalizeTransactionX` wg `channel_store.go:634-639`, **obie biorące `SELECT … FOR UPDATE` na tym samym wierszu `TeamMembers`**. Przepisz `LeaveTeam`, by je wołało. **Blokada wchodzi z tą fazą albo refaktor nie trzyma — patrz §4.4.** | **Tak** — test rollbacku *oraz* test współbieżności (§5.3 #11). |
| **3 — Przekieruj wołających** | `AddChannelMember` / `AddUserToChannel` delegują do `GrantMembership`. `bool` i pole opts są **ignorowane + deprecated**, jeszcze nie usunięte — to trzyma plugin API na zielono. Przepisz `syncables.go:69`, by przekazywał `TeamMembership`. | **Tak** — istniejące suity `app`/`api4` muszą zostać zielone; one są siatką regresji. |
| **4 — Backfill** | Migracja/job uzgadniający sieroty znalezione w Fazie 0. **Usunięcie nadania dostępu jest nieodwracalne — ta faza wymaga ludzkiej decyzji o polityce, nie wartości domyślnej.** | Test idempotencji. |
| **5 — Usuń furtkę** | Skasuj `skipTeamMemberIntegrityCheck` i `ChannelMemberOpts.SkipTeamMemberIntegrityCheck`. **Breaking change dla API pluginów — musi jechać z major release.** | Compile-time. |

Fazy 0–3 są niezależnie wypuszczalne i każda jest zyskiem netto, nawet jeśli łańcuch się na niej
zatrzyma. Faza 5 jest jedyną, która łamie kontrakt.

### 5.3 Przypadki testowe dla I-1

**Legalne (muszą przejść):**
1. Aktywny członek teamu dodany do publicznego kanału tego teamu → członkostwo nadane.
2. Aktywny członek teamu dodany do prywatnego kanału tego teamu → nadane.
3. Ścieżka group-sync (`syncables`): członek teamu utworzony, potem członkostwo kanałowe nadane w tym
   samym przepływie, **bez żadnego dodatkowego odczytu `Team().GetMember`** → nadane. *(Chroni powód
   wydajnościowy, dla którego istniała flaga skip — optymalizacja musi przeżyć.)*
4. DM/GM (`D`/`G`, bez `TeamId`) → skierowane do `GrantDirect`, nadane, I-1 nie stosowany.

**Nielegalne (muszą zwrócić nazwany błąd i NIC nie zapisać):**
5. Użytkownik, który nigdy nie był członkiem teamu → `ErrNotTeamMember`. **Asercja, że żaden wiersz
   `ChannelMembers` nie powstał** — nie tylko, że wrócił błąd.
6. Użytkownik z soft-usuniętym członkostwem w teamie (`DeleteAt > 0`) → `ErrTeamMembershipRevoked`.
7. Kanał z teamu A + członkostwo w teamie B → `ErrNotTeamMember` (niezgodność P1 — to przypadek,
   *którego dzisiejszy kod nie potrafi nawet wyrazić*, bo nigdy nie porównuje obu ID teamów).
8. **Test regresyjny usuniętej furtki:** żadna ścieżka kodu, przy żadnym wejściu, nie produkuje
   `ChannelMembership`, którego `teamID` ≠ `TeamId` kanału. Przy `Grant` jako jedynym konstruktorze
   pilnuje tego system typów; test dokumentuje intencję.

**Atomowość (Faza 2):**
9. `RevokeAllForTeamMember` z wymuszonym błędem na końcowym zapisie `TeamMembers` → **wszystkie**
   wiersze `ChannelMembers` nadal obecne (pełny rollback). Żadnego stanu częściowego.
10. `LeaveTeam` dla użytkownika w N kanałach → po commicie zero wierszy `ChannelMembers` w tym teamie,
    zero `ThreadMemberships`, członkostwo w teamie zniknęło. Jedna transakcja.
11. **Scenariusz TOCTOU z §3.1 jako jawny test współbieżności:** uruchom `LeaveTeam` i
    `AddChannelMember` dla tego samego użytkownika przeciw prawdziwej bazie, w obu kolejnościach,
    powtarzane pod obciążeniem. Asercja: stan końcowy **nigdy** nie jest „brak członkostwa w teamie +
    ocalałe członkostwo kanałowe" — tylko dwa legalne wyniki z §4.4 (dodanie odrzucone z
    `ErrTeamMembershipRevoked`, albo dodany wiersz zmieciony przez revoke).
    - Oczekiwany **czerwony na dzisiejszym kodzie** — i to czyni go wartym napisania.
    - Oczekiwany **nadal czerwony po samej Fazie 1** (typ agregatu nie zatrzyma wyścigu) i **zielony
      dopiero, gdy Faza 2 dowiezie współdzieloną blokadę `FOR UPDATE`.** Ten test jest wykonywalnym
      dowodem §4.4; jeśli przechodzi bez blokady, to znaczy, że **test naprawdę się nie ściga** i
      należy zwątpić w test, zanim zwątpi się w projekt.

### 5.4 Nowe nazwy „load-bearing" do rejestracji

Repo **nie ma `docs/reference/contract-surfaces.md`** (zweryfikowane — `/10x-init` nie był tu
uruchomiony, istnieje tylko `context/foundation/README.md`). Nie ma więc gdzie ich zarejestrować. Jeśli
rejestr powstanie, to są nazwy, które ten refaktor czyni nośnymi:

| Nazwa | Rodzaj | Dlaczego nośna |
|---|---|---|
| `membership.ChannelMembership` | Korzeń agregatu | Jedyny strażnik I-1. |
| `membership.TeamMembership` | Value object | Wejście warunku wstępnego; niesie rozróżnienie aktywny/odwołany. |
| `membership.Grant()` | Konstruktor | **Jedyny** sposób stworzenia członkostwa kanałowego. |
| `membership.ErrNotTeamMember` / `ErrTeamMembershipRevoked` / `ErrOrphanedMembership` | Błędy domenowe | Słownik fail-fast; mapowanie statusów API od nich zależy. |
| `ChannelStore.GrantMembership()` / `.RevokeAllForTeamMember()` | Kontrakt repozytorium | Granica atomowa — obie biorą **tę samą** blokadę wiersza `TeamMembers`. |
| `checkTeamsChannelMembersIntegrity()` | Kontrakt opsowy | Detektor, którego dziś nie ma. |
| ~~`skipTeamMemberIntegrityCheck`~~ / ~~`ChannelMemberOpts.SkipTeamMemberIntegrityCheck`~~ | **Wycofane** | Widoczne dla pluginów; usunięcie to breaking change (Faza 5). |

---

## Ograniczenia tego artefaktu

- **Nie istnieje żaden dokument wymagań** — I-1…I-6 są odtworzone z zachowania kodu. Teza „Team jest
  granicą dostępu" to **mój wniosek** z `README.md:3` + kodu autoryzacji, a nie zacytowana decyzja
  produktowa. *Jeśli faktyczną intencją zespołu jest, by opuszczenie teamu zachowywało dostęp do
  kanałów, cały ten plan rozwiązuje niewłaściwy problem — i to jest to jedno założenie, które człowiek
  musi potwierdzić przed Fazą 1.*
- **Scenariusz TOCTOU z §3.1 to tryb awarii dopuszczony przez projekt, a nie zaobserwowany bug.**
  Czytałem statycznie; nie uruchamiałem serwera, nie odtworzyłem wyścigu, nie zaglądałem do danych
  produkcyjnych. Braku blokady nie da się dowieść czytaniem. Faza 0 istnieje dokładnie po to, by
  zastąpić ten wniosek danymi, a test §5.3 #11 jest napisany tak, by wyścig albo się odtworzył, albo
  sfalsyfikował.
- **Szukałem *deterministycznej* ścieżki osierocenia i nie znalazłem jej.** Prześledziłem każdego
  produkcyjnego zapisującego do `TeamMembers` (`app/team.go:1415` przez `LeaveTeam`;
  `app/channel.go:2948` — eksmisja gościa z ostatniego kanału; `app/syncables.go:174` przez
  `RemoveUserFromTeam`) — wszystkie przechodzą przez sprzątanie kanałów. Dzisiejsze pęknięcie I-1 jest
  więc **latentne** (wyścig + furtka + Dziura 3), a nie stojącym bugiem, na który mogę wskazać palcem.
  **Wartością refaktoru jest uczynienie całej klasy usterek niereprezentowalną, a nie naprawa znanego
  incydentu** — i tak należy go sprzedawać, bez przehandlowania.
- **Teza o poziomie izolacji z §4.4 jest wyrozumowana, nie zmierzona.** Twierdzę, że `READ COMMITTED`
  nie serializuje tych dwóch ścieżek; nie uruchomiłem tego przeplecenia przeciw Postgresowi/MySQL, by
  to potwierdzić. Wynika to ze standardowej semantyki, ale to test z Fazy 2 ma to ustalić. Natomiast
  *przesłanka o soft-delete*, na której opiera się blokada, **jest** zweryfikowana
  (`app/teams/teams.go:248-251`) — podobnie jak wyjątek hard-delete (`team_store.go:1262-1266`).
- **Nie wykonano żadnego kodu** — nie uruchomiono testów ani builda. Każdy cytat został przeczytany;
  żaden nie został wykonany.
- **Powierzchnia plugin API** (`public/plugin/api.go:590`) wystawia `AddUserToChannel`; potwierdziłem
  sygnaturę, ale nie zaudytowałem konsumentów struktury opts po stronie pluginów. Promień rażenia Fazy
  5 jest więc **oszacowany, nie zmierzony**.
- Zakres to wyłącznie I-1. I-2…I-6 są zdiagnozowane, ale świadomie nie zaplanowane.
