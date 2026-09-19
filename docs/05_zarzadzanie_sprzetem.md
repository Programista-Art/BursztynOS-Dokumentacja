# 04. Zarządzanie Sprzętem i Inicjalizacja Podsystemów Procesora

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** GDT/TSS, IDT i wyjątki, PIC/APIC, HPET/LAPIC, RTC, PS/2, USB xHCI/HID, PCI, AHCI, E1000, HDA oraz zasady obsługi IRQ/MMIO  
**Status:** aktualny opis architektury sprzętowej; zastępuje wcześniejszą wersję ograniczoną głównie do PS/2 i niekalibrowanego LAPIC Timera

---

## 4.1 Cel warstwy sprzętowej

Warstwa sprzętowa Bursztyn OS odpowiada za przejęcie kontroli nad platformą x86-64 po zakończeniu bootstrapu i udostępnienie pozostałym podsystemom spójnych, kontrolowanych interfejsów.

Do jej zadań należą między innymi:

- konfiguracja GDT i TSS,
- inicjalizacja IDT,
- obsługa wyjątków procesora,
- wyłączenie legacy PIC i uruchomienie APIC,
- inicjalizacja HPET i LAPIC Timera,
- odczyt RTC CMOS,
- enumeracja PCI,
- obsługa SATA AHCI,
- obsługa USB xHCI/HID,
- obsługa Intel E1000,
- obsługa Intel HDA,
- przygotowanie urządzeń wejścia dla Event ABI,
- mapowanie MMIO przez VMM.

Architektura pozostaje obecnie **jednordzeniowa**. Nie należy opisywać jej jako pełnego SMP.

---

# 4.2 GDT64

W Long Mode segmentacja nie pełni już roli podstawowego mechanizmu izolacji pamięci. Ochronę przestrzeni adresowych zapewnia przede wszystkim VMM i stronicowanie.

GDT pozostaje jednak potrzebna do:

- rozróżnienia Ring 0 i Ring 3,
- konfiguracji selektorów używanych przez `SYSCALL/SYSRET`,
- załadowania TSS,
- poprawnych przejść poziomów uprzywilejowania.

Aktualny model logiczny:

| Indeks | Selektor | Typ | Ring | Przeznaczenie |
|---:|---:|---|---:|---|
| 0 | `0x00` | Null | — | wymagany deskryptor zerowy |
| 1 | `0x08` | Kod jądra | 0 | 64-bitowy kod Ring 0 |
| 2 | `0x10` | Dane jądra | 0 | dane i stosy Ring 0 |
| 3 | `0x1B` | Kod użytkownika | 3 | programy `.bur` |
| 4 | `0x23` | Dane użytkownika | 3 | dane i stos Ring 3 |
| 5 | `0x28` | TSS | 0 | deskryptor systemowy TSS |

Wartości selektorów powinny być traktowane jako część bieżącej implementacji kernela i weryfikowane względem aktualnej GDT.

---

# 4.3 TSS — korekta starszego opisu

Starsza wersja dokumentu mówiła:

> „Jądro alokuje 16-bajtowy segment TSS.”

To sformułowanie jest nieprecyzyjne.

W GDT **deskryptor TSS** w x86-64 ma 16 bajtów, ale właściwa **struktura TSS** jest osobnym, większym obiektem pamięciowym.

TSS przechowuje między innymi:

- `RSP0`,
- wskaźniki IST,
- pola wymagane przez architekturę x86-64.

Jądro ładuje TSS instrukcją:

```asm
ltr
```

z odpowiednim selektorem, np. `0x28`.

---

# 4.4 TSS, przerwania i SYSCALL

Krytyczna korekta względem starej dokumentacji:

> `SYSCALL` w x86-64 nie przełącza automatycznie `RSP` na `TSS.RSP0`.

Przy wejściu przez `SYSCALL` procesor:

- zmienia poziom wykonania zgodnie z MSR,
- ładuje RIP z `IA32_LSTAR`,
- zapisuje adres powrotu w `RCX`,
- zapisuje `RFLAGS` w `R11`,

ale przełączenie na właściwy stos Ring 0 musi wykonać kod wejściowy BWS.

TSS jest natomiast bezpośrednio istotny dla przejść Ring 3 → Ring 0 przez przerwania i wyjątki oraz dla mechanizmu IST.

---

# 4.5 Stosy Ring 0 i IST

Aktualny model bezpieczeństwa stosów obejmuje:

- oddzielone stosy Ring 0 dla kontekstów procesów,
- stos rzędu 64 KiB w wydzielonym slocie przestrzeni wirtualnej,
- guard page,
- kontrolę integralności stosu,
- dedykowany stos IST dla krytycznych wyjątków, w szczególności `#DF`.

Guard page ma zatrzymać rosnący stos przez `#PF`, zanim zapis przejdzie do sąsiedniego obszaru pamięci.

IST zwiększa szansę poprawnej diagnostyki awarii, gdy zwykły stos jądra jest uszkodzony.

---

# 4.6 IDT

Bursztyn OS używa 256-elementowej IDT dla x86-64.

Typowa bramka 64-bitowa ma 16 bajtów i przechowuje adres procedury ISR/IRQ oraz atrybuty bramki.

Wyjątki procesora zajmują wektory:

```text
0–31
```

Przykłady:

```text
#UD — Invalid Opcode
#DF — Double Fault
#GP — General Protection
#PF — Page Fault
```

Procedury wejściowe assemblera budują kontrolowaną ramę stanu i przekazują ją do kodu C/C++.

---

# 4.7 Obsługa wyjątków i Błąd Krytyczny Systemu

Podsystem wyjątków może prezentować diagnostykę zawierającą między innymi:

- numer wektora,
- kod błędu,
- RIP,
- stan istotnych rejestrów,
- informacje dotyczące Page Fault.

Bursztyn OS posiada graficzną ścieżkę prezentacji błędu krytycznego.

Nie należy jednak zakładać, że **każdy błąd Ring 3 musi kończyć cały system**.

Docelowa polityka powinna rozróżniać:

```text
błąd procesu Ring 3
```

od:

```text
krytyczny błąd Ring 0
```

Tam, gdzie stan jądra pozostaje bezpieczny, awaria aplikacji powinna prowadzić do zakończenia procesu i cleanupu jego zasobów zamiast globalnego zatrzymania systemu.

---

# 4.8 Legacy PIC

Po przejęciu kontroli nad przerwaniami Bursztyn OS maskuje klasyczne kontrolery 8259 PIC.

Porty:

```text
0x21
0xA1
```

otrzymują maskę blokującą legacy IRQ obsługiwane przez stary PIC.

Celem jest uniknięcie jednoczesnego routingu tego samego źródła przez PIC i APIC.

---

# 4.9 Local APIC

Local APIC jest podstawą współczesnej obsługi lokalnych przerwań i timera procesora w Bursztyn OS.

Adres bazowy APIC jest pobierany z:

```text
IA32_APIC_BASE
```

zamiast bezwarunkowo zakładać stały adres.

Na typowej platformie x86 adres często wynosi:

```text
0xFEE00000
```

ale implementacja powinna opierać się na wartości sprzętowej.

MMIO APIC musi być mapowane przez VMM z atrybutami odpowiednimi dla rejestrów urządzenia, a nie jak zwykły cache'owany RAM.

APIC posiada sprzętowy bit enable w `IA32_APIC_BASE`, a dodatkowe programowe włączenie Local APIC odbywa się przez Spurious Interrupt Vector Register.

---

# 4.10 EOI

Starszy audyt GUI potwierdził, że EOI jest wysyłane centralnie w warstwie IDT/APIC.

Sterowniki PS/2 nie powinny wysyłać drugiego EOI dla tego samego IRQ.

Zasada:

```text
obsługa jednego IRQ
      ↓
jedno kontrolowane EOI
```

Centralizacja EOI zmniejsza ryzyko:

- podwójnego potwierdzenia,
- zakończenia przerwania w niewłaściwym miejscu,
- niespójności między sterownikami.

---

# 4.11 HPET

Współczesna architektura czasu Bursztyn OS używa **HPET — High Precision Event Timer**.

HPET jest wykrywany przez ACPI.

Parser:

- obsługuje dane ACPI przekazane przez Multiboot2,
- waliduje RSDP,
- waliduje checksum RSDT/XSDT,
- waliduje odczytywane SDT,
- akceptuje tabelę HPET tylko przy prawidłowym GAS `System Memory`,
- wymaga niezerowego adresu,
- nie korzysta ze stałego, wpisanego na sztywno adresu HPET.

HPET jest mapowany przez VMM jako MMIO.

Aktualna konfiguracja używa main counter jako monotonicznego źródła czasu bez konieczności generowania IRQ przez comparatory.

---

# 4.12 Monotoniczny czas HPET

Pole `COUNTER_CLK_PERIOD` służy do wyznaczenia częstotliwości licznika.

API monotoniczne przelicza ticki na jednostki czasu bez używania floating point.

BWS 60 udostępnia Ring 3:

```text
monotonic HPET nanoseconds
```

lub zero, jeśli HPET nie jest dostępny.

Dla 32-bitowego main countera jądro posiada programowe rozszerzanie epoki przy wrap.

Monotoniczny czas powinien być używany do pomiaru przedziałów. RTC nie zastępuje go w tej roli.

---

# 4.13 Kalibracja LAPIC Timera

Starszy dokument opisywał stały, arbitralny `initial count`. Ten model jest nieaktualny.

Aktualna inicjalizacja:

1. zachowuje divider LAPIC równy 16;
2. uruchamia timer od:

```text
0xffffffff
```

3. mierzy **20 ms** przy użyciu HPET;
4. odczytuje spadek `current count`;
5. oblicza realną częstotliwość LAPIC po dividerze;
6. wyznacza periodic initial count dla:

```text
250 Hz
```

Dzięki temu deklaracja 250 Hz jest oparta na pomiarze, a nie arbitralnej stałej.

Jeżeli HPET nie jest dostępny, działa jawny fallback oznaczony logiem:

```text
TIMER-WARN
```

---

# 4.14 LAPIC Timer i scheduler

LAPIC Timer jest clockeventem schedulera.

Projekt pozostaje obecnie jednordzeniowy i używa planisty Round-Robin.

Ważna reguła:

```text
timer IRQ
→ minimalna praca ISR
→ scheduler / event state
→ EOI
→ cięższa praca poza IRQ
```

Kompozytor nie renderuje pełnej klatki bezpośrednio w IRQ.

PID 0 uczestniczy w zwykłym Round-Robin i może wykonywać oczekującą pracę kompozytora poza przerwaniem.

---

# 4.15 Pacing GUI

HPET jest również wykorzystywany do ograniczania częstotliwości pełnej kompozycji.

Aktualny limit odpowiada okresowi:

```text
16 666 667 ns
```

czyli około 60 Hz.

Żądanie `dirty` pozostaje aktywne do terminu kolejnej możliwej kompozycji.

Jeżeli nie ma zmian, pełna klatka nie jest generowana tylko dlatego, że upłynął kolejny okres.

---

# 4.16 RTC CMOS

RTC CMOS pozostaje źródłem czasu kalendarzowego.

Dostęp odbywa się przez porty:

```text
0x70
0x71
```

RTC służy między innymi do prezentowania użytkownikowi:

- godzin,
- minut,
- sekund,
- danych kalendarzowych.

Kod powinien uwzględniać format danych RTC, w tym możliwy BCD i sposób konfiguracji 12/24 h zależnie od rejestrów CMOS.

RTC nie jest używany jako precyzyjny clockevent schedulera.

Aktualny model:

```text
HPET  → monotoniczny czas / kalibracja
LAPIC → periodyczny clockevent schedulera
RTC   → czas kalendarzowy
```

---

# 4.17 Aktualizacja zegara pulpitu

Starszy model odczytywał RTC w ścieżce zależnej od pollingu myszy.

To zostało zmienione.

Aktualnie timer systemowy dostarcza pulpitowi scalany event około raz na sekundę zgodnie z czasem HPET.

Menedżer Okien:

1. odbiera event,
2. odczytuje RTC,
3. porównuje czas,
4. odświeża zegar tylko przy zmianie wartości.

Dzięki temu zegar pulpitu nie zależy od ruchu wskaźnika.

---

# 4.18 PCI

PCI jest podstawową warstwą wykrywania wielu urządzeń Bursztyn OS.

System wykorzystuje klasy i identyfikatory PCI do rozpoznawania między innymi:

- kontrolera SATA AHCI,
- Intel E1000,
- urządzeń audio,
- kontrolera USB,
- innych urządzeń raportowanych diagnostycznie.

Samo wykrycie klasy urządzenia nie oznacza istnienia działającego sterownika.

Przykład:

> NVMe może być widoczne w enumeracji PCI, ale aktualny Bursztyn OS nie posiada jeszcze działającego backendu blokowego NVMe.

---

# 4.19 AHCI SATA

Bursztyn OS posiada własny sterownik SATA AHCI.

Aktualna warstwa obsługuje:

- kontroler SATA AHCI,
- port ATA,
- DMA,
- LBA48,
- IDENTIFY,
- odczyt i zapis sektorów,
- ATA FLUSH używany w krytycznych ścieżkach instalatora.

Aktualny instalator obsługuje dyski z logicznym sektorem:

```text
512 B
```

w tym urządzenia:

```text
512e
```

Dyski 4Kn są obecnie odrzucane przez instalator.

---

# 4.20 Ograniczenia storage

Aktualnie nie istnieje działający backend blokowy dla:

- NVMe,
- IDE/ATA PIO jako osobnego kontrolera,
- USB Mass Storage,
- Intel RST,
- Intel VMD,
- trybów RAID.

USB w aktualnym jądrze obsługuje HID, ale nie posiada jeszcze warstwy:

```text
BOT/UAS
+
SCSI
+
block device
```

potrzebnej do użycia pamięci USB jako dysku systemowego.

---

# 4.21 USB xHCI

Bursztyn OS posiada natywny stos USB oparty na **xHCI**.

Jest on obecnie używany przede wszystkim do urządzeń HID.

Obsługiwane ścieżki obejmują:

- klawiaturę USB,
- mysz USB,
- podstawowe boot-HID reports,
- część raportów koła myszy.

Warstwa USB nie jest jeszcze uniwersalnym stosem wszystkich klas urządzeń.

---

# 4.22 Klawiatura — PS/2 i USB HID

Stary dokument opisywał wyłącznie PS/2.

Aktualnie Bursztyn OS może otrzymywać wejście klawiatury również przez USB HID.

Nowoczesny Event ABI rozdziela:

```text
fizyczne zdarzenie klawiatury
```

od:

```text
Unicode TEXT_INPUT
```

Pozwala to niezależnie obsługiwać:

- skróty,
- nawigację,
- klawisze funkcyjne,
- wpisywany tekst.

Event ABI posiada również maskę modyfikatorów, np.:

```text
SHIFT
CTRL
ALT
SUPER
CAPS
NUM
```

Pełne layouty, dead keys i IME pozostają przyszłą warstwą systemu.

---

# 4.23 Klawiatura PS/2

Sterownik i8042 pozostaje starszą ścieżką wejścia.

Dla klasycznego IRQ klawiatury dane są odczytywane z:

```text
0x60
```

i dekodowane z formatu scancode.

Nie należy już opisywać całej ścieżki jako:

```text
scancode → ASCII → aplikacja
```

ponieważ aktualny model posiada oddzielne zdarzenia fizycznego klawisza i Unicode text input.

---

# 4.24 Mysz PS/2

Sterownik PS/2 obsługuje klasyczny pakiet relatywnego ruchu i przyciski.

Aktualny model routingu GUI nie polega już na prostym globalnym stanie myszy.

Zdarzenie przechodzi przez:

```text
IRQ
 ↓
dekodowanie ruchu/przycisków
 ↓
hit-test / capture
 ↓
docelowy PID
 ↓
bounded event queue
```

Kliknięcie może ustawić focus najwyższej trafionej warstwy.

---

# 4.25 USB HID mouse

Mysz USB działa przez xHCI/HID.

Aktualne wsparcie obejmuje:

- relatywne `dx/dy`,
- przyciski,
- signed vertical/horizontal wheel dla obsługiwanych raportów,
- routing do systemowego Event ABI.

Obecne boot-HID i PS/2 nadal używają głównie 8-bitowych delta ruchu.

Pełne dekodowanie dowolnych HID report descriptors i prawdziwych high-resolution pointer reports pozostaje przyszłą pracą.

---

# 4.26 Event ABI wejścia

Nowoczesny model wejścia zawiera między innymi:

| Event | Funkcja |
|---:|---|
| 23 | fizyczne zdarzenie klawiatury |
| 24 | Unicode `TEXT_INPUT` |
| 25 | pionowe / poziome koło myszy |
| 26 | jawne zdarzenie wskaźnika wysokiej precyzji |

Struktura `bws_zdarzenie` zachowuje rozmiar:

```text
40 B
```

Ruch wskaźnika może być scalany, aby nie przepełniać kolejki przy bardzo wysokiej częstotliwości raportów.

---

# 4.27 Focus i capture

Sterowniki wejścia nie powinny decydować bezpośrednio o zachowaniu kontrolek aplikacji.

Po znormalizowaniu eventu podsystem GUI wybiera docelowy PID.

Model wykorzystuje:

- hit-test,
- focus,
- capture,
- per-process event queues.

Podczas drag operacji capture ma pierwszeństwo przed zwykłym hit-testem.

Capture musi być usuwany przy:

- zakończeniu interakcji,
- zamknięciu okna,
- zakończeniu procesu,
- błędzie/rollbacku operacji modalnej.

---

# 4.28 Intel E1000

Bursztyn OS posiada własny sterownik Intel E1000 wykorzystywany przez natywny stos sieciowy.

Na jego podstawie działają obecnie między innymi:

- Ethernet,
- ARP,
- IPv4,
- ICMP,
- UDP,
- DHCP,
- DNS,
- TCP,
- HTTP,
- HTTPS/TLS.

Sterownik sieciowy pracuje w Ring 0.

Należy odróżnić działanie samego E1000 od ograniczeń wyższej warstwy TCP. Aktualny stos bazowy nadal obsługuje tylko jedną systemową sesję TCP jednocześnie.

---

# 4.29 Intel HDA

Starsza roadmapa projektu zakładała AC97. Ten kierunek jest nieaktualny.

Bursztyn OS posiada obecnie sterownik **Intel HDA**.

Aktualny tor audio jest jednak ograniczony.

Udokumentowany stan:

```text
48 kHz
S16
stereo
```

oraz BWS 27:

```text
HDA test tone
```

Brakuje jeszcze pełnego publicznego API:

- owned audio streams,
- negocjacji formatów,
- zapisu ramek przez aplikacje,
- zapytania o wolne miejsce,
- systemowego miksowania.

---

# 4.30 Grafika jako urządzenie sprzętowe

Warstwa grafiki posiada trzy backendy:

- UEFI GOP,
- VESA VBE,
- Bochs VBE.

GOP i VESA dostarczają przede wszystkim framebuffer startowy.

Bochs VBE w środowisku QEMU pozwala dodatkowo przełączać tryby podczas działania systemu.

Niskopoziomowy sterownik grafiki nie powinien być mylony z BursztynUI ani Menedżerem Okien. Są to wyższe warstwy.

Schemat:

```text
sprzęt / framebuffer
      ↓
HAL grafiki
      ↓
bufory PMM/VMM
      ↓
kompozytor
      ↓
Menedżer Okien Ring 3
      ↓
BursztynUI
```

---

# 4.31 MMIO i VMM

Urządzenia takie jak:

- APIC,
- HPET,
- kontrolery PCI,
- framebuffer

nie powinny być obsługiwane przez przypadkowe dereferencje surowych adresów fizycznych.

Dostęp MMIO powinien przechodzić przez kontrolowane mapowania VMM z odpowiednimi atrybutami pamięci.

Dla HPET aktualna implementacja używa mapowania:

```text
supervisor
writable
PWT
PCD
```

Zasada architektoniczna:

> pamięć urządzenia nie jest zwykłym cache'owalnym RAM.

---

# 4.32 IRQ — minimalna praca w przerwaniu

Kod ISR powinien wykonywać możliwie mało pracy.

Preferowany model:

```text
IRQ
 ↓
odczytaj / potwierdź urządzenie
 ↓
zapisz mały event / stan
 ↓
EOI
 ↓
cięższa praca poza IRQ
```

Do IRQ nie powinny trafiać:

- pełna kompozycja pulpitu,
- długie dekodowanie,
- synchroniczny networking,
- kosztowne alokacje,
- długie logowanie diagnostyczne.

---

# 4.33 Scheduler a długie operacje Ring 0

Aktualny timer jest poprawnie kalibrowany, ale pozostaje inny problem.

Synchroniczne:

- HTTP,
- HTTPS,
- TLS

mogą wykonywać dłuższe pętle pracy w Ring 0.

Timer w tym czasie może obsługiwać IRQ, ale nie należy wykonywać nieprzemyślanego context switcha w środku dowolnej ramy wejścia `SYSCALL`.

Docelowo długie I/O powinno używać modelu:

```text
request
 ↓
WAITING_IO
 ↓
worker / state machine
 ↓
completion event
 ↓
wake process
```

Pełny async I/O nie jest jeszcze zaimplementowany.

---

# 4.34 Kolejność inicjalizacji podsystemów

Dokładna kolejność powinna być zgodna z zależnościami aktualnego kodu, ale architektonicznie obowiązuje model zbliżony do:

```text
Long Mode
  ↓
wczesny kernel_main
  ↓
PMM / VMM
  ↓
pełna GDT / TSS
  ↓
IDT / wyjątki
  ↓
ACPI
  ↓
APIC
  ↓
HPET
  ↓
kalibracja LAPIC
  ↓
scheduler
  ↓
PCI
  ↓
AHCI / storage
  ↓
xHCI / HID
  ↓
E1000
  ↓
HDA
  ↓
grafika / kompozytor
  ↓
BWS
  ↓
Ring 3
```

To jest model dokumentacyjny, nie gwarantowany zapis kolejności każdej pojedynczej funkcji w `kernel.cpp`.

---

# 4.35 Aktualne ograniczenia sprzętowe

Na obecnym etapie projekt nie powinien deklarować uniwersalnej zgodności z komputerami x86-64.

Istotne ograniczenia:

- brak SMP,
- brak sterownika NVMe,
- brak USB Mass Storage,
- brak 4Kn w instalatorze,
- brak Intel RST/VMD/RAID,
- brak pełnego PnP,
- ograniczony power management,
- brak Secure Boot,
- ograniczona liczba kart sieciowych,
- ograniczony zestaw układów audio,
- brak sprzętowej akceleracji GPU,
- częściowe HID report descriptor support.

Głównym środowiskiem testowym pozostają QEMU/Bochs oraz wybrane scenariusze sprzętowe.

---

# 4.36 Zasady bezpiecznego kodu sterowników

Sterowniki Ring 0 powiększają Trusted Computing Base, dlatego powinny przestrzegać szczególnie rygorystycznych reguł.

### MMIO

- walidować rozmiary BAR,
- nie dereferencjonować niezweryfikowanych adresów,
- używać właściwych mapowań VMM.

### DMA

- jawnie określać właściciela bufora,
- pilnować alignment,
- sprawdzać długości,
- nie przekazywać urządzeniu przypadkowego adresu Ring 3.

### IRQ

- minimalizować czas w ISR,
- nie wykonywać dużych alokacji,
- unikać ciężkiego logowania,
- nie wysyłać podwójnego EOI.

### Bufory

- używać `size_t` / typów bez znaku dla rozmiarów,
- sprawdzać overflow,
- nie ufać długościom raportowanym przez sprzęt.

### Timeout

Polling sprzętu musi mieć ograniczenie czasu/liczby iteracji.

Nigdy:

```cpp
while (!device_ready()) {
}
```

bez mechanizmu timeout.

---

# 4.37 Testy podsystemów sprzętowych

Minimalna macierz testowa powinna obejmować:

### CPU / wyjątki

```text
#UD
#GP
#PF Ring 3
#PF Ring 0
#DF / IST path
```

### Timery

```text
HPET detected
HPET absent → TIMER-WARN fallback
monotonic ordering
LAPIC calibration
250 Hz tolerance
```

### Input

```text
PS/2 keyboard
PS/2 mouse
USB HID keyboard
USB HID mouse
mouse wheel
modifier mask
TEXT_INPUT
```

### Storage

```text
AHCI IDENTIFY
read
write
flush
out-of-range
I/O error propagation
```

### USB

```text
xHCI enumeration
keyboard reconnect
mouse reconnect
unsupported device class
```

### Sieć

```text
E1000 initialization
ARP
DHCP
ICMP
UDP
TCP
```

### Audio

```text
HDA init
test tone
failure path
```

---

# 4.38 Różnice względem starszej wersji dokumentu

Najważniejsze korekty:

1. **TSS**  
   16 bajtów dotyczy deskryptora TSS w GDT, a nie całej struktury TSS.

2. **SYSCALL**  
   TSS nie przełącza automatycznie `RSP` przy `SYSCALL`.

3. **LAPIC Timer**  
   Nie używa już wyłącznie arbitralnego initial count. Jest kalibrowany przez HPET na oknie 20 ms i ustawiany na 250 Hz.

4. **Czas**  
   HPET jest źródłem monotonicznym; RTC pozostaje zegarem kalendarzowym.

5. **Wejście**  
   System nie jest już wyłącznie PS/2. Posiada xHCI i USB HID.

6. **Klawiatura**  
   Aktualny Event ABI rozdziela physical-key i `TEXT_INPUT`.

7. **Mysz**  
   Dodano wheel event oraz model hit-test/focus/capture/per-process queue.

8. **Audio**  
   Aktualnym urządzeniem jest Intel HDA, a nie planowany AC97.

9. **Storage**  
   AHCI działa, natomiast NVMe, 4Kn, USB Mass Storage i RST/VMD pozostają ograniczeniami.

10. **IRQ**  
    EOI jest centralizowane, a ciężka praca powinna odbywać się poza ISR.

---

# 4.39 Podsumowanie

Aktualna warstwa sprzętowa Bursztyn OS obejmuje już znacznie więcej niż pierwotna architektura GDT/APIC/PS2.

Obecny stos można uprościć do:

```text
x86-64
├── GDT / TSS
├── IDT / wyjątki / IST
├── Local APIC
├── HPET
├── LAPIC Timer 250 Hz
├── RTC CMOS
├── PCI
├── SATA AHCI
├── USB xHCI
│   └── HID keyboard/mouse
├── Intel E1000
├── Intel HDA
└── framebuffer HAL
    ├── UEFI GOP
    ├── VESA VBE
    └── Bochs VBE
```

Najważniejsze obecne ograniczenia sprzętowe to brak SMP, NVMe, USB Mass Storage, pełnego PnP, zaawansowanego power management oraz sprzętowego GPU.

---

## 4.40 Dokumenty powiązane

Aktualny stan należy interpretować razem z:

- `02_architektura_systemu.md` — Ring 0/Ring 3, TSS i PZB,
- `03_proces_rozruchu.md` — bootstrap i wejście w Long Mode,
- `04_audyt_gui_schedulera.md` — Event ABI, focus/capture i scheduler,
- `05_hpet_i_pacing.md` — aktualny HPET/LAPIC i pacing GUI,
- `09_tryb_graficzny.md` — UEFI GOP, VESA VBE i Bochs VBE,
- `10_siec.md` — E1000 i stos sieciowy,
- `11_audyt_bws_nowoczesne.md` — nowoczesne eventy, kursory i ograniczenia audio/network,
- `instalator_audyt.md` — AHCI, GPT i ograniczenia storage.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod,
2. aktualne testy runtime,
3. najnowszy audyt konkretnego podsystemu,
4. starsze dokumenty opisowe.
