# 01. Wstęp i Filozofia Bursztyn OS

**Stan dokumentu:** aktualizacja 2026-09-18  
**Status projektu:** aktywny rozwój / eksperymentalny system operacyjny  
**Architektura docelowa:** x86-64, własne jądro, własny ABI, własny system plików, własny stos GUI i własny ekosystem aplikacji

---

## 1.1 Cel projektu

**Bursztyn OS** jest niezależnym, polskim systemem operacyjnym rozwijanym od podstaw w modelu bare-metal.

Projekt nie wykorzystuje jądra Linux, BSD ani Windows NT jako bazy. Własne są między innymi:

- jądro systemu,
- zarządzanie pamięcią fizyczną i wirtualną,
- planista i procesy Ring 3,
- Bursztynowe Wywołania Systemowe (BWS),
- model bezpieczeństwa PZB,
- system plików BSP2/PSF,
- sterowniki sprzętowe,
- stos sieciowy,
- środowisko graficzne i kompozytor,
- framework BursztynUI,
- format programów `.bur`,
- paczki aplikacji `.cebula`,
- instalator,
- mechanizm aktualizacji A/B.

Celem projektu nie jest stworzenie kolejnej dystrybucji istniejącego systemu, lecz poznanie i zaprojektowanie całego stosu operacyjnego — od uruchomienia procesora w Long Mode, przez pamięć, sterowniki, ABI i system plików, aż po aplikacje Ring 3 i własny interfejs graficzny.

Bursztyn OS jest obecnie projektem **edukacyjnym, eksperymentalnym i badawczo-inżynierskim**, rozwijanym z myślą o stopniowym przejściu w kierunku realnie użytkowego systemu desktopowego.

Docelowym elementem ekosystemu ma być również język programowania **Avocado**, generujący natywne programy `.bur` i paczki `.cebula`. Integracja ta pozostaje kierunkiem rozwoju, a nie zakończonym etapem projektu.

---

## 1.2 Tożsamość techniczna i nazewnictwo

Bursztyn OS świadomie buduje własną tożsamość techniczną. Polskie nazewnictwo nie zastępuje precyzji inżynierskiej, lecz stanowi warstwę projektu, która ułatwia tworzenie spójnego, rozpoznawalnego ekosystemu.

Aktualnie najważniejsze filary systemu to:

1. **Jądro Bursztyna**  
   Centralny komponent systemu pracujący w sprzętowym Ring 0. Odpowiada za pamięć, planowanie procesów, obsługę przerwań, sterowniki, BWS, bezpieczeństwo i niskopoziomowe mechanizmy systemowe.

2. **BSP2 / PSF — Bursztynowy System Plików**  
   Autorski system plików używający logicznych bloków 4 KiB, 64-bitowego adresowania bloków oraz wskaźników bezpośrednich i pośrednich. Aktualna ścieżka instalatora korzysta z dysków SATA AHCI i sektorów logicznych 512 B / 512e.

3. **BWS — Bursztynowe Wywołania Systemowe**  
   Własny interfejs ABI pomiędzy Ring 3 i Ring 0, wykonywany przez `SYSCALL`.  
   Aktualne drzewo projektu używa BWS co najmniej do numeru **87**. Dokumentacja ABI jest nadal konsolidowana, ponieważ starsze rozdziały opisują wcześniejsze etapy numeracji.

4. **PZB — Poziom Zaufania Bursztyna**  
   Logiczny model bezpieczeństwa w skali 0–5, współpracujący z 64-bitowymi maskami uprawnień `PRAWO_*`. PZB nie jest dodatkowym ringiem procesora — sprzętowo aplikacje użytkownika pozostają w Ring 3.

5. **BursztynUI**  
   Natywny framework graficzny dla aplikacji Ring 3. Aktualny inwentarz projektu obejmuje około **62 publiczne kontrolki, dialogi i elementy infrastruktury GUI**.

6. **`.bur` — Bursztynowy Program Wykonywalny**  
   Natywny format wykonywalny dla programów Ring 3.

7. **`.cebula` — Paczka Aplikacji**  
   Struktura dystrybucyjna zawierająca program `.bur`, manifest `opis.aplikacji`, ikonę i zasoby.

8. **Aktualizacje A/B**  
   Autorski mechanizm aktualizacji wykorzystujący dwa sloty systemowe, próbny rozruch, health checkpoint i automatyczny rollback.

### Konwencje nazewnicze

W kodzie niskopoziomowym stosowana jest przede wszystkim konwencja:

```text
snake_case
```

bez polskich znaków diakrytycznych, np.:

```text
wezel_indeksowy
menedzer_okien
uzytkownicy
```

Warstwa widoczna dla użytkownika — GUI, Powłoka Bursztynowa, komunikaty, menu i dokumentacja — używa pełnego języka polskiego i UTF-8.

W starszych dokumentach można spotkać nazwę **BZL**. Aktualnym terminem projektu jest **PZB — Poziom Zaufania Bursztyna** i to tej nazwy należy używać w nowych materiałach.

---

## 1.3 Zasady projektowe

Rozwój Bursztyn OS opiera się na kilku zasadach.

### 1.3.1 Własna architektura zamiast nakładki

Bursztyn OS nie ma być:

- dystrybucją Linux,
- środowiskiem uruchomionym na Windows,
- forkowaną wersją BSD,
- nakładką graficzną na gotowy kernel.

Każda główna warstwa systemu ma mieć własny kontrakt i własną implementację.

### 1.3.2 Ring 0 i Ring 3

Kod uprzywilejowany działa w Ring 0, a aplikacje użytkownika w Ring 3.

Aplikacje nie otrzymują bezpośredniego dostępu do:

- portów I/O,
- pamięci jądra,
- tablic stron,
- sprzętu,
- fizycznych adresów zasobów systemowych.

Operacje uprzywilejowane są wykonywane przez kontrolowane BWS.

### 1.3.3 Bezpieczeństwo fail-closed

Jeśli system nie może potwierdzić poprawności operacji, preferowanym zachowaniem jest odmowa wykonania zamiast kontynuowania z niepewnym stanem.

Dotyczy to między innymi:

- walidacji wskaźników Ring 3,
- uchwytów zasobów,
- praw PZB,
- rozmiarów buforów,
- operacji instalatora,
- aktualizacji A/B,
- zmian trybu graficznego.

### 1.3.4 Ograniczone i jawne zasoby

Nowe API powinny mieć jawne limity i jednoznaczne reguły własności.

W nowszych BWS stosowane są m.in.:

- generacyjne uchwyty,
- ograniczone kolejki,
- limity rozmiarów,
- kontrola właściciela per PID,
- cleanup po zakończeniu procesu.

### 1.3.5 Nieudawanie nieistniejących funkcji

Jeśli architektura nie posiada jeszcze pełnej implementacji danego mechanizmu, dokumentacja powinna to zaznaczać.

Przykłady:

- brak pełnego async I/O,
- brak SMP,
- brak pełnego IPv6,
- brak konkurencyjnych wielu sesji TCP,
- brak pełnego publicznego API strumieni audio,
- brak sprzętowej akceleracji GPU.

### 1.3.6 Testy są częścią architektury

Nowa funkcja nie powinna być uznawana za ukończoną tylko dlatego, że kompiluje się lub działa w jednym scenariuszu.

Projekt rozwija:

- testy hostowe,
- testy QEMU,
- testy BWS,
- testy Ring 3,
- testy pamięci,
- testy instalatora,
- testy GUI/BursztynUI,
- testy zmiany rozdzielczości,
- testy aktualizacji A/B.

---

## 1.4 Aktualny stan projektu

Bursztyn OS nie znajduje się jeszcze w stanie „pełnej dojrzałości operacyjnej”. Trafniejsze określenie to:

> **zaawansowany eksperymentalny system operacyjny w aktywnej fazie rozwoju i stabilizacji.**

Poniższe obszary są już realnie zaimplementowane.

### 1.4.1 Rozruch i procesor

System posiada:

- rozruch przez GRUB i Multiboot2,
- przejście do x86-64 Long Mode,
- GDT i TSS,
- IDT,
- APIC,
- HPET,
- RTC,
- własną obsługę wyjątków,
- sprzętową separację Ring 0 / Ring 3.

Hybrydowy obraz ISO może obsługiwać ścieżki BIOS i UEFI, natomiast obecny instalator dyskowy jest projektowany dla UEFI x86-64.

### 1.4.2 Pamięć i planowanie

Zaimplementowano:

- PMM,
- VMM,
- 4-poziomowe stronicowanie x86-64,
- HHDM,
- prywatne przestrzenie procesów,
- stosy jądra chronione mechanizmami bezpieczeństwa,
- SHM,
- planistę Round-Robin,
- LAPIC Timer kalibrowany przez HPET,
- periodic clockevent 250 Hz.

Projekt pozostaje obecnie **jednordzeniowy**. SMP jest przyszłym etapem rozwoju.

### 1.4.3 Ring 3, BWS i procesy

Aplikacje `.bur` działają w Ring 3 i komunikują się z jądrem przez BWS.

Nowsze rozszerzenia obejmują m.in.:

- geometrię własnego okna,
- monitor, work area i skalę GUI,
- SHM,
- IPC,
- obserwowanie zmian plików,
- clipboard MIME,
- kursory systemowe i własne,
- socket facade TCP/IPv4,
- aktualizacje OTA,
- podsystem motywów.

BWS 64–67 implementują SHM z generacyjnymi uchwytami, kontrolą dostępu per PID i limitowanymi rozmiarami.

BWS 68–71 implementują ograniczone IPC endpoints i kolejki komunikatów.

### 1.4.4 System plików i storage

Aktualny BSP2/PSF wykorzystuje:

- bloki 4096 B,
- 64-bitowe adresowanie bloków,
- wskaźniki bezpośrednie,
- bloki pośrednie,
- bloki podwójnie pośrednie.

Warstwa storage obsługuje SATA AHCI.

Obecny instalator:

- używa GPT,
- tworzy ESP FAT32 512 MiB,
- tworzy partycję BSP2 do 2 GiB w bieżącym wydaniu,
- obsługuje sektor logiczny 512 B / 512e,
- odrzuca 4Kn,
- nie obsługuje jeszcze NVMe, Intel RST/VMD/RAID ani USB Mass Storage jako dysku docelowego.

Secure Boot musi być obecnie wyłączony.

### 1.4.5 Grafika i BursztynUI

Warstwa grafiki posiada trzy backendy:

- UEFI GOP,
- VESA VBE,
- Bochs VBE.

System posiada:

- framebuffer,
- backbuffer,
- kompozytor,
- powierzchnie aplikacji,
- Menedżer Okien Ring 3,
- pulpit,
- pasek zadań,
- Menu Start,
- popupy,
- system focus/capture,
- własne kursory,
- powiadomienia,
- system motywów.

BursztynUI zawiera obecnie około 62 publiczne elementy frameworka, w tym:

- podstawowe kontrolki,
- layouty,
- menu,
- tabele i listy,
- kontrolki wirtualizowane,
- `OknoDialogowe`,
- `DialogPliku`,
- `DialogKoloru`,
- `DialogKomunikatu`,
- system `PolecenieUI`,
- popup manager,
- powiadomienia typu toast.

Obsługiwane skale UI:

- 100%,
- 125%,
- 150%,
- 175%,
- 200%.

Dla Bochs VBE system testował tryby aż do 3840×2160 przy odpowiedniej ilości VRAM.

Zmiana rozdzielczości posiada mechanizm potwierdzenia i automatycznego rollbacku.

### 1.4.6 Sieć

Aktualny stos sieciowy zawiera:

- Intel E1000,
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

Dla aplikacji Ring 3 istnieje BWS socket facade TCP/IPv4.

Obecne ograniczenie warstwy TCP:

> stos bazowy pozwala nadal na jedną systemową sesję TCP jednocześnie.

Pełne IPv6 i konkurencyjne sockety pozostają przyszłymi etapami.

### 1.4.7 USB i wejście

Projekt posiada:

- natywny stos USB xHCI,
- klawiaturę USB HID,
- mysz USB HID,
- zdarzenia koła myszy dla obsługiwanych raportów USB,
- wejście PS/2 jako starszą ścieżkę,
- rozdzielenie zdarzeń klawiszy fizycznych i wejścia tekstowego Unicode,
- maskę modyfikatorów klawiatury.

Pełna obsługa dowolnych deskryptorów HID i zaawansowanych urządzeń wejścia pozostaje ograniczona.

### 1.4.8 Audio

Zaimplementowano Intel HDA.

Aktualny tor audio pozostaje ograniczony:

- stałe 48 kHz,
- S16,
- stereo,
- test tone przez BWS,
- brak pełnego publicznego API strumieni,
- brak kompletnego systemowego miksera,
- brak pełnej negocjacji formatów.

Starszy plan użycia AC97 jest już nieaktualny.

### 1.4.9 Instalator

Bursztyn OS posiada graficzny instalator Ring 3.

Aktualna ścieżka instalacji:

```text
UEFI x86-64
    ↓
GPT
    ↓
ESP FAT32
    ↓
BSP2
    ↓
SATA AHCI
```

Instalator stosuje:

- walidację geometrii dysku,
- CRC GPT,
- zapis z flush,
- odczyt kontrolny,
- dwukopiowy stan instalacji z numerem sekwencji i CRC.

Brak obsługiwanego AHCI powoduje bezpieczne zatrzymanie instalacji bez zapisu na dysk.

### 1.4.10 Aktualizacje A/B

System posiada własny model aktualizacji A/B.

Mechanizm obejmuje:

- Slot A,
- Slot B,
- aktywny i nieaktywny slot,
- instalację do nieaktywnego slotu,
- próbny rozruch,
- health checkpoint,
- automatyczne zatwierdzenie,
- rollback przy nieudanym rozruchu,
- SHA-256,
- RSA-2048,
- CRC32,
- generacyjny stan rozruchu,
- współpracę z `grubenv`.

BWS 84 i 85 udostępniają Ring 3 stan oraz polecenia podsystemu OTA.

Autorytatywny stan A/B jest przechowywany w dwóch rekordach LBA 2046 i 2047.

---

## 1.5 Obecne ograniczenia i obszary stabilizacji

Dokumentacja projektu powinna jawnie oddzielać funkcje zaimplementowane od funkcji docelowych.

Na obecnym etapie najważniejsze ograniczenia to:

- brak SMP,
- brak sprzętowej akceleracji GPU,
- brak NVMe jako backendu blokowego,
- brak USB Mass Storage jako docelowego storage instalatora,
- brak 4Kn,
- brak Secure Boot,
- brak swapu i kompresji pamięci,
- brak pełnego IPv6,
- jedna systemowa sesja TCP,
- brak pełnego async I/O,
- brak pełnego API streamów audio,
- brak kompletnego power management laptopowego,
- brak szerokiej kompatybilności z realnym sprzętem,
- ograniczone testy poza QEMU dla części podsystemów.

GUI i grafika pozostają w aktywnej stabilizacji, szczególnie w scenariuszach:

- zmiany rozdzielczości,
- wysokich rozdzielczości,
- popupów,
- dialogów modalnych,
- cyklu życia dużych buforów grafiki PMM/VMM.

Wynik pojedynczego audytu PASS nie oznacza automatycznie, że cały system jest produkcyjnie stabilny.

---

## 1.6 Roadmapa

Roadmapa projektu nie jest już linią „Etap 1–8”. System rozwija się obecnie równolegle w kilku podsystemach.

### Priorytet A — stabilizacja obecnej platformy

Najbliższe prace:

- eliminacja regresji PMM/VMM w puli buforów grafiki,
- stabilizacja `DialogPliku` i popupów po zmianie rozdzielczości,
- usunięcie pozostałych błędów testów BursztynUI,
- stabilizacja `SelektorKoloru`,
- eliminacja niepotrzebnych alokacji w hot-path GUI,
- poprawa raportowania `Testy Bursztyn OS`,
- konsolidacja dokumentacji BWS,
- dalsze testy manualne 1920×1080, 2560×1440 i 4K.

### Priorytet B — kernel i sprzęt

Kolejne duże kroki:

- SMP,
- pełniejszy model wielowątkowy,
- NVMe,
- USB Mass Storage,
- async I/O,
- rozbudowa PnP,
- suspend/resume,
- zarządzanie energią,
- Secure Boot,
- TPM-backed security,
- szyfrowanie danych.

### Priorytet C — sieć i usługi

Planowane:

- wiele jednoczesnych socketów,
- readiness/events,
- IPv6,
- przenoszenie długich operacji I/O poza synchroniczne ścieżki Ring 0,
- bardziej kompletne usługi Ring 3.

### Priorytet D — grafika

Planowane:

- dalsza optymalizacja kompozytora,
- precyzyjne damage regions,
- lepsza obsługa wielu monitorów,
- VirtIO-GPU jako etap pośredni,
- docelowo sprzętowa akceleracja GPU.

### Priorytet E — audio

Planowane:

- owned audio streams,
- buforowanie,
- mixer,
- format negotiation,
- aplikacyjne API audio.

### Priorytet F — Avocado

Docelowy potok:

```text
kod Avocado
    ↓
kompilator Avocado
    ↓
program.bur
    +
opis.aplikacji
    ↓
paczka .cebula
    ↓
instalacja / uruchomienie w Ring 3
```

Kompilator Avocado pozostaje przyszłym, osobnym etapem projektu.

---

## 1.7 Filozofia rozwoju

Bursztyn OS nie ma konkurować z Linux, Windows czy macOS poprzez kopiowanie ich rozwiązań funkcja po funkcji.

Celem jest budowa własnej, spójnej architektury i wykorzystanie istniejących systemów jako źródła wiedzy o problemach, które każdy dojrzały system operacyjny musi rozwiązać.

Projekt powinien:

- czerpać z dobrych praktyk systemowych,
- unikać kopiowania API bez potrzeby,
- zachowywać własne nazewnictwo i własne kontrakty,
- preferować bezpieczeństwo nad pozorną kompletnością,
- dokumentować ograniczenia,
- nie deklarować funkcji jako gotowej bez testów,
- zachowywać kompatybilność opublikowanego ABI tam, gdzie jest to możliwe.

---

## 1.8 Definicja aktualnego stanu projektu

Najbardziej precyzyjny opis Bursztyn OS na obecnym etapie brzmi:

> **Bursztyn OS jest niezależnym, eksperymentalnym systemem operacyjnym x86-64 rozwijanym od podstaw. Posiada własne jądro, Ring 3, BWS, model bezpieczeństwa PZB, system plików BSP2/PSF, sterowniki, natywny stos sieciowy, środowisko graficzne BursztynUI, instalator oraz aktualizacje A/B. System jest funkcjonalny w środowisku testowym, ale pozostaje w aktywnej fazie stabilizacji, rozszerzania kompatybilności sprzętowej i rozwoju API.**

To określenie powinno zastąpić wcześniejsze sformułowania o „pełnej dojrzałości operacyjnej”, dopóki projekt nie przejdzie szerszych testów sprzętowych, bezpieczeństwa, kompatybilności i długotrwałej stabilności.

---

## 1.9 Powiązane dokumenty

Szczegóły techniczne znajdują się w dalszych rozdziałach:

- `02_architektura_systemu.md` — Ring 0 / Ring 3 i PZB,
- `03_proces_rozruchu.md` — rozruch i Long Mode,
- `04_zarzadzanie_sprzetem.md` — GDT, IDT, APIC i sprzęt,
- `05_bursztynowy_system_plikow.md` — BSP/PSF,
- `06_wywolania_systemowe.md` — BWS; część opisu jest historyczna,
- `07_ekosystem_i_formaty.md` — `.bur`, `.cebula` i Avocado,
- `09_tryb_graficzny.md` — grafika i HAL,
- `10_siec.md` — stos sieciowy,
- `11_audyt_bws_nowoczesne.md` — nowsze rozszerzenia ABI,
- `instalator_audyt.md` — aktualna architektura instalatora,
- `ustawienia_ekranu_audyt.md` — rozdzielczość i skalowanie,
- `aktualizacja-systemu.md` — A/B, OTA i rollback.

Przy rozbieżnościach między starymi rozdziałami i nowszymi audytami pierwszeństwo powinny mieć:

1. aktualny kod,
2. nowszy audyt,
3. aktualne testy runtime,
4. starsza dokumentacja historyczna.
