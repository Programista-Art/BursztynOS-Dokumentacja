# HPET, LAPIC i pacing GUI

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** monotoniczne źródło czasu, HPET, kalibracja LAPIC, clockevent 250 Hz, BWS 60, pacing kompozytora, timer pulpitu i zależności GUI  
**Status:** aktualny opis architektury czasu; zastępuje starsze informacje o niekalibrowanym LAPIC Timerze

---

## 1. Rola podsystemu czasu

Bursztyn OS rozdziela obecnie trzy różne zastosowania czasu:

```text
HPET
  → monotoniczne źródło czasu
  → kalibracja LAPIC
  → deadline/pacing GUI

LAPIC Timer
  → periodyczny clockevent schedulera
  → 250 Hz

RTC CMOS
  → czas kalendarzowy widoczny dla użytkownika
```

Te trzy mechanizmy nie powinny być ze sobą utożsamiane.

RTC nie jest źródłem czasu dla planisty, a arbitralna wartość `initial count` LAPIC nie jest już podstawą deklarowanej częstotliwości schedulera.

---

## 2. Wykrywanie ACPI

Multiboot2 może przekazać opcjonalny tag:

```text
ACPI old
```

lub:

```text
ACPI new
```

Bursztyn OS wykorzystuje te dane do odnalezienia struktur ACPI.

Parser waliduje:

- RSDP,
- checksum RSDP,
- RSDT lub XSDT,
- checksum każdej używanej tabeli SDT,
- sygnaturę poszukiwanej tabeli.

Adres HPET nie jest wpisany na sztywno.

---

## 3. Wykrywanie tabeli HPET

Tabela:

```text
HPET
```

jest akceptowana tylko wtedy, gdy jej opis sprzętowy spełnia wymagania aktualnej implementacji.

Sprawdzane są między innymi:

- GAS `System Memory`,
- niezerowy adres,
- zerowy bit offset,
- poprawność tabeli ACPI.

Jądro nie powinno zakładać historycznego, typowego adresu HPET bez odczytania ACPI.

---

## 4. Mapowanie HPET przez VMM

Rejestry HPET są pamięcią MMIO i nie są traktowane jak zwykły RAM.

Aktualne mapowanie VMM używa właściwości:

```text
supervisor
writable
PWT
PCD
```

Dzięki temu Ring 3 nie otrzymuje bezpośredniego dostępu do MMIO HPET.

Publiczne API udostępnia wynik pomiaru czasu, a nie fizyczny adres urządzenia.

---

## 5. Main Counter

Aktualna implementacja używa głównego licznika HPET jako monotonicznego źródła czasu.

Main counter jest uruchamiany bez używania comparator IRQ jako podstawowego mechanizmu planisty.

Schemat:

```text
HPET main counter
      ↓
odczyt ticków
      ↓
przeliczenie według COUNTER_CLK_PERIOD
      ↓
czas monotoniczny
```

To oznacza, że HPET pełni obecnie przede wszystkim rolę:

- zegara odniesienia,
- źródła monotonicznego czasu,
- wzorca kalibracyjnego dla LAPIC.

---

## 6. `COUNTER_CLK_PERIOD`

HPET raportuje okres jednego ticka przez:

```text
COUNTER_CLK_PERIOD
```

Na jego podstawie Bursztyn OS wylicza częstotliwość i przelicza ticki na jednostki czasu.

Implementacja nie wymaga zmiennoprzecinkowego `float`/`double`.

Przeliczenie wykorzystuje arytmetykę całkowitą z rozdzieleniem:

```text
quotient
+
remainder
```

co ogranicza ryzyko utraty precyzji i pozwala używać mechanizmu w kodzie jądra bez FPU.

---

## 7. Licznik 32-bitowy i wrap

HPET może udostępniać licznik o szerokości 32 bitów.

Aktualna architektura uwzględnia wrap i rozszerza epokę programowo.

Koncepcyjnie:

```text
new_counter < previous_counter
        ↓
wykrycie wrap
        ↓
zwiększenie programowej epoki
```

Dzięki temu monotoniczny czas udostępniany przez system nie powinien cofać się przy zwykłym przepełnieniu 32-bitowego main countera.

---

## 8. BWS 60 — monotoniczny czas Ring 3

BWS 60 pozostaje publicznym API czasu monotonicznego:

```text
BWS 60 — monotonic HPET nanoseconds
```

Wywołanie:

- nie udostępnia MMIO,
- nie zwraca fizycznego adresu HPET,
- zwraca czas monotoniczny w nanosekundach,
- może zwrócić `0`, gdy HPET nie jest dostępny.

W nowszym rozszerzeniu ABI numer 60 został zachowany i nie utworzono drugiego syscalla dla tego samego mechanizmu.

Aplikacje powinny używać czasu monotonicznego do:

- pomiaru czasu trwania operacji,
- animacji,
- timeoutów,
- deadline,
- profilowania,

a RTC do prezentacji daty i godziny.

---

## 9. LAPIC jako clockevent

Local APIC Timer jest używany jako periodyczne źródło przerwań schedulera.

Aktualna konfiguracja zachowuje divider:

```text
16
```

Nie używa już jednak arbitralnego, niezweryfikowanego `initial count` jako podstawy deklaracji 250 Hz.

---

## 10. Kalibracja LAPIC przez HPET

Podczas bootu LAPIC jest kalibrowany względem HPET.

Aktualna procedura:

1. ustawia licznik LAPIC na:

```text
0xffffffff
```

2. rozpoczyna pomiar;
3. odmierza **20 ms** przy pomocy HPET;
4. odczytuje aktualny `current count`;
5. oblicza liczbę ticków LAPIC przypadających na zmierzone 20 ms;
6. wylicza częstotliwość po dividerze;
7. oblicza `periodic initial count` dla:

```text
250 Hz
```

Koncepcyjnie:

```text
HPET 20 ms
    ↓
delta LAPIC count
    ↓
częstotliwość LAPIC
    ↓
initial count dla 250 Hz
```

Dzięki temu 250 Hz jest wartością wyliczoną z pomiaru sprzętowego.

---

## 11. Fallback bez HPET

Jeżeli HPET nie może zostać poprawnie wykryty lub uruchomiony, Bursztyn OS posiada jawny fallback.

Sytuacja jest raportowana logiem:

```text
TIMER-WARN
```

Fallback nie powinien być przedstawiany jako równoważny dokładności poprawnie skalibrowanego HPET/LAPIC.

W testach i diagnostyce należy rozróżniać:

```text
HPET calibration path
```

od:

```text
fallback path
```

---

## 12. Częstotliwość schedulera

Po poprawnej kalibracji periodic LAPIC Timer jest ustawiany na:

```text
250 Hz
```

co odpowiada nominalnemu okresowi około:

```text
4 ms
```

na tick.

Nie oznacza to, że każda aplikacja otrzymuje dokładnie jeden kwant co 4 ms. Rzeczywiste planowanie zależy od:

- stanu procesu,
- kolejki runnable,
- blokad,
- aktualnego miejsca wykonania,
- reguł Round-Robin,
- kodu wykonywanego w Ring 0.

---

## 13. Scheduler pozostaje jednordzeniowy

Aktualny Bursztyn OS pozostaje systemem UP.

Scheduler Round-Robin działa na jednym logicznym CPU.

HPET i kalibracja LAPIC nie oznaczają jeszcze:

- SMP,
- per-CPU timerów,
- per-CPU runqueue,
- synchronizacji między rdzeniami.

Te elementy należą do przyszłej architektury SMP.

---

## 14. Timer IRQ a kod Ring 0

Ważne pozostaje ograniczenie opisane we wcześniejszym audycie schedulera.

Timer może przerwać zarówno:

```text
Ring 3
```

jak i kod działający już w:

```text
Ring 0
```

Nie należy wykonywać arbitralnego context switcha w środku synchronicznego syscalla tylko dlatego, że pojawił się tick LAPIC.

Rama wejścia przez `SYSCALL` nie jest tym samym kontraktem co zwykła rama przerwania procesu Ring 3.

Ma to znaczenie zwłaszcza dla długich, synchronicznych ścieżek:

- TCP,
- HTTP,
- HTTPS,
- TLS.

Docelowe rozwiązanie długiego I/O powinno opierać się na:

```text
request
→ WAITING_IO
→ worker/state machine
→ completion
→ wake process
```

Pełny async I/O nie jest obecnie zaimplementowany.

---

## 15. Minimalna praca w timer IRQ

Timer IRQ nie powinien wykonywać ciężkiej pracy.

Docelowy model:

```text
LAPIC IRQ
    ↓
aktualizacja stanu czasu / schedulera
    ↓
ustawienie pending event/work
    ↓
EOI
    ↓
powrót / decyzja schedulera
```

W szczególności ISR nie powinien wykonywać:

- pełnej kompozycji ekranu,
- dużych alokacji,
- skalowania tapety,
- operacji plikowych,
- synchronicznego TLS,
- rozbudowanego logowania.

---

## 16. PID 0 i praca kompozytora

PID 0 jest normalnym uczestnikiem Round-Robin.

Oczekująca praca kompozytora jest wykonywana przez PID 0 **poza IRQ**.

Schemat:

```text
aplikacja / event
      ↓
dirty request
      ↓
pending compositor work
      ↓
deadline osiągnięty
      ↓
PID 0
      ↓
kompozycja
```

Dzięki temu kod timera nie musi składać framebufferu w kontekście przerwania.

---

## 17. `gui_odswiez()` i dirty state

Żądanie odświeżenia GUI nie powinno oznaczać natychmiastowej pełnej kompozycji w kontekście procesu wywołującego.

Aktualna zasada:

```text
gui_odswiez()
→ oznacz jako dirty
```

a następnie:

```text
PID 0
→ wykonaj kompozycję, gdy pozwala pacing
```

To ogranicza liczbę kosztownych przejść po framebufferze podczas:

- drag,
- ruchu wskaźnika,
- szybkich zmian kontrolek,
- serii zdarzeń wejścia.

---

## 18. Pacing kompozytora

Aktualny limit pełnej klatki odpowiada deadline:

```text
16 666 667 ns
```

czyli około:

```text
60 Hz
```

Dirty request nie jest kasowany tylko dlatego, że klatka nie może być jeszcze wykonana.

Pozostaje aktywny do momentu osiągnięcia deadline.

Koncepcyjnie:

```text
dirty = true
       ↓
czy minęło 16 666 667 ns?
       ├── NIE → zachowaj dirty
       └── TAK → compositor może złożyć klatkę
```

---

## 19. 60 Hz nie oznacza ciągłego renderowania

Limit pacingu jest **maksymalną częstotliwością pełnej kompozycji**, a nie wymuszeniem stałego render loop.

Jeżeli scena się nie zmieniła:

```text
dirty = false
```

to compositor nie powinien wykonywać pełnej klatki tylko dlatego, że minęło kolejne 16,67 ms.

Celem jest:

- ograniczenie pracy CPU,
- brak niepotrzebnych kopii framebufferu,
- zachowanie responsywności,
- stabilniejszy czas klatki.

---

## 20. Stan dirty i damage

Historyczny audyt GUI opisywał compositor jako oparty głównie na globalnym:

```text
dirty
```

a nie pełnym systemie dirty rectangles.

To nadal jest poprawny opis bazowej ścieżki pełnej kompozycji.

Nowsze prace nad:

- geometrią okien,
- popupami,
- dialogami,
- cieniami

wprowadziły jednak bardziej lokalne operacje na starym i nowym obszarze zasobu.

Dlatego aktualny stan należy opisywać ostrożnie:

> **Bazowy compositor nadal nie jest pełnym tile/damage-region compositor, ale wybrane ścieżki GUI i popupów korzystają już z informacji o starych i nowych obszarach wymagających ponownego złożenia.**

Nie należy ani twierdzić, że system nadal posiada wyłącznie jeden globalny dirty w każdej ścieżce, ani że pełny compositor dirty-rect jest już ukończony.

---

## 21. Timer pulpitu

Zegar pulpitu nie zależy już od ruchu myszy.

System dostarcza pulpitowi scalany event timerowy około:

```text
1 raz / sekundę
```

według monotonicznego czasu HPET.

Menedżer Okien:

1. odbiera event,
2. odczytuje RTC,
3. porównuje aktualną wartość czasu kalendarzowego,
4. żąda odświeżenia tylko wtedy, gdy wyświetlana wartość rzeczywiście się zmieniła.

Rozdziela to:

```text
HPET → kiedy sprawdzić
RTC  → jaki czas wyświetlić
```

---

## 22. Scalanie eventów timerowych

Timer GUI może być scalany, ponieważ dla zegara pulpitu nie ma potrzeby dostarczenia procesu osobno każdego ticka LAPIC.

Przykładowo:

```text
wiele ticków 250 Hz
      ↓
logika deadline
      ↓
jeden event TIMER dla pulpitu
```

Zmniejsza to presję na kolejkę zdarzeń Ring 3.

---

## 23. Kursor jako overlay

Kursor jest rysowany jako overlay końcowego etapu kompozycji.

IRQ myszy nie wykonuje już schematu:

```text
ukryj stary kursor
→ odtwórz fragment starego backbufferu
→ narysuj kursor w nowym miejscu
```

na potencjalnie nieaktualnym backbufferze.

Aktualny model:

```text
scene composition
      ↓
final frame
      ↓
cursor overlay
      ↓
present
```

upraszcza spójność klatki.

---

## 24. Z-Order

Kliknięcie może podnieść trafioną warstwę aplikacji w Z-Order.

Warstwa pulpitu:

```text
z_order <= 0
```

nie jest traktowana jak zwykłe okno do podnoszenia ponad aplikacje.

Z-Order, focus i capture pozostają osobnymi stanami logicznymi.

---

## 25. Popupy i pacing

Nowsza architektura GUI wprowadziła top-level popup surfaces.

Dotyczy to między innymi:

- menu,
- dialogów,
- tooltipów,
- toastów.

Popupy mogą posiadać własne:

- bounds,
- damage starego i nowego obszaru,
- cień,
- lifetime,
- powiązanie z generacją ekranu.

Pacing nadal obowiązuje dla końcowej prezentacji klatki, ale zmiana popupu musi zachować informację o tym, które obszary sceny wymagają odbudowy.

---

## 26. Generacja ekranu

Po zmianie fizycznej rozdzielczości zasoby zależne od framebufferu muszą odpowiadać aktualnej generacji ekranu.

Nowszy model popupów przechowuje:

```text
generacja_ekranu
```

i odrzuca powierzchnię, jeżeli:

```text
popup.generation != active_display.generation
```

Chroni to przed użyciem powierzchni utworzonej dla starego framebufferu po zmianie trybu.

Mechanizm generacji jest związany z cyklem życia grafiki, a nie z samym pomiarem HPET.

---

## 27. Zmiana rozdzielczości a pacing

Zmiana trybu fizycznego nie jest zwykłą animacją i nie powinna być sztucznie blokowana przez normalny 60 Hz pacing.

Operacja ekranu posiada własny model:

```text
prepare new buffers
      ↓
prepare wallpaper
      ↓
krótki lock
      ↓
hardware mode switch
      ↓
atomic graphics/compositor state swap
      ↓
nowa generacja
      ↓
spójna klatka
      ↓
unlock
      ↓
release old resources
```

Kosztowne przygotowanie buforów powinno odbywać się poza spinlockiem ekranu.

---

## 28. Dedykowana pula PMM/VMM grafiki

Duże bufory graficzne są alokowane z dedykowanej puli PMM/VMM, a nie ze standardowej małej sterty jądra.

Historyczny audyt ekranu potwierdzał dla swojej wersji kodu, że powtarzane przejścia:

```text
1280x800
→ 1920x1080
→ 2560x1440
→ 1920x1080
→ 1280x800
```

nie zwiększały użycia PMM, heap ani graphics-buffer pool.

Ten wynik należy traktować jako wynik **konkretnej wersji audytowanej**, a nie wieczystą gwarancję wszystkich późniejszych zmian.

---

## 29. Aktualny status regresji grafiki

Po późniejszych zmianach popup/compositor/display-generation w bieżącym cyklu rozwoju pojawił się log:

```text
[EKRAN] BLAD puli PMM/VMM
```

Jest to aktualnie traktowane jako regresja cyklu życia/alokacji grafiki wymagająca naprawy.

Nie ma dowodu, że problem wynika z HPET, kalibracji LAPIC ani samego pacingu 60 Hz.

Podejrzane są przede wszystkim:

- zwalnianie starych powierzchni po zmianie generacji,
- częściowe alokacje/rollback,
- backing surfaces popupów,
- accounting dedykowanej puli grafiki.

Po naprawie trzeba ponownie wykonać testy leak-free transitions.

---

## 30. Interakcja z BWS 62

BWS 62 zwraca informacje o aktualnym monitorze/obszarze roboczym/skali.

W aktualnej architekturze fizyczna rozdzielczość i skala UI to dwie różne wartości.

Pacing opiera się na czasie, a nie na współczynniku skalowania.

Zmiana:

```text
100% → 125% → 150% → 175% → 200%
```

nie powinna zmieniać częstotliwości LAPIC ani HPET.

---

## 31. Diagnostyka wydajności

Makro:

```text
BURSZTYN_PERF_TIMER
```

kontroluje okresowe logi `[PERF]`.

Konfiguracja:

```text
BURSZTYN_PERF_TIMER=0
```

wyłącza je podczas kompilacji.

W normalnym buildzie nie należy generować ciągłego spamu timerowego.

Diagnostyka może być przydatna do pomiaru:

- ticków LAPIC,
- częstotliwości kompozycji,
- czasu między klatkami,
- opóźnień,
- liczby pending dirty requests.

---

## 32. Jakich logów nie należy mieszać

Ważne jest rozdzielenie źródła problemu.

### Timer / HPET

```text
TIMER-WARN
HPET init failure
LAPIC calibration failure
```

### Grafika / PMM/VMM

```text
[EKRAN] BLAD puli PMM/VMM
```

### Kompozytor / popup

```text
popup generation mismatch
stale surface
damage/lifetime diagnostics
```

Wystąpienie błędu grafiki po zmianie rozdzielczości nie powinno automatycznie prowadzić do modyfikowania częstotliwości LAPIC lub pacingu bez dowodu przyczynowego.

---

## 33. Testy HPET

Minimalny test podsystemu HPET powinien obejmować:

```text
ACPI old
ACPI new
bad RSDP checksum
bad RSDT/XSDT checksum
missing HPET table
wrong GAS space
zero address
non-zero bit offset
32-bit counter wrap
monotonic ordering
```

Dla poprawnej ścieżki:

```text
t1 <= t2 <= t3
```

musi być zachowane dla kolejnych odczytów monotonicznych.

---

## 34. Testy kalibracji LAPIC

Należy sprawdzać:

```text
HPET available
→ calibration success
→ 250 Hz target

HPET unavailable
→ TIMER-WARN
→ deterministic fallback
```

Pomiar powinien weryfikować rzeczywisty odstęp timer IRQ w rozsądnej tolerancji, a nie tylko wartość wpisaną do rejestru LAPIC.

---

## 35. Testy pacingu GUI

Minimalna macierz:

```text
idle desktop
continuous mouse movement
window drag
rapid control updates
popup open/close
toast notifications
resolution change
UI scale change
```

Oczekiwania:

- brak pełnego renderowania w IRQ,
- brak > nominalnego limitu pełnych klatek wynikającego z pacingu,
- dirty nie ginie przed deadline,
- brak ciągłego renderowania bez zmian,
- kursor pozostaje responsywny,
- timer pulpitu działa bez ruchu myszy.

---

## 36. Testy integracyjne po zmianie rozdzielczości

Po naprawie bieżącej regresji PMM/VMM należy ponownie wykonać:

```text
1280x720
→ 1920x1080
→ 2560x1440
→ 1920x1080
→ 1280x720
```

w wielu cyklach.

Po powrocie do stanu początkowego:

```text
PMM delta                = 0
VMM mapped-page delta    = 0
graphics buffer delta    = 0
stale popup count        = 0
```

oraz nie powinien wystąpić:

```text
[EKRAN] BLAD puli PMM/VMM
```

Test grafiki jest osobny od testu częstotliwości HPET/LAPIC, ale oba podsystemy spotykają się w runtime GUI.

---

## 37. Ograniczenia aktualnej architektury

Aktualnie pozostają następujące ograniczenia:

- system jest jednordzeniowy,
- scheduler pozostaje Round-Robin,
- pełny async I/O nie istnieje,
- długie synchroniczne operacje Ring 0 mogą nadal wpływać na responsywność,
- bazowy compositor nie jest jeszcze kompletnym tile/damage compositor,
- nie każdy backend grafiki obsługuje runtime mode switching,
- bieżący kod GUI jest nadal stabilizowany przy wysokich rozdzielczościach i popupach.

Nie są to ograniczenia samego HPET, lecz całego środowiska wykonawczego, w którym mechanizm czasu działa.

---

## 38. Zasady architektoniczne

Aktualny podsystem powinien zachowywać następujące reguły:

1. HPET jest monotonicznym źródłem czasu.
2. RTC służy do czasu kalendarzowego.
3. LAPIC jest kalibrowany przed deklarowaniem częstotliwości 250 Hz.
4. Brak HPET musi być jawnie raportowany.
5. Ring 3 nie otrzymuje MMIO HPET.
6. BWS 60 udostępnia monotoniczny czas, nie sprzęt.
7. Timer IRQ nie renderuje GUI.
8. Pending compositor work wykonuje się poza IRQ.
9. Dirty request nie ginie przed deadline.
10. 60 Hz jest maksymalnym pacingiem pełnej klatki, nie obowiązkowym render loop.
11. Kursor jest overlayem końcowego etapu kompozycji.
12. Zegar pulpitu jest wybudzany timerem, a nie ruchem myszy.
13. Pacing nie może maskować błędów lifecycle grafiki.
14. Błąd PMM/VMM grafiki musi być naprawiany w warstwie własności/alokacji, a nie przez zmianę timera.
15. Przyszłe SMP będzie wymagało ponownego audytu współbieżności clockeventów i kompozytora.

---

## 39. Najważniejsze zmiany względem pierwotnej wersji dokumentu

Pierwotny dokument był już w większości zgodny z aktualnym modelem HPET/LAPIC. Aktualizacja rozszerza go jednak o obecny stan całego systemu.

Dodano i doprecyzowano:

- rolę BWS 60,
- rozdzielenie HPET/LAPIC/RTC,
- znaczenie okresu 250 Hz,
- ograniczenie context switcha podczas synchronicznego Ring 0,
- różnicę między globalnym dirty a nowszym lokalnym damage popupów,
- top-level popup surfaces,
- generację ekranu,
- interakcję ze zmianą rozdzielczości,
- dedykowaną pulę PMM/VMM grafiki,
- aktualną regresję `[EKRAN] BLAD puli PMM/VMM`,
- testy wymagane po jej naprawie,
- dokładniejsze zasady diagnostyki.

Nie zmieniono podstawowych parametrów potwierdzonych przez obecny audyt HPET:

```text
LAPIC divider = 16
calibration window = 20 ms
calibration start count = 0xffffffff
scheduler target = 250 Hz
GUI full-frame pacing = 16 666 667 ns
desktop timer ≈ 1 s
```

---

## 40. Dokumenty powiązane

Aktualny stan należy czytać razem z:

- `04_audyt_gui_schedulera.md` — historyczne problemy wejścia/schedulera i model event queues,
- `04_zarzadzanie_sprzetem.md` — ACPI, HPET, APIC, RTC i IRQ,
- `11_audyt_bws_nowoczesne.md` — BWS 60 i współczesne Event ABI,
- `ustawienia_ekranu_audyt.md` — PMM/VMM graphics buffers, mode transitions i display lock,
- `09_tryb_graficzny.md` — backendy grafiki i kompozytor.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod HPET/LAPIC,
2. aktualny kod schedulera i kompozytora,
3. bieżące logi runtime,
4. najnowszy audyt danego podsystemu,
5. starsza dokumentacja historyczna.
