# Audyt GUI, wejścia i schedulera

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** kolejki zdarzeń GUI, focus/capture, klawiatura i mysz PS/2/USB HID, HPET/LAPIC, scheduler, pacing kompozytora, popupy oraz cykl życia zasobów graficznych  
**Status:** dokument zastępuje wcześniejszy audyt opisujący stan sprzed kalibracji HPET/LAPIC i przed rozbudową nowoczesnego Event ABI

---

## 1. Cel audytu

Celem tego dokumentu jest opisanie aktualnego modelu współpracy pomiędzy:

- urządzeniami wejścia,
- IRQ,
- kolejkami zdarzeń procesów Ring 3,
- focus/capture,
- schedulerem,
- HPET i LAPIC Timerem,
- Menedżerem Okien,
- kompozytorem,
- popupami i dialogami,
- zasobami grafiki zależnymi od aktualnej generacji ekranu.

Starsza wersja dokumentu zawierała kilka ograniczeń, które zostały już usunięte. W szczególności nieaktualne jest twierdzenie, że LAPIC Timer nie ma kalibracji ani znanej częstotliwości.

---

# 2. Problemy wykryte w pierwotnym modelu

Pierwszy audyt wykazał kilka problemów architektonicznych.

## 2.1 Globalne oczekiwanie na mysz

Legacy BWS 18:

```text
gui_pobierz_mysz
```

blokowało proces przez wspólny mechanizm oczekiwania na zmianę myszy.

Wszystkie aplikacje obserwowały jeden globalny stan, zamiast otrzymywać własne zdarzenia.

Powodowało to między innymi:

- wybudzanie wielu procesów po jednym zdarzeniu,
- brak jednoznacznego adresata kliknięcia,
- możliwość zgubienia zbocza przycisku,
- możliwość uznania tego samego stanu za klik przez kilka aplikacji.

## 2.2 Globalne „przejęcie myszy”

Legacy BWS 22 opierało się na zmiennej w rodzaju:

```text
pid_przejmujacy_mysz
```

i nie implementowało pełnego modelu:

```text
hit-test
→ focus
→ capture
→ routing eventu
```

Ostatni proces wywołujący syscall mógł nadpisać właściciela myszy.

## 2.3 Klawiatura bez prawidłowego focusu

Starszy kod kierował część znaków do wspólnego bufora zamiast do aktywnego okna.

Nie rozdzielał też wystarczająco:

- fizycznego klawisza,
- tekstu Unicode,
- modyfikatorów.

## 2.4 Kompozycja wykonywana przez aplikację

Legacy BWS 17 wykonywało kosztowną ścieżkę odświeżania.

Aplikacje potrafiły wywoływać ją podczas:

- ruchu,
- drag,
- kolejnych zmian kontrolki.

Powodowało to nadmierną liczbę przebiegów kompozytora.

## 2.5 Zegar pulpitu zależny od pollingu myszy

Menedżer Okien sprawdzał RTC w ścieżce zależnej od obsługi myszy.

Przy braku ruchu myszy aktualizacja zegara mogła nie następować regularnie.

## 2.6 Brak kalibracji LAPIC

Pierwotny model używał periodycznego LAPIC z arbitralnym `initial count`.

To ograniczenie zostało później usunięte przez wdrożenie HPET i kalibrację LAPIC.

---

# 3. Aktualny model zdarzeń GUI

Podstawowy model wejścia nie opiera się już na odczytywaniu jednego globalnego stanu przez wszystkie aplikacje.

Każdy proces GUI posiada własną kolejkę zdarzeń.

Nowoczesne API używa między innymi:

```text
BWS 37 — event poll
BWS 38 — event wait
BWS 39 — input capture
```

Legacy BWS:

```text
18 — read mouse state
22 — mouse ownership
```

pozostają częścią historycznego ABI, ale nie powinny być podstawą nowych komponentów BursztynUI.

---

# 4. Routing zdarzenia wejścia

Aktualna ścieżka jest koncepcyjnie następująca:

```text
IRQ / sterownik HID
       ↓
normalizacja zdarzenia
       ↓
hit-test / focus / capture
       ↓
wybór docelowego PID
       ↓
kolejka zdarzeń procesu
       ↓
BWS 37 / BWS 38
       ↓
BursztynUI
       ↓
kontrolka docelowa
```

Zdarzenie nie powinno być globalnie kopiowane do wszystkich procesów.

---

# 5. Focus

Focus określa proces/okno, które otrzymuje wejście klawiatury i zwykłe zdarzenia interakcji.

Kliknięcie w warstwę aplikacji:

```text
MOUSE_DOWN
```

powoduje wybór trafionej warstwy i zmianę aktywnego focusu.

Współczesny model powinien utrzymywać niezależnie:

- aktywne okno,
- focus klawiatury,
- capture wskaźnika.

Są to różne pojęcia i nie powinny być reprezentowane jednym globalnym PID.

---

# 6. Input capture

Capture jest używany podczas interakcji, która rozpoczęła się w jednej kontrolce lub oknie i powinna pozostać skierowana do tego samego procesu mimo opuszczenia jego granic.

Przykłady:

- przeciąganie okna,
- przeciąganie suwaka,
- zaznaczanie,
- operacje drag & drop.

BWS 39 zapewnia systemową podstawę capture.

Podczas capture:

```text
pointer event
    ↓
capture PID
```

ma pierwszeństwo przed normalnym hit-testem.

Capture musi zostać zwolniony między innymi po:

- zakończeniu operacji,
- zamknięciu okna,
- zakończeniu procesu,
- błędzie dialogu,
- bezpiecznym wycofaniu operacji modalnej.

---

# 7. Bounded event queue

Kolejka zdarzeń procesu jest ograniczona.

Proces oczekujący na pustą kolejkę może przejść do stanu blokady zdarzeniowej, a enqueue budzi tylko proces, dla którego zdarzenie jest przeznaczone.

Dzięki temu:

```text
jedno zdarzenie
≠
wybudzenie wszystkich aplikacji
```

Zdarzenia o wysokiej częstotliwości mogą być scalane.

Dotyczy to przede wszystkim:

- ruchu wskaźnika,
- timerów,
- wybranych zdarzeń aktualizacyjnych.

Nie należy jednak scalać zdarzeń, których kolejność ma znaczenie semantyczne, np. niezależnych kliknięć.

---

# 8. Nowoczesne Event ABI

Nowsza warstwa BWS rozszerzyła model wejścia.

Aktualnie udokumentowane są między innymi:

| Event | Znaczenie |
|---:|---|
| 23 | fizyczne zdarzenie klawiatury |
| 24 | `TEXT_INPUT` Unicode |
| 25 | pionowe/poziome koło myszy |
| 26 | jawne zdarzenie precyzyjnego wskaźnika |
| 27 | file watcher |
| 28 | gotowość / powiadomienie IPC |

Struktura `bws_zdarzenie` zachowuje rozmiar 40 bajtów.

---

# 9. Klawiatura: klawisz fizyczny a tekst

Aktualny model oddziela:

```text
KEY
```

od:

```text
TEXT_INPUT
```

Jest to ważne dla poprawnego GUI.

Fizyczny klawisz służy do:

- skrótów,
- nawigacji,
- sterowania fokusem,
- klawiszy funkcyjnych,
- obsługi CTRL/ALT/SUPER.

`TEXT_INPUT` reprezentuje tekst Unicode, który ma zostać wpisany do kontrolki.

Dzięki temu kontrolka tekstowa nie musi samodzielnie rekonstruować Unicode z surowych scancode.

---

# 10. Modyfikatory klawiatury

Event ABI posiada sześciobitową maskę modyfikatorów.

Pozwala przekazywać stan kombinacji takich jak:

```text
SHIFT
CTRL
ALT
SUPER
CAPS
NUM
```

Bez osobnego publicznego syscalla.

Stan jest zapisywany w istniejących polach struktury zdarzenia.

Pełne:

- layouty klawiatury,
- dead keys,
- zaawansowane composition,
- IME

pozostają przyszłą warstwą Ring 3 / usług systemowych.

---

# 11. PS/2 i USB HID

Starsza dokumentacja opisywała wejście głównie przez PS/2.

Aktualny Bursztyn OS posiada również natywną obsługę:

```text
USB xHCI
+
USB HID keyboard
+
USB HID mouse
```

PS/2 nadal może stanowić starszą ścieżkę kompatybilności.

## 11.1 Klawiatura USB

Sterownik HID przekazuje znormalizowane zdarzenia do wspólnego systemowego Event ABI.

## 11.2 Mysz USB

Aktualna obsługa obejmuje:

- ruch relatywny,
- przyciski,
- koło myszy dla obsługiwanych raportów,
- routing do focus/capture PID.

---

# 12. Koło myszy

Event 25 obsługuje:

- pionową wartość signed,
- poziomą wartość signed.

Aktualne pokrycie sprzętowe pozostaje częściowe.

Obsługiwane są raporty USB HID zawierające odpowiednie bajty wheel.

Nadal brakuje pełnego:

- PS/2 wheel negotiation,
- uniwersalnego parsera dowolnego HID report descriptor.

---

# 13. High-resolution pointer

Event 26 dodaje jawny kontrakt zdarzenia wskaźnika wraz z:

- współrzędnymi,
- relatywnym ruchem,
- stanem przycisków,
- modyfikatorami,
- timestampem.

Aktualny sprzętowy backend PS/2 oraz boot-HID nadal operuje głównie na relatywnych delta o ograniczonej rozdzielczości.

Pełne wykorzystanie urządzeń high-resolution wymaga dalszego parsowania deskryptorów HID.

---

# 14. HPET jako źródło czasu

Najważniejsza zmiana względem starego audytu:

> **LAPIC Timer jest już kalibrowany.**

Bursztyn OS wykrywa HPET przez ACPI.

Parser:

- obsługuje ACPI RSDP old/new,
- waliduje checksum RSDT/XSDT,
- waliduje tabele SDT,
- akceptuje HPET z GAS `System Memory`,
- nie używa na sztywno wpisanego fizycznego adresu HPET.

HPET jest mapowany przez VMM jako:

```text
supervisor
writable
PWT
PCD
```

Main counter jest używany jako monotoniczne źródło czasu.

---

# 15. Kalibracja LAPIC

LAPIC zachowuje divider:

```text
16
```

Podczas bootu licznik jest uruchamiany od:

```text
0xffffffff
```

na okres **20 ms** mierzony przez HPET.

Różnica `current count` umożliwia obliczenie realnej częstotliwości timera po dividerze.

Dopiero z tej wartości wyliczany jest periodic initial count dla:

```text
250 Hz
```

W przypadku braku HPET działa jawny fallback sygnalizowany logiem:

```text
TIMER-WARN
```

Stare ograniczenie:

```text
„LAPIC nie jest skalibrowany”
```

jest więc nieaktualne.

---

# 16. Scheduler

Projekt nadal jest **jednordzeniowy**.

Scheduler działa w modelu Round-Robin.

PID 0 uczestniczy w zwykłym planowaniu i wykonuje pracę systemową, która nie powinna być wykonywana bezpośrednio w IRQ.

Aktualna architektura zachowuje zasadę:

```text
IRQ
→ minimalna obsługa
→ ustawienie stanu / event
→ EOI
→ dalsza praca poza IRQ
```

---

# 17. Timer i preempcja

Ważne pozostaje rozróżnienie dwóch przypadków.

## 17.1 Timer przerywa kod Ring 3

ISR ma prawidłową ramę przerwania i scheduler może podjąć decyzję o przełączeniu kontekstu zgodnie z aktualną architekturą.

## 17.2 Timer przerywa synchroniczny syscall Ring 0

Rama wejścia przez `SYSCALL` nie jest tym samym co zwykła rama przejścia Ring 3 → ISR.

Nie należy wymuszać nieprzemyślanego context switcha w środku kodu Ring 0 wyłącznie dlatego, że pojawił się timer.

To nadal ma znaczenie dla długich operacji sieciowych i TLS.

---

# 18. EOI

EOI jest wysyłane centralnie przez warstwę IDT/APIC.

Sterowniki urządzeń nie powinny wysyłać dodatkowego EOI dla tego samego przerwania.

Zasada:

```text
jedno obsłużone IRQ
→ jedno kontrolowane EOI
```

zapobiega podwójnemu potwierdzaniu przerwań.

---

# 19. GUI refresh

Aktualny model nie zakłada wykonywania pełnej kompozycji bezpośrednio z BWS wywołanego przez aplikację.

Żądanie odświeżenia powinno oznaczać:

```text
oznacz stan jako dirty
```

a nie:

```text
natychmiast składaj cały desktop w kontekście aplikacji
```

Właściwą kompozycję wykonuje kod systemowy poza IRQ.

---

# 20. Pacing kompozytora

HPET służy również do pacingu GUI.

Pending dirty pozostaje aktywne do terminu kompozycji.

Maksymalna częstotliwość pełnej klatki w aktualnym modelu odpowiada okresowi około:

```text
16 666 667 ns
```

czyli około:

```text
60 Hz
```

Nie oznacza to, że system zawsze generuje 60 pełnych klatek na sekundę.

Jeżeli nic się nie zmieniło, kolejna kompozycja nie jest wymagana.

---

# 21. Zegar pulpitu

Zegar pulpitu nie jest już uzależniony od pętli myszy.

Timer systemowy dostarcza pulpitowi event około raz na sekundę według HPET.

Pulpit:

1. odbiera `TIMER`,
2. odczytuje RTC,
3. porównuje czas,
4. oznacza GUI jako wymagające aktualizacji tylko przy rzeczywistej zmianie.

---

# 22. Kursor

Kursor jest rysowany jako overlay końcowego etapu kompozycji.

IRQ myszy nie powinien wykonywać:

```text
hide old cursor
restore backbuffer
draw new cursor
```

na potencjalnie nieaktualnej klatce.

Model overlay eliminuje zależność IRQ od bieżącej wersji backbufferu.

---

# 23. Z-Order

Kliknięcie w aplikację może podnieść jej warstwę w Z-Order.

Warstwa pulpitu:

```text
z_order <= 0
```

nie powinna być przypadkowo podnoszona ponad zwykłe aplikacje.

Z-Order, focus oraz capture są powiązane, ale nie są tym samym stanem.

---

# 24. Kompozytor — stan bazowy

Historyczny model kompozytora używał jednego globalnego `dirty`.

Oznaczało to:

```text
dirty = true
→ następna klatka składa cały ekran
```

HPET/pacing znacznie ograniczył liczbę takich pełnych kompozycji, ale sam globalny `dirty` nie jest odpowiednikiem kompletnego systemu dirty rectangles.

---

# 25. Nowsza warstwa popupów

Nowsze prace nad BursztynUI wprowadziły osobny model popupów.

Popupy, menu, dialogi i część powiadomień nie powinny być przycinane do powierzchni właściciela.

Aktualna architektura popupów obejmuje między innymi:

- odłączone powierzchnie top-level,
- publikowanie popupów do kompozytora,
- obsługę stosu popupów,
- popupy zagnieżdżone,
- damage starego i nowego obszaru popupu,
- uwzględnianie cieni,
- przywracanie focusu,
- współpracę z capture.

W związku z tym stary opis:

> „kompozytor ma wyłącznie jeden globalny dirty i żadnych lokalnych damage regions”

nie opisuje już w pełni całej bieżącej ścieżki popupów.

Jednocześnie nie należy deklarować, że cały kompozytor został już przepisany na kompletny tile/damage compositor.

---

# 26. Popupy a owner window

Popup top-level nie powinien być przycinany przez prostokąt okna właściciela.

Dotyczy to między innymi:

- menu rozwijanych,
- menu kontekstowych,
- `DialogPliku`,
- `DialogKoloru`,
- tooltipów,
- toastów.

Kompozytor powinien traktować ich widoczny prostokąt jako osobny element sceny.

---

# 27. Drag i damage popupu

Przesunięcie popupu lub dialogu wymaga odświeżenia:

```text
old bounds
+
new bounds
```

oraz odpowiedniej przestrzeni cienia.

Brak damage starego obszaru prowadzi do pozostawiania śladów podczas przeciągania.

---

# 28. Generacja ekranu

Po zmianie fizycznego trybu graficznego framebuffer i zależne od niego zasoby otrzymują nową generację.

Powierzchnia popupu posiada powiązanie z:

```text
generacja_ekranu
```

Jeżeli:

```text
popup.generation != active_display.generation
```

stara powierzchnia nie może zostać użyta do kompozycji nowego ekranu.

Mechanizm został wprowadzony po problemach obserwowanych przy przejściu z 1280×720 do 1920×1080.

---

# 29. Inwalidacja generacji nie może oznaczać wycieku

Samo oznaczenie starej powierzchni jako:

```text
invalid
```

nie wystarcza.

Zasób może nadal posiadać:

- ramki PMM,
- mapowania VMM,
- slot w puli buforów grafiki,
- backing storage.

Dlatego cykl życia musi być:

```text
invalidate
→ detach
→ release backing resource
```

albo:

```text
rebuild for new generation
```

bez pozostawiania zasobów starego framebufferu.

---

# 30. Aktualna regresja puli PMM/VMM

W najnowszym etapie rozwoju po zmianach GUI/popup/display-generation pojawił się log:

```text
[EKRAN] BLAD puli PMM/VMM
```

Ten błąd jest obecnie traktowany jako **regresja do zdiagnozowania**, a nie jako oczekiwane ograniczenie architektury.

Podejrzane obszary obejmują:

- cykl życia popup surfaces,
- combined popup submission,
- zwalnianie zasobów starej generacji,
- rollback zmiany trybu,
- częściowo wykonane alokacje,
- accounting w dedykowanej puli grafiki.

Do czasu zamknięcia regresji dokumentacja nie powinna twierdzić, że wszystkie wielokrotne zmiany rozdzielczości i popupy są w pełni stabilne.

---

# 31. Dedykowana pula grafiki

Duże bufory grafiki nie są alokowane z małej sterty kernela.

Podsystem używa dedykowanej puli PMM/VMM.

Udokumentowana baza przestrzeni wirtualnej:

```text
BUFORY_GRAFIKI_BAZA = 0xffff900000000000
```

HHDM znajduje się w osobnej gałęzi:

```text
0xffff800000000000
```

Odmapowanie bufora powinno:

1. odczytać fizyczną ramkę z PTE,
2. wyczyścić PTE,
3. wykonać `invlpg`,
4. zwrócić ramkę do PMM,
5. zwolnić odpowiedni zakres/slot wirtualny i metadata allocation.

Każda poprawna alokacja musi mieć dokładnie jednego właściciela i jedną końcową ścieżkę release.

---

# 32. Tryby obrazu a pamięć GUI

Audyt ekranu testował między innymi:

```text
1280x800
→ 1920x1080
→ 2560x1440
→ 1920x1080
→ 1280x800
```

w trzech cyklach.

Historyczny wynik audytu dla tej wersji kodu:

```text
PMM delta  = 0
HEAP delta = 0
BUF delta  = 0
```

Był to wynik konkretnej wersji sprzed późniejszych zmian popup/compositor.

Pojawienie się aktualnego:

```text
[EKRAN] BLAD puli PMM/VMM
```

oznacza, że test leak-free transitions należy ponownie wykonać po naprawie obecnej regresji.

---

# 33. Zmiana rozdzielczości

Aktualny podsystem ekranu posiada:

- dynamiczną listę trybów Bochs VBE,
- przygotowanie nowych buforów przed commit,
- aktualizację geometrii kompozytora,
- 15-sekundowe potwierdzenie,
- rollback do poprzedniego trybu,
- zapis zatwierdzonej konfiguracji.

Operacje kosztowne:

- alokacja,
- zerowanie,
- przygotowanie tapety

powinny odbywać się poza globalnym lockiem ekranu.

Lock powinien chronić możliwie krótki atomowy commit.

---

# 34. Lock ekranu

Audyt podsystemu ekranu usunął globalne opakowanie całej operacji BWS jednym długim `GuardEkranu`.

Docelowy wzorzec:

```text
prepare new resources outside lock
        ↓
lock
        ↓
atomic state swap
        ↓
update geometry/generation
        ↓
unlock
        ↓
release old resources
```

Nie należy wykonywać dużych alokacji PMM/VMM ani skalowania tapety pod spinlockiem.

---

# 35. Cleanup procesu

Starszy `SYS_EXIT` wykonywał już deferred cleanup części zasobów, ale audyt wykazał konieczność rozszerzenia cleanupu GUI.

Aktualny model procesu powinien czyścić między innymi:

- warstwę okna,
- focus, jeśli należy do procesu,
- capture, jeśli należy do procesu,
- kolejkę zdarzeń,
- custom cursor,
- popupy,
- SHM,
- IPC,
- file watchers,
- socket.

Po zakończeniu procesu nie może pozostać:

```text
focus PID = dead process
capture PID = dead process
popup owner = dead process
```

---

# 36. Sieć i TLS

To ograniczenie ze starego audytu pozostaje istotne.

HTTP/HTTPS/TLS posiadają nadal istotne ścieżki synchroniczne.

Długa operacja Ring 0 może pogorszyć responsywność GUI, ponieważ nie należy wykonywać niebezpiecznego context switcha w środku arbitralnej ramy wejścia syscall tylko po to, aby wymusić interaktywność.

Docelowe rozwiązanie wymaga architektury w rodzaju:

```text
request
→ WAITING_IO
→ worker / state machine
→ completion
→ wake process
```

Nowsza warstwa BWS celowo **nie udaje**, że pełne async I/O już istnieje.

---

# 37. Socket facade nie rozwiązuje jeszcze całej współbieżności sieci

BWS 79–83 zapewnia:

- socket create,
- connect,
- send,
- receive,
- close.

Underlying TCP stack ma jednak nadal ograniczenie:

```text
jedna systemowa sesja TCP
```

Brakuje również pełnego readiness event dla wielu równoległych socketów.

To ma znaczenie dla przyszłej responsywności GUI aplikacji sieciowych.

---

# 38. Jednordzeniowość

Projekt nadal działa jako system UP.

Oznacza to, że:

- kernel,
- scheduler,
- sterowniki,
- kompozytor

nie muszą jeszcze obsługiwać pełnej równoległości kilku CPU.

Nie należy jednak budować nowych struktur w sposób uniemożliwiający przyszłe SMP.

Szczególnie w kompozytorze należy unikać długoterminowego przechowywania niechronionych surowych pointerów do powierzchni, które mogą w przyszłości zostać zwolnione przez inny kontekst.

Dla SMP potrzebne będą mechanizmy typu:

- refcount,
- snapshot,
- ownership,
- lock lub RCU-like lifetime model zależnie od struktury.

---

# 39. Co zostało naprawione względem starego audytu

### Naprawione / zastąpione

- globalne oczekiwanie wszystkich procesów na mysz,
- routing kliknięcia wyłącznie na podstawie ostatniego syscalla,
- rekonstrukcja kliknięcia wyłącznie z globalnego stanu,
- brak oddzielnego Event ABI dla procesów,
- brak jawnego keyboard focus model,
- zależność zegara pulpitu od pętli myszy,
- natychmiastowa kompozycja w IRQ/aplikacji,
- brak kalibracji LAPIC,
- brak monotonicznego HPET czasu,
- brak eventów physical key / text input,
- brak maski modifierów,
- brak wheel event,
- brak jawnego pointer event,
- brak USB xHCI/HID jako nowoczesnej ścieżki wejścia.

### Nadal ograniczone

- synchroniczny networking/TLS,
- brak pełnego async I/O,
- jedna sesja TCP,
- pełny parser HID descriptors,
- PS/2 wheel,
- prawdziwe high-resolution HID,
- pełny compositor damage model dla całej sceny,
- SMP.

### Aktualnie ponownie badane

- lifecycle powierzchni popupów po zmianie generacji,
- zużycie dedykowanej puli PMM/VMM,
- wielokrotne przełączanie wysokich rozdzielczości,
- popupy i dialogi po zmianie 1280×720 → 1920×1080.

---

# 40. Status testów

Należy rozróżniać wyniki różnych wersji kodu.

## Audyt HPET/LAPIC

Potwierdza:

- wykrywanie HPET przez ACPI,
- kalibrację LAPIC,
- target 250 Hz,
- pacing GUI około 60 Hz,
- wykonywanie pending compositor work przez PID 0 poza IRQ.

## Audyt ustawień ekranu z 2026-09-06

Raportował:

```text
PASS assertions = 31128
FAIL assertions = 0
```

i stabilne delty pamięci w testowanym wówczas kodzie.

Status tego audytu:

```text
NOT MANUALLY VERIFIED BY HUMAN
```

## Obecny stan

Po późniejszych zmianach GUI/popup/display-generation wystąpiła regresja:

```text
[EKRAN] BLAD puli PMM/VMM
```

Dlatego historycznego `31128 / 0` nie należy przedstawiać jako dowodu braku regresji w obecnym HEAD.

Po naprawie trzeba ponownie wykonać testy:

```text
PMM delta
VMM mapping delta
graphics pool delta
popup lifecycle
mode transitions
```

---

# 41. Wymagane testy regresyjne

Minimalny zestaw powinien obejmować:

### Wejście

```text
PS/2 keyboard
PS/2 mouse
USB HID keyboard
USB HID mouse
wheel
modifier combinations
physical key
TEXT_INPUT
```

### Focus/capture

```text
click A → keyboard goes to A
click B → keyboard goes to B
drag outside window → capture stays with owner
release → capture cleared
process exit during capture → capture cleared
```

### Kolejki

```text
event wait
single-process wake
queue pressure
coalesced pointer move
multiple clicks not lost
```

### Scheduler

```text
250 Hz calibrated timer
PID 0 scheduled
no rendering in IRQ
desktop timer once per second
```

### Grafika

```text
1280x720
→ 1920x1080
→ confirm
→ popup/dialog open/close
→ 2560x1440
→ 1920x1080
→ 1280x720
```

Po pełnym cyklu:

```text
PMM delta = 0
VMM mapped-page delta = 0
graphics-buffer delta = 0
stale popup count = 0
```

---

# 42. Najważniejsze zasady architektoniczne

Aktualny model GUI i schedulera powinien utrzymywać następujące reguły:

1. IRQ nie wykonuje kompozycji całego ekranu.
2. Jedno zdarzenie budzi tylko proces, dla którego jest przeznaczone.
3. Focus i capture są niezależnymi stanami.
4. TEXT_INPUT jest oddzielony od fizycznego klawisza.
5. HPET jest źródłem czasu, a LAPIC jest kalibrowany.
6. Pełna klatka jest wykonywana poza IRQ.
7. Kursor jest overlayem końcowego etapu.
8. Popup top-level nie jest przycinany do owner window.
9. Zmiana generacji ekranu unieważnia stare powierzchnie.
10. Inwalidacja musi prowadzić do zwolnienia lub przebudowania zasobu.
11. Duże bufory grafiki korzystają z dedykowanej puli PMM/VMM.
12. Alokacje i kosztowne przygotowanie nie powinny odbywać się pod spinlockiem ekranu.
13. Cleanup procesu usuwa focus/capture/popupy i pozostałe zasoby.
14. Synchroniczne TLS nie powinno być maskowane sztucznym context switchem w środku syscall.
15. Projekt nadal jest UP i architektura musi zostać przygotowana na przyszłe SMP.

---

# 43. Podsumowanie

Pierwotny audyt GUI wykrył prawdziwe problemy z globalnym wejściem, niewłaściwym routingiem myszy, pollingiem zegara i niekalibrowanym timerem.

Od tego czasu architektura została znacząco rozbudowana:

```text
globalny stan myszy
        ↓
per-process event queue

brak focus routing
        ↓
focus + capture + hit-test

klawisz/tekst razem
        ↓
physical KEY + Unicode TEXT_INPUT

arbitralny LAPIC count
        ↓
HPET-calibrated LAPIC 250 Hz

kompozycja z IRQ/aplikacji
        ↓
pending dirty + PID 0 + pacing

popup w owner surface
        ↓
detached top-level popup surfaces

brak generacji zasobów ekranu
        ↓
display generation validation
```

Najważniejszym obecnym problemem nie jest już kalibracja LAPIC, lecz poprawny **lifetime zasobów graficznych po zmianie generacji ekranu**, ponieważ w najnowszym stanie pojawiła się regresja:

```text
[EKRAN] BLAD puli PMM/VMM
```

Do czasu jej naprawy nie należy opisywać podsystemu GUI jako w pełni stabilnego przy wielokrotnych zmianach wysokiej rozdzielczości.

---

# 44. Dokumenty powiązane

Aktualny stan należy odczytywać razem z:

- `02_architektura_systemu.md`
- `04_zarzadzanie_sprzetem.md`
- `05_hpet_i_pacing.md`
- `09_tryb_graficzny.md`
- `11_audyt_bws_nowoczesne.md`
- `ustawienia_ekranu_audyt.md`

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod,
2. aktualny log runtime,
3. najnowszy audyt danego podsystemu,
4. starsze dokumenty opisowe.

Szczególnie:

- `05_hpet_i_pacing.md` zastępuje stare twierdzenie o niekalibrowanym LAPIC,
- `11_audyt_bws_nowoczesne.md` zastępuje stary model wejścia w zakresie Event ABI,
- `ustawienia_ekranu_audyt.md` opisuje architekturę puli grafiki i zmian trybu,
- bieżąca regresja PMM/VMM wymaga ponownej walidacji wyników dotyczących leak-free mode transitions.
