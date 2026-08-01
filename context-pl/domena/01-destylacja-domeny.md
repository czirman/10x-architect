---
title: Destylacja domeny — rdzeń komunikacyjny Mattermost
created: 2026-07-11
type: domain-distillation
---

# Destylacja domeny — rdzeń komunikacyjny Mattermost

> Produkt tego dokumentu to **mapa domeny**, nie kod. Nazwy bytów, agregatów i reguł
> zostały **odkryte** z kodu i dokumentów źródłowych, nie założone z góry. Każde
> twierdzenie jest zakotwiczone cytatem `plik:linia`, który realnie zweryfikowano.

---

## KROK 0 — Kontekst projektu (odkrycie)

**Czym jest produkt.** Mattermost to „open core, self-hosted collaboration platform that
offers chat, workflow automation, voice calling, screen sharing, and AI integration”,
napisana w Go + React, działająca jako pojedynczy binarny plik na Linux + PostgreSQL
(`mattermost/README.md:3`). Wydania miesięczne, licencja MIT (`README.md:3`).

**Dokumenty wymagań — OGRANICZENIE.** W repozytorium **brak PRD / vision / tech-stack.md**
(`find` po `*prd*`, `*vision*`, `*tech-stack*` nie zwrócił żadnego dokumentu produktowego).
Jedyne narracje źródłowe to `README.md` (marketingowa) i `CHANGELOG.md` (pusty stub
odsyłający do docs online, `CHANGELOG.md:1-4`). W konsekwencji **cel produktu i success
criteria destyluję z kodu warstwy kontraktowej** (`server/public/model/`) oraz z wcześniej
wykonanej mapy repo (`context/map/repo-map.md`), a **nie** z dokumentu wymagań. To najmocniejsze
ograniczenie całego artefaktu: reguły biznesowe rekonstruuję z zachowania kodu, więc
„niezmiennik deklarowany” = to, co kod wymusza dziś, a nie to, co ktoś spisał jako intencję.

**Stack i gdzie żyje logika biznesowa** (zweryfikowane układem katalogów):

| Warstwa | Ścieżka | Rola |
|---|---|---|
| Kontrakt / model domenowy | `server/public/model/*.go` (284 pliki) | Struktury bytów + `IsValid()` — **walidacja strukturalna**, nie niezmienniki biznesowe |
| Logika aplikacyjna (serwis) | `server/channels/app/*.go` | **Tu żyją niezmienniki międzybytowe** (członkostwo, wątki, grupy) |
| Persystencja | `server/channels/store/` | Kontrakt dostępu do danych, ograniczenia unikalności DM/GM |
| API | `server/channels/api4/` | Wejście HTTP, autoryzacja |
| UI | `webapp/` (TypeScript, React) | Konsola admina, kompozytor postów |

**Zakres destylacji (świadome zawężenie).** 284 plików modelu to nie jest cel destylacji.
Idę w głąb wyłącznie na **rdzeniu komunikacyjnym**: `Team → Channel → ChannelMember → Post`
+ `User` + `Role/Permission`. To jednoznacznie rdzeń produktu („chat”, `README.md:3`) i miejsce,
gdzie żyją niezmienniki. Powierzchnię konfiguracji (`config.go`) — hub #1 wg mapy git
(`context/map/repo-map.md:106,146`) — klasyfikuję jako **subdomenę wspierającą/generyczną** i tylko
mostkuję do wcześniejszej pracy `context/changes/config-surface/`. Zawężenie **nie jest** zakładaniem
nazw — byty nadal odkrywam z kodu; deklaruję tylko, które poddrzewo zbadałem w głąb.

---

## KROK 1 — Ubiquitous Language (rdzeń komunikacyjny)

Terminy wyciągnięte z kodu + README. Definicja / cytat źródłowy / gdzie żyje w kodzie.

| Termin | Definicja (odkryta) | Źródło pojęcia | Życie w kodzie |
|---|---|---|---|
| **Team** | Przestrzeń nadrzędna grupująca kanały i użytkowników; typ `O` (open) lub `I` (invite). | „collaboration platform” `README.md:3` | `type Team struct` `model/team.go:26`; typy `TeamOpen/TeamInvite` `team.go:15-16`; `IsValid` `team.go:129` |
| **Channel** | Miejsce rozmowy wewnątrz Team; sześć typów. | „chat” `README.md:3` | `type Channel struct` `model/channel.go:83`; `type ChannelType` `channel.go:25` |
| **ChannelType** | `O` open, `P` private, `D` direct (DM), `G` group (GM), `BO`/`BP` board. | — | `model/channel.go:28-33` |
| **Direct Message (DM)** | Kanał typu `D` między dokładnie dwoma użytkownikami; nazwa = `userA__userB`. | — | `model/channel.go:30`; kodowanie nazwy `channel.go:348-351`; tworzenie `app/channel.go:466` |
| **Group Message (GM)** | Kanał typu `G` dla 3–8 użytkowników. | — | `model/channel.go:31`; limity `ChannelGroupMinUsers=3`/`MaxUsers=8` `channel.go:37-38` |
| **ChannelMember** | Członkostwo użytkownika w kanale; role scheme + NotifyProps. | — | `type ChannelMember struct` `model/channel_member.go:54`; `IsValid` `channel_member.go:129` |
| **Scheme role (Guest/User/Admin)** | Rola członka w schemacie uprawnień kanału. | — | `SchemeGuest/SchemeUser/SchemeAdmin` `model/channel_member.go:66-68` |
| **Post** | Wiadomość w kanale; opcjonalnie odpowiedź w wątku (`RootId`). | „chat” `README.md:3` | `type Post struct` `model/post.go:127`; `RootId` `post.go:136`; `IsValid` `post.go:479` |
| **Thread / RootId** | Wątek: post-odpowiedź wskazuje `RootId` na post-korzeń; max 1 poziom. | — | `post.go:136`; egzekwowane w `app/post.go:299-306` |
| **User** | Konto użytkownika; może być botem lub użytkownikiem zdalnym (`IsRemote`). | — | `type User struct` `model/user.go` (1160 linii); `IsBot`, `IsRemote()` używane `app/channel.go:471`, `app/post.go:246` |
| **Role / Permission / Scheme** | Model RBAC: role posiadają uprawnienia; schematy mapują role na poziomy (team/channel). | „workflow automation” `README.md:3` | `model/role.go` (1267 l.); `model/permission.go` (2772 l.); migracje `app/permissions_migrations.go` |
| **GroupConstrained** | Flaga: członkostwo kanału/teamu ma być zsynchronizowane z powiązaną grupą (LDAP/SAML). | — | pole `channel.go:100`; `IsGroupConstrained()` egzekwowane `app/channel.go:1802` |
| **SharedChannel** | Kanał współdzielony między zdalnymi instancjami (federacja). | — | `app/shared_channel.go`; rekord tworzony `app/channel.go:513` |
| **Configuration** | Kontrakt konfiguracji serwera — subdomena wspierająca (patrz KROK 2). | mapa: hub #1 `repo-map.md:146` | `model/config.go` (BRAK niezmiennika biznesowego rdzenia) |

---

## KROK 2 — Klasyfikacja subdomen (Core / Supporting / Generic)

Rdzeń = to, co stanowi sens i przewagę produktu („chat” + kontrola dostępu do rozmów).
Uzasadnienie odwołuje się do wizji z `README.md:3` (brak formalnych success criteria — patrz KROK 0).

| Obszar / pojęcie | Kategoria | Uzasadnienie |
|---|---|---|
| **Channel + ChannelMember** (członkostwo, typy, wątki) | **Core** | Istota „chat”; to na tym stoi cała wartość produktu. Tu żyją najostrzejsze niezmienniki międzybytowe (`app/channel.go:1888`, `app/post.go:299`). |
| **Post / Thread** | **Core** | Nośnik samej rozmowy. Integralność wątku = poprawność rdzeniowego doświadczenia. |
| **Team** | **Core** | Granica organizacyjna kanałów i członkostwa; determinuje niezmiennik członkostwa (`app/channel.go:1888`). |
| **Role / Permission / Scheme (RBAC)** | **Core** | Kontrola dostępu do rozmów to część przewagi „self-hosted / enterprise” (`README.md:3`, DevSecOps use-case `README.md:11`). |
| **GroupConstrained / synchronizacja z grupami (LDAP/SAML)** | **Supporting** | Wspiera enterprise, ale to integracja z tożsamością, nie sedno czatu. |
| **SharedChannel / federacja** | **Supporting** | Rozszerza zasięg rdzenia; nie jest jego istotą. |
| **Notifications / NotifyProps** | **Supporting** | Wspiera rozmowę, ale wymienne, nie różnicujące. |
| **Configuration surface** (`config.go`, `admin_definition.tsx`) | **Generic / Supporting** | Hub o najwyższym churnie (`repo-map.md:106,146`), ale to zarządzanie konfiguracją — problem rozwiązany generycznie, nie przewaga domenowa. **Mostek do** `context/changes/config-surface/`: tamta praca badała *przepływ* konfiguracji; z perspektywy DDD to warstwa techniczna otaczająca rdzeń. |
| **Audit / Compliance** | **Generic** | Wymóg regulacyjny, generyczny wzorzec. `model/audit.go`, `compliance.go`. |
| **Cluster / Cloud / analytics** | **Generic** | Infrastruktura operacyjna. |

**Wniosek KROK 2:** rdzeń to `Team → Channel → ChannelMember → Post` z nakładką RBAC.
Powierzchnia konfiguracji, mimo najwyższej aktywności git, **nie jest rdzeniem domeny** —
jest generyczną warstwą techniczną. To rozjazd „aktywność ≠ znaczenie domenowe”
wprost odnotowany w mapie (`repo-map.md:146`).

---

## KROK 3 — Kandydaci na agregaty i ich niezmienniki

> Kluczowe rozróżnienie (potwierdzone w kodzie): `IsValid()` w `model/` to **deklaracja
> strukturalna** (format ID, przynależność do enuma, długości pól) — **nie** egzekwowanie
> niezmiennika biznesowego. Niezmienniki międzybytowe żyją w `server/channels/app/`.
> Status poniżej: **egzekwuje** / **deklaruje** / **ignoruje**.

### Kandydat A — Channel jako korzeń agregatu członkostwa (Channel + ChannelMembers)

| Niezmiennik | Cytat źródłowy | Status egzekwowania |
|---|---|---|
| Członkiem kanału może być tylko **aktywny członek Teamu** kanału | `app/channel.go:1888-1903` (`GetMember(channel.TeamId, user.Id)`, `teamMember.DeleteAt > 0` → błąd) | **Egzekwuje w warstwie serwisu, opt-out** — bramkowane parametrem `skipTeamMemberIntegrityCheck bool` (`channel.go:1887`). Weryfikacja użycia: jedyny `skip=true` to `app/syncables.go:69` (grupowa synchronizacja — członek Teamu właśnie dodany, kontrola redundantna); bezpośredni wywołujący przekazują `false` (`channel.go:2653`, `team.go:721`). Zatem **nie jest to obchodzona furtka, lecz optymalizacja redundancji** — ale niezmiennik nadal nie jest gwarantowany *z konstrukcji* agregatu, tylko dyscypliną wywołań. |
| Kanał **Direct** ma **dokładnie 2** członków | brak deklaracji w `model.Channel.IsValid`; wymuszone **sygnaturą** `createDirectChannelWithUser(user, otherUser)` `app/channel.go:466` + kodowaniem nazwy `userA__userB` `channel.go:348-351` | **Ignoruje na poziomie agregatu** — „2” wynika z liczby argumentów funkcji i store, nie z niezmiennika bytu. |
| Kanał **Group** ma **3–8** członków | `app/channel.go:561` (`len(userIDs) > ChannelGroupMaxUsers \|\| < ChannelGroupMinUsers`) | **Egzekwuje** — ale tylko przy tworzeniu, w metodzie serwisu, nie w bycie. |
| Kanał `group_constrained` → członkostwo = powiązana grupa | flaga `channel.go:100`; egzekwowanie rozproszone: `app/channel.go:1802` (dodawanie), `app/channel.go:2915` (usuwanie), `app/access_control.go:2098`, sync `app/group.go:409` | **Deklaruje flagę, egzekwuje rozproszony proces** — spójność zależy od joba synchronizującego, nie od agregatu. |

### Kandydat B — Post jako korzeń agregatu wątku

| Niezmiennik | Cytat źródłowy | Status egzekwowania |
|---|---|---|
| Post-korzeń wątku musi być w **tym samym kanale** co odpowiedź | `app/post.go:299-301` (`!parentPostList.IsChannelId(post.ChannelId)` → błąd) | **Egzekwuje w warstwie app** — `model.Post.IsValid` sprawdza tylko format `RootId` (`post.go:500`). |
| Wątek jest **płaski** (max 1 poziom — nie można odpowiadać na odpowiedź) | `app/post.go:304-306` (`if rootPost.RootId != ""` → błąd) | **Egzekwuje w warstwie app** — całkowicie nieznane modelowi. |

### Kandydat C — Team jako korzeń agregatu członkostwa organizacyjnego

| Niezmiennik | Cytat źródłowy | Status egzekwowania |
|---|---|---|
| Team `group_constrained` → członkostwo = powiązana grupa | `app/team.go:686,762`; `app/access_control.go:2123` | **Egzekwuje rozproszony proces**, analogicznie do kanału. |
| Typ Teamu ∈ {`O`,`I`} | `model/team.go:174` | **Deklaruje** (strukturalnie, w `IsValid`). |

---

## KROK 4 — Rozjazdy MODEL vs KOD (najcenniejsza część)

> „Model” = to, co byt deklaruje o sobie w `server/public/model/*.go`.
> „Kod” = gdzie reguła jest naprawdę (lub wcale) egzekwowana.

| # | Reguła domenowa | MODEL mówi | KOD robi | Dowód |
|---|---|---|---|---|
| 1 | Członek kanału musi być członkiem Teamu | `Channel.IsValid` / `ChannelMember.IsValid` **nic** o Teamie — tylko format ID i NotifyProps | Kontrola w jednej metodzie serwisu, opt-out flagą `skipTeamMemberIntegrityCheck` (jedyny `skip=true`: sync w `syncables.go:69`, redundantnie) | `model/channel_member.go:129-147` (brak) vs `app/channel.go:1887-1903` + `syncables.go:69` |
| 2 | Odpowiedź i korzeń wątku w tym samym kanale | `Post.IsValid` sprawdza tylko `IsValidId(RootId)` | Serwis dociąga korzeń i porównuje kanał | `model/post.go:500` vs `app/post.go:299-301` |
| 3 | Wątek płaski (bez odpowiedzi na odpowiedź) | **milczy** | Serwis odrzuca `rootPost.RootId != ""` | `model/post.go` (brak) vs `app/post.go:304-306` |
| 4 | DM = dokładnie 2 członków | **milczy** — `IsValid` nie liczy członków (blok `channel.go:348-351` odrzuca *nie-DM* kanały o nazwie kolidującej z wzorcem `__`, nie zlicza członków DM) | Wymuszone sygnaturą 2-argumentową `createDirectChannelWithUser(user, otherUser)` + store | `model/channel.go:311-386` (brak zliczania) vs `app/channel.go:466` |
| 5 | GM = 3–8 członków | **milczy** (stałe `channel.go:37-38` istnieją, ale `IsValid` ich nie używa) | Sprawdzenie tylko przy `createGroupChannel` | `model/channel.go:311` (brak) vs `app/channel.go:561` |
| 6 | `group_constrained` → członkostwo == grupa | Deklaruje **tylko** flagę + że jest ona sensowna dla O/P (`channel.go:379-382`) | Spójność utrzymuje rozproszony proces + job sync (≥5 miejsc) | `model/channel.go:380` vs `app/channel.go:1802,2915`, `access_control.go:2098`, `group.go:409` |

**Sedno rozjazdu:** wiedza domenowa o tym, *czym jest poprawny kanał/wątek*, istnieje i jest
bogata — ale mieszka w **czasownikach serwisu** (`app/`), nie w **rzeczownikach domeny**
(`model/`). Byty są anemiczne: `IsValid()` pilnuje kształtu pola, a nie prawdy biznesowej.
Najgroźniejszy przypadek to #1 — niezmiennik rdzeniowy, który można *poprosić* o pominięcie
przekazując `true`.

---

## KROK 5 — Ranking refaktoru (wartość × ryzyko)

Wartość = jak rdzeniowy jest niezmiennik. Ryzyko = jak słabo jest dziś egzekwowany
(łatwość naruszenia).

| Ranga | Agregat / niezmiennik | Wartość | Ryzyko | Uzasadnienie |
|---|---|---|---|---|
| **#1** | **Channel + ChannelMembers — „członek kanału musi być aktywnym członkiem Teamu”** | **Bardzo wysoka** (rdzeń: kanał należy do Teamu; naruszenie = dostęp do rozmów spoza uprawnień) | **Wysokie (strukturalne)** | Niezmiennik **nie jest gwarantowany z konstrukcji** — egzekwowany w jednej metodzie serwisu, opt-out flagą (`app/channel.go:1887`); agregat go nie pilnuje. Faktyczne ryzyko „obejścia” niskie (jedyny `skip=true` jest redundantny — `syncables.go:69`), ale spójność zależy od dyscypliny wywołań, nie od bytu. |
| #2 | Post/Thread — same-channel + płaskość wątku | Wysoka (integralność rozmowy) | Średnie | Egzekwowane, ale rozproszone w `app/post.go` i całkowicie nieobecne w bycie `Post`; łatwo obejść nową ścieżką tworzenia postu. |
| #3 | GroupConstrained — członkostwo == grupa | Średnia (enterprise, subdomena wspierająca) | Wysokie | Spójność zależna od ≥5 rozproszonych miejsc + jobu sync; brak pojedynczego strażnika. |
| #4 | GM 3–8 / DM = 2 | Średnia | Niskie | Wymuszone strukturalnie (sygnatura/store); trudniej naruszyć przypadkiem. |

### #1 do refaktoru: agregat **Channel + ChannelMembers** wokół niezmiennika członkostwa-Teamu

**Dlaczego #1:** to jedyny kandydat z kombinacją *najwyższa wartość* (rdzeniowa reguła
bezpieczeństwa dostępu) × *najwyższe ryzyko* (egzekwowanie w jednym miejscu, celowo
wyłączalne booleanem, bez wsparcia agregatu/store). Refaktor polegałby na przeniesieniu
niezmiennika z wyłączalnej metody serwisu do korzenia agregatu Channel (metoda typu
`AddMember` odmawiająca dodania nie-członka Teamu), tak aby niezmiennik był prawdziwy
**z konstrukcji**, a nie utrzymywany dyscypliną wywołań serwisu.
To ustawia bezpośrednio następny krok łańcucha (`m4l5-2-invariant-aggregate-refactor`).

---

## Ograniczenia artefaktu

- **Brak dokumentu wymagań** (PRD/vision) — reguły biznesowe zrekonstruowane z zachowania
  kodu warstwy `app/`, nie z intencji spisanej przez zespół (KROK 0). „Niezmiennik
  deklarowany” = to, co kod wymusza dziś.
- **Zakres zawężony** do rdzenia komunikacyjnego; 278 pozostałych plików `model/` (cloud,
  cluster, compliance, oauth, ...) nie analizowano w głąb.
- Cytaty `plik:linia` odnoszą się do stanu repo `mattermost/` w tym checkout (branch
  `module-4-lesson-5`); numery linii mogą się przesunąć po edycjach.
- **Warstwa store nie zbadana w głąb** — nie zweryfikowano, czy `store/sqlstore/` gwarantuje
  niezmiennik członkostwa (np. przez FK). Teza DDD („byt sam go nie pilnuje”) obowiązuje
  niezależnie, ale ewentualna gwarancja na poziomie bazy pozostaje niezweryfikowana.
- Analiza statyczna (grep + odczyt) — nie uruchamiano kodu ani testów; nie badano
  zachowań runtime, wydajności ani pokrycia testami niezmienników.
