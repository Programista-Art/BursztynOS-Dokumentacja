# Audyt podsystemu Ustawienia ekranu (Bursztyn OS)

**Stan dokumentu:** aktualizacja 2026-09-19  
**Pierwotny audyt:** 2026-09-06  
**Zakres:** tryby ekranu, HAL, Bochs VBE, skala UI, PMM/VMM graphics buffers, kompozytor, popupy/dialogi, generacja ekranu, rollback rozdzielczości i trwałość konfiguracji  
**Status manualny:** nadal brak potwierdzonej pełnej weryfikacji na realnym sprzęcie  
**Status bieżący:** historyczny audyt ekranu zakończył się `31128/0`, ale późniejsze zmiany GUI/popupów ujawniły nową regresję cyklu życia buforów PMM/VMM, dlatego podsystemu **nie należy obecnie opisywać jako w pełni stabilnego pamięciowo**

---

## 1. Najważniejsza zmiana względem audytu z 2026-09-06

Audyt z 2026-09-06 raportował:

```text
PASS assertions = 31128
FAIL assertions = 0
```

oraz dla trzech pełnych cykli przełączania trybów:

```text
PMM delta  = 0
HEAP delta = 0
BUF delta  = 0
```

Był to poprawny wynik dla tamtej rewizji.

Od tego czasu rozbudowano jednak między innymi:

- BursztynUI,
- dialogi,
- popupy top-level,
- toast notifications,
- menu i nested popup stack,
- generację ekranu,
- damage starego i nowego obszaru popupu,
- zachowanie popupów przy wysokich rozdzielczościach.

W bieżącym cyklu rozwoju zaobserwowano następnie:

```text
[EKRAN] BLAD puli PMM/VMM
```

Dlatego starszego wyniku `31128/0` oraz `delta=0` nie wolno traktować jako dowodu braku wycieku w aktualnym `HEAD`.

---

# 2. Aktualna klasyfikacja statusu

Najbardziej rzetelny status:

```text
dynamiczne tryby             = zaimplementowane
skalowanie UI                = zaimplementowane
rollback 15 s                = zaimplementowany
persist ekran.cfg            = zaimplementowany
BWS62 monitor/work-area      = zaimplementowany
dedykowana pula grafiki      = zaimplementowana
popup display generation     = zaimplementowana
wysokie rozdzielczości       = aktywnie stabilizowane
lifetime popup/backing       = wymaga dalszego audytu
PMM/VMM leak-free guarantee  = obecnie NIEPOTWIERDZONA
real hardware                = NOT MANUALLY VERIFIED
```

Nie należy obecnie kończyć dokumentu zdaniem:

> „podsystem jest w pełni poprawny, stabilny pamięciowo i sprzętowo”.

Poprawniejsze podsumowanie:

> **Podsystem Ustawienia ekranu jest funkcjonalny i posiada rozbudowaną ścieżkę testową, ale po późniejszych zmianach popupów i generacji ekranu wymaga ponownej weryfikacji zarządzania dużymi zasobami PMM/VMM, szczególnie przy rozdzielczościach 1920×1080 i wyższych.**

---

# 3. Backend Bochs VBE i lista trybów

W audytowanym środowisku QEMU/Bochs sterownik posiadał 10 kandydatów:

```text
800x600
1024x768
1280x720
1280x800
1366x768
1360x768
1600x900
1920x1080
2560x1440
3840x2160
```

Udostępniano 9 poprawnych trybów:

```text
1280x720x32
800x600x32
1024x768x32
1280x800x32
1360x768x32
1600x900x32
1920x1080x32
2560x1440x32
3840x2160x32
```

Tryb:

```text
1366x768
```

jest odrzucany przez wymaganie wyrównania szerokości do 8 pikseli.

Warunek:

```cpp
!(t.szerokosc & 7)
```

dla `1366` nie jest spełniony.

Bieżąca aplikacja `Ustawienia ekranu` nadal powinna pobierać listę trybów **dynamicznie z HAL**, zamiast utrzymywać niezależną statyczną listę w Ring 3.

---

# 4. 4K i VRAM

Dla:

```text
3840 × 2160 × 4
```

rozmiar pojedynczego XRGB8888 framebuffer-sized buffer wynosi:

```text
33 177 600 B
≈ 31.64 MiB
```

Historyczny audyt wykrywał w QEMU:

```text
67 108 864 B
= 64 MiB VRAM
```

więc sam sprzętowy LFB 4K mieścił się w VRAM.

Należy jednak rozróżnić:

```text
VRAM urządzenia
```

od:

```text
RAM systemowy używany przez backbuffer,
tapetę, powierzchnie okien i popupy.
```

Przy 4K problemem nie jest tylko VRAM.

Jednoczesne powierzchnie systemowe mogą wymagać wielu dziesiątek lub setek MiB RAM zależnie od liczby okien i popupów.

---

# 5. Dedykowana pula PMM/VMM grafiki

Duże bufory nie są alokowane ze zwykłej sterty jądra.

Wykorzystywana jest dedykowana pula:

```text
PMM/VMM graphics buffers
```

obejmująca m.in.:

- backbuffer,
- przygotowaną tapetę,
- powierzchnie okien,
- framebuffer-dependent scratch surfaces,
- nowsze backing surfaces popupów.

Historyczny audyt dokumentował przestrzeń:

```text
BUFORY_GRAFIKI_BAZA = 0xffff900000000000
```

oddzieloną od HHDM.

Adres ten jest detalem implementacji i może się zmienić. Ważniejszym kontraktem jest to, że:

> **duże zasoby graficzne nie mogą zużywać zwykłej małej sterty kernela.**

---

# 6. Poprawny lifecycle bufora grafiki

Dla każdego bufora należy zachować symetrię:

```text
reserve VA
→ allocate PMM frames
→ map PTE
→ publish resource
...
→ detach resource
→ clear PTE
→ invalidate TLB
→ return PMM frame
→ free virtual slot/range
→ clear metadata
```

Samo:

```cpp
pointer = nullptr;
```

nie oznacza zwolnienia zasobu.

Przy odmapowaniu należy dopilnować, aby:

1. PTE zostało wyczyszczone;
2. TLB został unieważniony;
3. fizyczna ramka wróciła do PMM;
4. wirtualny slot/range został oznaczony jako wolny;
5. metadane aktywnego bufora zostały usunięte.

---

# 7. Rollback częściowej alokacji

Każda funkcja przygotowująca duży framebuffer-sized resource musi być odporna na częściowy błąd.

Przykład:

```text
zarezerwowano VA
↓
zaalokowano 1500 ramek
↓
ramka 1501 FAIL
```

Poprawny rollback:

```text
odmapuj wszystkie 1500 stron
zwróć wszystkie 1500 ramek
zwolnij VA range
wyzeruj metadata
zwróć FAIL
```

Niepełny rollback jest jednym z głównych podejrzanych w diagnostyce:

```text
[EKRAN] BLAD puli PMM/VMM
```

---

# 8. Transakcyjna zmiana rozdzielczości

Aktualna architektura powinna zachowywać model:

```text
1. validate target mode
2. prepare new graphics state
3. prepare wallpaper
4. prepare dependent resources
5. krótki display lock
6. hardware mode switch
7. atomic graphics-state swap
8. compositor geometry update
9. increment/update display generation
10. compose coherent frame
11. unlock
12. release old resources
```

Najważniejsza reguła:

> **dużych alokacji, zerowania ani skalowania tapety nie wykonuje się pod `zablokuj_ekran()`.**

---

# 9. Display lock

Pod krótką blokadą mogą znajdować się operacje wymagające atomowej spójności stanu:

```text
bochs_ustaw_tryb
grafika_wymien_stan
skladacz_zastosuj_ekran
aktualizacja generacji
złożenie pierwszej spójnej klatki
```

Poza blokadą powinny znajdować się:

```text
VmmMapujBuforGrafiki
grafika_przygotuj_ekran
tapety_przygotuj_ekran
grafika_zwolnij_stan
VmmOdmapujBuforGrafiki
psf_flush_cache
```

To pozostaje ważne również przed przyszłym SMP.

---

# 10. Timeout i rollback

Zmiana fizycznego trybu uruchamia:

```text
15 s
```

okno potwierdzenia.

Jeżeli użytkownik nie potwierdzi nowego trybu:

```text
ekran_obsluz_timeout()
```

powinien przywrócić:

- poprzedni tryb sprzętowy,
- poprzedni graphics state,
- geometrię kompozytora,
- prawidłową generację ekranu,
- poprawny routing popup/focus/capture.

Plik:

```text
/ustawienia/ekran.cfg
```

nie może zostać nadpisany niepotwierdzoną konfiguracją.

---

# 11. Trwałość `/ustawienia/ekran.cfg`

Po jawnej akcji:

```text
EKRAN_ZACHOWAJ
```

konfiguracja jest utrwalana w:

```text
/ustawienia/ekran.cfg
```

Ścieżka powinna obejmować:

```text
write
→ psf_flush_cache()
→ read-back verification
```

Zmiana tymczasowa i zmiana zatwierdzona to dwa różne stany.

---

# 12. Startup config

Przy starcie:

- prawidłowa skala UI może być zastosowana niezależnie od możliwości runtime mode switch;
- backend GOP/VESA może zachować tryb bootloadera;
- Bochs VBE może dynamicznie ustawić obsługiwany zapisany tryb;
- uszkodzona konfiguracja powinna prowadzić do bezpiecznego fallbacku.

Nie należy traktować rozdzielczości i skali jako jednej wartości.

---

# 13. Skale UI

Aktualny zakres:

```text
100%
125%
150%
175%
200%
```

BursztynUI obejmuje dziś znacznie większy zestaw elementów niż w pierwszym audycie.

Skalowanie dotyczy między innymi:

- paska tytułowego,
- przycisków systemowych,
- tekstu UTF-8,
- Przycisku,
- PoleTekstowe,
- List,
- Tabel,
- Drzewa,
- WidokIkon,
- WidokMiniatur,
- TabelaWirtualna,
- ListaWirtualna,
- EdytorFormatowany,
- PoleWyszukiwania,
- SelektorKoloru,
- OknoDialogowe,
- DialogPliku,
- DialogKoloru,
- Menu,
- Tooltips,
- Powiadomienia,
- Paska Zadań,
- Menu Start.

Nie należy z samego istnienia skalowania wyciągać wniosku, że każda nowa kontrolka została już poprawnie zweryfikowana na każdej skali.

---

# 14. BWS 62

BWS 62 nadal służy do uzyskania informacji o monitorze/obszarze roboczym.

Aktualny model jest praktycznie single-monitor, ale struktura ABI jest zaprojektowana rozszerzalnie.

Logicznie zwracane są m.in.:

```text
szer
wys
work_x
work_y
work_szer
work_wys
scale_x1000
```

Historyczny model work area:

```text
work_x    = 0
work_y    = 0
work_szer = szer
work_wys  = wys - ui_skaluj(40, skala)
```

pozostaje właściwy dla obecnego Paska Zadań, dopóki bieżący kod nie definiuje innej geometrii.

---

# 15. BWS 63 i publikacja work area

BWS 63 umożliwia zaufanemu komponentowi systemowemu publikowanie globalnego work area.

Zwykła aplikacja nie powinna mieć możliwości dowolnego przesunięcia globalnego obszaru roboczego.

Operacja powinna wymagać odpowiedniego prawa systemowego, np.:

```text
PRAWO_SYSTEM_CONFIG
```

To oddziela:

```text
query monitora
```

od:

```text
zmiany globalnej polityki pulpitu
```

---

# 16. Mysz i wskaźnik po zmianie trybu

Po zmianie fizycznego ekranu współrzędne kursora muszą zostać ograniczone do nowych granic:

```text
0 <= x < width
0 <= y < height
```

Należy testować szczególnie:

```text
2560x1440 → 1280x720
```

gdy poprzednia pozycja kursora leży poza nowym ekranem.

Input routing nie powinien przekazać pierwszego eventu po zmianie do niepoprawnego hit-test region.

---

# 17. Okna po zmianie rozdzielczości

Okna niemaksymalizowane powinny zostać dopasowane do nowego work area.

Okna zmaksymalizowane powinny zostać przeliczone na nowy:

```text
{work_x, work_y, work_szer, work_wys}
```

Należy zachować przy tym rozdzielenie:

```text
window geometry
focus
capture
popup ownership
modal state
```

Sama zmiana geometrii nie może pozostawić starego capture lub niewidzialnego modala.

---

# 18. Stary model popupów

Audyt z 2026-09-06 mówił:

```text
otwarte popupy aplikacji są zamykane podczas resetu ekranu
```

Od tego czasu architektura została rozszerzona.

Popupy nie są już wyłącznie elementem przycinanym do powierzchni właściciela.

Nowszy model używa:

```text
detached top-level popup surfaces
```

publikowanych do kompozytora.

Dotyczy to m.in.:

- menu,
- DialogPliku,
- DialogKoloru,
- tooltipów,
- toastów,
- nested popup stack.

---

# 19. Damage popupów

Przy przesunięciu/zamknięciu popupu compositor musi uwzględnić:

```text
old bounds
+
new bounds
+
shadow extent
```

Brak starego damage prowadzi do pozostawiania śladów.

Ten mechanizm nie oznacza jeszcze pełnego tile-based damage compositor dla całego systemu.

Aktualny stan najlepiej opisać jako:

> **globalny/klatkowy compositor rozszerzony o lokalne damage informacje w wybranych ścieżkach popupów i geometrii.**

---

# 20. `generacja_ekranu`

Każda powierzchnia zależna od geometrii fizycznego ekranu powinna być związana z aktualną generacją.

Dla popupów:

```text
popup.generacja_ekranu
```

musi odpowiadać:

```text
aktualna_generacja_ekranu
```

Jeżeli generacje są różne:

```text
nie prezentuj starej powierzchni
```

i wykonaj:

```text
release
```

albo:

```text
rebuild
```

dla nowej generacji.

---

# 21. Krytyczna reguła: invalidation musi zwalniać zasób

Sam warunek:

```text
generation mismatch
→ ignore popup
```

jest niewystarczający.

Jeżeli backing surface pozostanie zmapowana, otrzymujemy:

```text
niewidoczny resource leak
```

Jeżeli popup był modalny, trzeba również zwolnić lub przebudować:

```text
capture
focus
modal ownership
```

W przeciwnym razie możliwy jest stan:

```text
popup niewidoczny
+
aplikacja nadal zablokowana modalnie
```

---

# 22. Aktualna hipoteza regresji PMM/VMM

Po dodaniu generacji ekranu główne hipotezy należy sprawdzać w tej kolejności:

```text
1. stale popup surface odłączona logicznie, ale nie odmapowana
2. combined popup surface reallokowana bez zwolnienia starej
3. partial VMM map failure bez kompletnego rollbacku
4. rollback zmiany trybu pozostawia nowy bufor
5. stary display-state nie jest zwalniany po udanym commit
6. slot VA puli nie wraca do freelisty
7. licznik/accounting bufora nie jest dekrementowany
8. dopiero na końcu: pula rzeczywiście za mała
```

Nie należy zaczynać od zwiększenia limitu puli.

---

# 23. Rozmiary pomocne w diagnostyce

Dla XRGB8888 i typowego stride:

```text
1920x1080:
7680 * 1080
= 8 294 400 B

2560x1440:
10240 * 1440
= 14 745 600 B

3840x2160:
15360 * 2160
= 33 177 600 B
```

Jeżeli w trakcie zmiany trybu jednocześnie istnieją:

```text
old backbuffer
new backbuffer
old wallpaper
new wallpaper
window surfaces
popup surfaces
combined popup
```

rzeczywisty peak jest wielokrotnie większy od jednego framebufferu.

Dlatego każdy wzrost limitu puli musi być poprzedzony wyliczeniem **maksymalnego legalnego stanu jednoczesnego**.

---

# 24. Historia testu `delta=0`

Historyczny test:

```text
1280x800
→ 1920x1080
→ 2560x1440
→ 1920x1080
→ 1280x800
```

wykonany trzy razy uzyskał:

```text
PMM delta  = 0
HEAP delta = 0
BUF delta  = 0
```

Ten test należy zachować, ale rozszerzyć.

Nowa wersja nie może testować samej rozdzielczości.

Musi również otwierać popupy/dialogi między zmianami.

---

# 25. Nowy wymagany test wysokich rozdzielczości

Minimalny scenariusz:

```text
boot 1280x720

open DialogPliku
close DialogPliku × N

switch 1920x1080
open DialogPliku
move dialog
open nested popup/menu
close
repeat × N

switch 2560x1440
repeat

switch 1920x1080
repeat

switch 1280x720
repeat
```

Po każdym pełnym cyklu należy zmierzyć:

```text
PMM free frames
VMM mapped pages
graphics pool bytes
active graphics buffers
popup backing count
combined popup count
```

Po powrocie do baseline:

```text
delta = 0
```

dla każdej kategorii.

---

# 26. Diagnostyka popupów

Przy reprodukcji błędu wysokiej rozdzielczości należy analizować logi:

```text
[DIALOG-OPEN]
[POPUP-DBG]
[COMP-LOCK]
```

Szczególnie istotne:

```text
dialog bounds
popup generation
screen generation
backing size
allocation id
old/new damage
release path
```

Rozbudowane logowanie geometrii powinno być kompilacyjnie bramkowane, np.:

```text
BURSZTYN_DIAG_GEOM_TRACE
```

aby zwykły build nie generował zbędnego spamu.

---

# 27. DialogPliku jako test integracyjny

`DialogPliku` jest dobrym testem, ponieważ jednocześnie angażuje:

- `OknoDialogowe`,
- popup top-level,
- BursztynUI,
- tabelę/drzewo,
- PasekSciezki,
- listowanie BSP2,
- focus,
- capture,
- modalność,
- skalowanie,
- wysoką liczbę elementów UI.

Dlatego awaria przy 1920×1080 nie powinna być redukowana do „błędu samego DialogPliku”.

Może ujawniać problem kompozytora lub graphics-buffer lifetime.

---

# 28. Powiadomienia i tooltipy

Nowszy model powinien również testować:

```text
toast
tooltip
menu kontekstowe
menu główne
```

przy zmianie trybu.

Każdy z tych elementów może posiadać odłączoną powierzchnię i shadow damage.

Test tylko okien aplikacji nie pokrywa już pełnego stanu kompozytora.

---

# 29. `combined popup submission`

Obecna architektura posiada ograniczenie polegające na scalaniu popupów procesu do jednego submission.

W praktyce oznacza to, że zmiana stosu popupów może wymagać ponownego renderu połączonej powierzchni.

To upraszcza ABI, ale zwiększa wymagania dotyczące:

- poprawnego lifetime,
- zwalniania starego submission,
- wyliczania jego rozmiaru,
- synchronizacji z generacją ekranu.

Ten element musi wejść do audytu `[EKRAN] BLAD puli PMM/VMM`.

---

# 30. Przezroczystość

Aktualne popupy mają ograniczoną semantykę przezroczystości.

Nie należy opisywać ich jak pełnego nowoczesnego compositing API z dowolnym per-pixel alpha we wszystkich ścieżkach.

Przy audycie pamięci istotniejsze jest jednak to, że przezroczyste narożniki i cień powiększają obszar damage poza sam prostokąt treści popupu.

---

# 31. Tapeta

Dotychczasowy model pozostaje sensowny:

### Zmiana rozdzielczości

```text
1 przygotowanie/przeskalowanie
```

dla nowego fizycznego rozmiaru.

### Zmiana samej skali UI

Nie powinna wymagać:

```text
ponownego odczytu BTP
ponownego dekodowania BTP
ponownego skalowania fizycznej tapety
```

### Per frame

Compositor powinien kopiować przygotowaną powierzchnię, nie dekodować źródłowy plik.

---

# 32. Pacing a ustawienia ekranu

Pełna kompozycja GUI jest paced przez monotoniczny HPET.

Nominalny minimalny odstęp pełnych klatek:

```text
16 666 667 ns
```

czyli maksymalnie około:

```text
60 Hz
```

Zmiana trybu jest jednak transakcją systemową i nie powinna być błędnie blokowana przez normalny animation pacing.

Pierwsza spójna klatka po commit powinna zostać złożona w kontrolowany sposób.

---

# 33. BWS62 a wiele monitorów

Aktualny system jest praktycznie:

```text
single-monitor
```

Struktura monitor info jest przygotowana rozszerzalnie, ale dokument nie powinien sugerować pełnej wielomonitorowości.

Nie ma jeszcze potwierdzonego kompletnego modelu:

```text
hotplug
per-monitor scale
window migration
multiple work areas
```

---

# 34. Backend GOP/VESA a runtime mode switch

Należy zachować wyraźne rozróżnienie:

### Bochs VBE

```text
runtime mode switch
```

jest podstawową testowaną ścieżką.

### GOP / VESA

w aktualnym modelu powinny być traktowane przede wszystkim jako:

```text
boot-time framebuffer backends
```

bez zakładania identycznej dynamicznej możliwości jak Bochs.

---

# 35. Historyczny margines obrazu kernela

Audyt z 2026-09-06 raportował:

```text
kernel end = 0x0079DCA0
limit      = 0x00E00000
```

czyli około:

```text
7.6 MiB wykorzystane
6.38 MiB historycznego marginesu
```

To **historyczny snapshot**, a nie bieżąca gwarancja.

Nie należy mówić:

```text
jądro ma 14 MiB
```

Poprawnie:

> **0x00E00000 było limitem obrazu kernela w audytowanej konfiguracji, a sam obraz kończył się około 7.6 MiB.**

Po rozbudowie BWS/BursztynUI/aktualizacji należy ponownie odczytać aktualny symbol końca obrazu.

---

# 36. Aktualny szeroki wynik diagnostyczny projektu

Najnowszy szeroki przebieg diagnostyczny projektu nie kończy się już wynikiem całkowicie zielonym.

Zaobserwowano:

```text
PASS assertions = 36349
FAIL assertions = 24
```

oraz grupy zarówno `PASS`, jak i `FAIL`.

Błędy nie dotyczą wyłącznie `Ustawienia ekranu`, dlatego nie należy przepisywać `24 FAIL` jako „24 błędy ekranu”.

Wśród aktualnie wymagających uwagi obszarów są m.in.:

- BursztynUI,
- popupy/dialogi,
- motywy BWS 86/87,
- wybrane Kontrolki3,
- SelektorKoloru,
- raportowanie statusu testów,
- bieżąca regresja PMM/VMM grafiki.

Wniosek dla tego dokumentu:

> **screen audit 31128/0 jest historycznie ważny, ale nie może już stanowić jedynego bieżącego werdyktu dla podsystemu graficznego.**

---

# 37. Problem raportowania testów

W ostatnich testach projektu zdarzało się, że lokalna sekcja drukowała:

```text
OK
```

mimo wzrostu globalnego licznika błędów.

Dlatego każdy test ekranu powinien używać baseline:

```text
errors_before = global_fail_count
...
group_fail = global_fail_count - errors_before
```

i dopiero na tej podstawie drukować:

```text
PASS / FAIL
```

Asercja powinna raportować:

```text
suite
test
expression
file
line
expected
actual
```

Bez tego lokalne `OK` może być mylące.

---

# 38. Testy skalowania BursztynUI

Nowa macierz powinna obejmować każdą skalę:

```text
100
125
150
175
200
```

dla rozdzielczości co najmniej:

```text
1280x720
1920x1080
2560x1440
```

oraz elementów:

```text
DialogPliku
DialogKoloru
Tabela
Drzewo
TabelaWirtualna
ListaWirtualna
PoleWyszukiwania
SelektorKoloru
Tooltip
Toast
MenuKontekstowe
nested popup
```

To lepiej odpowiada aktualnemu BursztynUI niż historyczny zestaw kontrolek.

---

# 39. Testy timeout rollback

Dla każdej reprezentatywnej rozdzielczości:

```text
old mode
→ switch new mode
→ do not confirm
→ wait >15 s
→ rollback
```

należy sprawdzić:

```text
hardware mode restored
screen generation coherent
mouse clamped
windows repositioned
popup stack valid
focus valid
capture valid
PMM delta 0
VMM delta 0
BUF delta 0
ekran.cfg unchanged
```

---

# 40. Testy potwierdzenia

Dla:

```text
old mode
→ new mode
→ confirm
```

należy sprawdzić:

```text
ekran.cfg updated
flush success
read-back correct
old buffers released
new generation retained
next restart loads scale/config correctly
```

---

# 41. Testy błędów alokacji

Należy dodać fault injection:

```text
fail PMM frame #1
fail PMM frame #N/2
fail final PMM frame
fail VMM map after N pages
fail wallpaper allocation
fail popup backing allocation
fail combined popup allocation
```

Po każdym błędzie:

```text
all deltas = 0
old display still functional
no stale lock
no invisible modal
```

To jest kluczowe przed uznaniem obecnej regresji PMM/VMM za naprawioną.

---

# 42. Testy generacji ekranu

Minimalne przypadki:

```text
create popup generation N
switch mode → generation N+1
attempt old popup submit → reject
old backing released
open new popup → generation N+1
```

oraz:

```text
open modal dialog
switch mode
invalidate popup
verify focus/capture/modal state
```

Nie wystarcza sprawdzenie, że stary popup „nie jest widoczny”.

Trzeba sprawdzić, że nie istnieje już jako zasób.

---

# 43. Testy wysokiego DPI/skali bez zmiany trybu

Zmiana:

```text
100% → 125% → 150% → 175% → 200%
```

bez zmiany rozdzielczości nie powinna:

- rezerwować nowego framebuffer-sized backbufferu,
- dekodować tapety od nowa,
- zmieniać generacji fizycznego ekranu, jeżeli generacja oznacza wyłącznie display mode,
- pozostawiać starych popupów w złej skali.

Popupy widoczne podczas zmiany skali powinny być:

```text
rerendered
```

lub:

```text
zamknięte/rebuilt
```

zgodnie z aktualną polityką.

---

# 44. Testy VRAM

Lista trybów powinna być zależna od raportowanego VRAM.

Przykładowa macierz:

```text
16 MiB
32 MiB
64 MiB
```

Dla każdego wariantu:

```text
nie reklamuj trybu niespełniającego size <= VRAM
```

Nie wolno utożsamiać sukcesu testu 64 MiB z gwarancją działania 4K na każdej konfiguracji QEMU/Bochs.

---

# 45. Real hardware

Historyczny audyt miał status:

```text
NOT MANUALLY VERIFIED BY HUMAN
```

Nie ma podstaw, by zamienić to na:

```text
verified on real hardware
```

bez przeprowadzenia testów.

Dodatkowo dynamiczny Bochs VBE jest głównie ścieżką środowisk wirtualnych.

Na fizycznym sprzęcie należy osobno zweryfikować:

- GOP framebuffer,
- pitch,
- pixel format,
- zachowanie skalowania,
- brak runtime mode switch,
- startup config,
- kursor,
- okna,
- powrót do GUI po błędnej konfiguracji.

---

# 46. Aktualne ograniczenia architektoniczne

Podsystem ekranu nadal ma ograniczenia:

```text
jeden monitor logiczny
brak pełnej akceleracji GPU
renderowanie głównie CPU
brak pełnego tile compositor
popup submission scalany per proces
ograniczona przezroczystość popupów
system nadal UP
real hardware niezweryfikowany manualnie
```

VirtIO-GPU może poprawić transport/prezentację, ale nie rozwiąże automatycznie problemów:

```text
ownership
generation
lifetime
damage
focus
capture
```

---

# 47. Zasada naprawy regresji PMM/VMM

Dopuszczalna kolejność pracy:

```text
1. policz live resources
2. dodaj allocation IDs
3. loguj map/unmap
4. zmierz delta PMM/VMM/BUF
5. reprodukuj przy 1920+
6. wskaż brakującą parę alloc/free
7. napraw lifecycle
8. dopiero potem policz legalny peak
9. zwiększ pool tylko jeśli peak tego wymaga
```

Niedopuszczalne jako samodzielna „naprawa”:

```text
podwój limit graphics pool
```

bez udowodnienia zerowego wycieku.

---

# 48. Zalecane liczniki diagnostyczne

Warto utrzymywać:

```text
graphics_buffers_active
graphics_bytes_active
graphics_bytes_peak
pmm_frames_used_by_graphics
vmm_pages_mapped_by_graphics
popup_backings_active
combined_popup_backings_active
display_generation
```

Dla każdego zasobu:

```text
allocation_id
owner_pid
resource_type
width
height
stride
bytes
generation
```

Pozwala to odnaleźć dokładny zasób pozostający po zmianie trybu.

---

# 49. Release gate podsystemu ekranu

Przed ponownym oznaczeniem podsystemu jako `STABILNE` wymagane jest:

```text
[ ] brak [EKRAN] BLAD puli PMM/VMM
[ ] repeated transitions delta=0
[ ] popup/backing delta=0
[ ] DialogPliku 1920x1080 repeated PASS
[ ] DialogPliku 2560x1440 repeated PASS
[ ] timeout rollback PASS
[ ] confirm/persist PASS
[ ] all UI scales PASS
[ ] stale generation resources = 0
[ ] no invisible modal/capture
[ ] fault-injection rollback PASS
[ ] full suite reports local/global FAIL consistently
[ ] manual visual pass in QEMU
[ ] real-hardware GOP pass before hardware-stable claim
```

---

# 50. Podsumowanie aktualnego stanu

Historyczny audyt poprawnie potwierdził dla rewizji z 2026-09-06:

```text
9 trybów Bochs
4K przy 64 MiB VRAM
100–200% UI scale
15 s rollback
ekran.cfg persist
BWS62
dedykowaną pulę grafiki
31128 PASS / 0 FAIL
3x transition delta=0
```

Późniejszy rozwój GUI dodał jednak:

```text
detached top-level popup surfaces
nested popup stack
old/new damage
shadow damage
display generation
bardziej złożone dialogi BursztynUI
```

a następnie ujawnił bieżący problem:

```text
[EKRAN] BLAD puli PMM/VMM
```

Dlatego obecny werdykt powinien brzmieć:

> **Mechanizm zmiany rozdzielczości i skalowania jest funkcjonalny, a jego bazowa architektura transakcyjna pozostaje właściwa. Po rozbudowie popupów i generacji ekranu nie jest jednak jeszcze potwierdzona pełna stabilność lifecycle zasobów graficznych. Priorytetem jest doprowadzenie powtarzanych zmian trybu wraz z popupami/dialogami do zerowych delt PMM/VMM/BUF.**

---

## 51. Dokumenty powiązane

Ten audyt należy czytać razem z:

- `09_tryb_graficzny.md` — HAL i kompozytor,
- `04_audyt_gui_schedulera.md` — input, focus/capture, popupy i pacing,
- `05_hpet_i_pacing.md` — HPET i pacing GUI,
- `06_wywolania_systemowe.md` — BWS 61–63 i nowoczesne ABI,
- `11_audyt_bws_nowoczesne.md` — BWS monitor/work-area,
- dokumentacją BursztynUI i dialogów.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny `bufory_grafiki.cpp`,
2. aktualny VMM/PMM,
3. aktualny kod `grafika_przygotuj_ekran` / `grafika_zwolnij_stan`,
4. aktualny kompozytor i popup lifecycle,
5. aktualne logi runtime,
6. najnowsze testy,
7. historyczny audyt z 2026-09-06.

---

## 52. Funkcje, które należy szczególnie audytować przy obecnej regresji

```text
VmmMapujBuforGrafiki
VmmOdmapujBuforGrafiki

grafika_przygotuj_ekran
grafika_wymien_stan
grafika_zwolnij_stan

skladacz_zastosuj_ekran
skladacz_cofnij_ekran
skladacz_reset_ekranu

popup create/update/destroy
combined popup submission
display-generation invalidation

tapety_przygotuj_ekran
tapety commit/rollback

ekran_obsluz_timeout
ekran confirm/persist
```

Dla każdej z tych ścieżek trzeba wskazać jednoznacznego właściciela zasobu oraz dokładne miejsce zwolnienia.
