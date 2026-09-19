# 06. Bursztynowe Wywołania Systemowe (BWS)

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** ABI Ring 3 → Ring 0, brama `SYSCALL`, rejestry BWS, bezpieczeństwo user-copy, PZB, pełna mapa BWS 1–87, nowoczesne uchwyty i ograniczenia bieżącej implementacji  
**Status:** dokument zastępuje historyczny opis 26 BWS

---

## 6.1 Rola BWS

**BWS — Bursztynowe Wywołania Systemowe** to publiczny interfejs pomiędzy programami Ring 3 i Jądrem Bursztyna Ring 0.

Program `.bur` nie wykonuje bezpośrednio operacji uprzywilejowanych takich jak:

- dostęp do portów I/O,
- mapowanie fizycznej pamięci,
- sterowanie AHCI,
- dostęp do E1000,
- obsługa xHCI,
- modyfikacja tablic stron,
- bezpośredni zapis do zasobów kernela.

Zamiast tego aplikacja składa żądanie BWS, a jądro:

1. identyfikuje proces wywołujący,
2. sprawdza numer BWS,
3. waliduje argumenty,
4. sprawdza wskaźniki Ring 3,
5. sprawdza PZB i `PRAWO_*`,
6. sprawdza własność uchwytów,
7. wykonuje operację,
8. zwraca wynik do Ring 3.

BWS jest więc zarówno ABI, jak i granicą bezpieczeństwa.

---

## 6.2 Aktualny zakres ABI

Historyczna dokumentacja opisywała **26 wywołań BWS**.

Ten stan jest nieaktualny.

Aktualne źródło numeracji w `bws_api.h` definiuje:

```text
BWS 0      — brak / nieprzydzielony
BWS 1–60   — starszy, zamrożony zakres ABI
BWS 61–83  — nowoczesne rozszerzenia ABI
BWS 84–85  — OTA / aktualizacje systemu
BWS 86–87  — podsystem motywów
BWS 88     — pierwszy wolny numer
```

Aktualnie obowiązuje więc zakres publicznych numerów:

```text
1..87
```

Numer:

```text
0
```

nie jest obsługiwanym BWS.

Nieznany numer BWS jest odrzucany, a dispatcher zwraca:

```text
UINT64_MAX
```

Nowe numery należy przydzielać wyłącznie przez centralny plik:

```text
bws_api.h
```

---

## 6.3 Zasada zgodności ABI

Zakres BWS 1–60 jest traktowany jako historycznie opublikowany i zamrożony.

Nowsza rozbudowa była wykonywana **addytywnie**:

```text
istniejący numer
→ nie zmieniaj znaczenia

nowa funkcja
→ przydziel kolejny wolny numer
```

Przykłady zachowania zgodności:

```text
BWS 57 — GUI blit
BWS 60 — monotoniczne nanosekundy HPET
```

pozostały bez zmiany podczas dodawania BWS 61–83.

BWS 84–85 zostały następnie dodane dla OTA, a 86–87 dla motywów.

---

# 6.4 Sprzętowa brama `SYSCALL`

Bursztyn OS używa instrukcji:

```asm
syscall
```

do przejścia z Ring 3 do Ring 0.

Podczas inicjalizacji jądro konfiguruje między innymi:

- `IA32_LSTAR`,
- `IA32_STAR`,
- `IA32_FMASK`,
- bazy GS wykorzystywane przez `SWAPGS`.

`IA32_LSTAR` wskazuje na asemblerową bramę BWS:

```text
brama_wywolan_systemowych
```

---

## 6.5 Ważna korekta: TSS nie przełącza RSP przy SYSCALL

Starsza dokumentacja mówiła, że instrukcja `SYSCALL` automatycznie pobiera `RSP0` z TSS.

W x86-64 jest to niepoprawne.

Sprzętowe zachowanie `SYSCALL` obejmuje między innymi:

```text
RCX = RIP użytkownika po SYSCALL
R11 = RFLAGS użytkownika
CS/SS = selektory wynikające z IA32_STAR
RIP = IA32_LSTAR
```

ale:

```text
RSP NIE jest automatycznie przełączany
```

na `TSS.RSP0`.

Bursztyn OS przełącza stos jawnie we własnej bramie asemblerowej.

---

# 6.6 Prywatny stos Ring 0

Aktualna brama BWS nie przechowuje stosu użytkownika w jednej globalnej zmiennej.

Po wejściu:

```text
RSP = stos Ring 3
```

brama:

1. zachowuje użytkowy RSP,
2. wykonuje `SWAPGS`,
3. stosuje `LFENCE`,
4. pobiera z kernelowego GS bezpieczny stos jądra bieżącego procesu,
5. przechodzi na prywatny stos Ring 0,
6. zapisuje kontekst,
7. dopiero wtedy wywołuje kod C++ dispatchera.

Scheduler aktualizuje adres bezpiecznego stosu przy zmianie procesu.

---

# 6.7 `SWAPGS` i GS

Aktualny model wykorzystuje oddzielne bazy GS dla Ring 3 i Ring 0.

W uproszczeniu:

```text
Ring 3:
GS_BASE          → przestrzeń użytkownika
KERNEL_GS_BASE   → dane kernela / stos BWS

SYSCALL:
SWAPGS

Ring 0:
GS_BASE          → kernelowy kontekst BWS
KERNEL_GS_BASE   → poprzedni user GS
```

`LFENCE` po `SWAPGS` ogranicza spekulacyjne wykorzystanie nieprawidłowej bazy GS przed odczytem danych kernela.

---

# 6.8 `IA32_FMASK`

Jądro używa `IA32_FMASK`, aby przy wejściu BWS wyczyścić niebezpieczne flagi procesora.

Aktualna brama nie powinna dziedziczyć z Ring 3 między innymi flag mogących wpłynąć na poprawność kodu Ring 0.

Przed `SYSRETQ` jądro przygotowuje kontrolowany zestaw flag użytkownika.

Aktualny kod definiuje bezpieczne RFLAGS powrotne m.in. z:

```text
bit 1 = 1
IF    = 1
```

i celowo nie obiecuje zachowania flag arytmetycznych przez BWS.

---

# 6.9 ABI rejestrów BWS

Aktualne ABI Ring 3 → BWS jest jawnie zdefiniowane w `syscall.S` oraz wrapperze `bws_wywolaj()`.

| Rejestr | Znaczenie |
|---|---|
| `R8` | numer BWS |
| `R9` | argument 1 |
| `R10` | argument 2 |
| `R12` | argument 3 |
| `R13` | argument 4 |
| `RAX` | wynik |

Wrapper Ring 3 ma postać logicznie równoważną:

```cpp
uint64_t bws_wywolaj(
    uint64_t numer,
    uint64_t arg1 = 0,
    uint64_t arg2 = 0,
    uint64_t arg3 = 0,
    uint64_t arg4 = 0);
```

Przykładowa implementacja:

```cpp
register uint64_t r8  asm("r8")  = numer;
register uint64_t r9  asm("r9")  = arg1;
register uint64_t r10 asm("r10") = arg2;
register uint64_t r12 asm("r12") = arg3;
register uint64_t r13 asm("r13") = arg4;
register uint64_t rax asm("rax");

asm volatile(
    "syscall"
    : "=a"(rax)
    : "r"(r8), "r"(r9), "r"(r10), "r"(r12), "r"(r13)
    : "rcx", "r11", "memory", "cc"
);
```

`RCX` i `R11` są clobbered przez sprzętowe zachowanie `SYSCALL`.

---

# 6.10 ABI wewnątrz Ring 0

Po zapisaniu kontekstu brama asemblerowa przygotowuje argumenty zgodnie z ABI funkcji C++ dispatchera.

Koncepcyjnie:

```text
RDI = numer BWS
RSI = argument 1
RDX = argument 2
RCX = argument 3
R8  = argument 4
RAX = wynik
```

Dispatcher wywołuje następnie odpowiednią implementację kernela.

---

# 6.11 Powrót do Ring 3

Przed `SYSRETQ` brama:

- blokuje maskowalne IRQ na krytyczny fragment powrotu,
- sprawdza docelowy RIP,
- sprawdza docelowy RSP,
- przygotowuje bezpieczne RFLAGS,
- odtwarza zachowywane rejestry,
- wykonuje `SWAPGS`,
- wraca do Ring 3.

Nie należy wykonywać `SYSRETQ` do niezweryfikowanego, niekanonicznego adresu użytkownika.

---

# 6.12 BWS 38 i blokowanie procesu

Normalny syscall nie jest arbitralnie przełączany w połowie wykonania tylko dlatego, że wystąpił timer.

Wyjątkiem jest specjalnie zaprojektowana ścieżka blokująca, m.in.:

```text
BWS 38 — event wait
```

BWS 38 zachowuje ramę `SYSCALL` na prywatnym stosie Ring 0.

Scheduler może zablokować proces oczekujący na zdarzenie, a po jego wybudzeniu wznowić ten sam syscall.

To jest kontrolowany mechanizm, a nie ogólna preempcja dowolnego kodu BWS.

---

# 6.13 Dispatcher

Starsze BWS 1–60 są obsługiwane przez główny dispatcher w `syscalls.cpp`.

Nowszy zakres:

```text
61 <= BWS < 88
```

jest delegowany do:

```text
bws_nowoczesne_obsluz(...)
```

Obejmuje to również:

- OTA 84–85,
- motywy 86–87.

Dzięki temu nowe ABI jest odseparowane od historycznego dużego `switch` dla BWS 1–60.

---

# 6.14 Pełna mapa BWS 1–87

Poniższa tabela odpowiada aktualnej centralnej numeracji `bws_api.h`.

| BWS | Nazwa kanoniczna | Znaczenie |
|---:|---|---|
| 0 | `BWS_BRAK` | nieprzydzielony / nieobsługiwany |
| 1 | `BWS_WYPISZ` | tekst / diagnostyka |
| 2 | `BWS_UTWORZ_PLIK` | utworzenie pliku |
| 3 | `BWS_ZAPISZ_PLIK` | zapis pliku |
| 4 | `BWS_POBIERZ_ZNAK` | legacy pobranie znaku |
| 5 | `BWS_CZYTAJ_PLIK` | odczyt pliku |
| 6 | `BWS_LISTUJ_KATALOG` | lista katalogu |
| 7 | `BWS_USUN_TWOR` | usunięcie obiektu |
| 8 | `BWS_ZMIEN_NAZWE` | zmiana nazwy |
| 9 | `BWS_CZAS_RTC` | tekst czasu RTC |
| 10 | `BWS_URUCHOM_PROGRAM` | uruchomienie `.bur` |
| 11 | `BWS_PING` | ICMP ping |
| 12 | `BWS_DNS_LEGACY` | legacy DNS |
| 13 | `BWS_HTTP_DO_PLIKU_LEGACY` | legacy HTTP do pliku |
| 14 | `BWS_GUI_RYSUJ_OKNO` | legacy rysowanie okna |
| 15 | `BWS_GUI_TEKST` | tekst GUI |
| 16 | `BWS_GUI_WYCZYSC` | czyszczenie obszaru GUI |
| 17 | `BWS_GUI_ODSWIEZ` | żądanie odświeżenia GUI |
| 18 | `BWS_GUI_POBIERZ_MYSZ` | legacy stan myszy |
| 19 | `BWS_GUI_ODSWIEZ_PULPIT` | legacy odświeżenie pulpitu |
| 20 | `BWS_GUI_TEKST_KOLOR` | tekst kolorowany / skalowany |
| 21 | `BWS_GUI_PROSTOKAT` | prostokąt GUI |
| 22 | `BWS_GUI_PRZEJECIE_MYSZY` | legacy przejęcie myszy |
| 23 | `BWS_GUI_ROZDZIELCZOSC` | rozdzielczość ekranu |
| 24 | `BWS_GUI_SZEROKOSC_ZNAKU` | szerokość glifu |
| 25 | `BWS_RESET` | reset systemu |
| 26 | `BWS_SHUTDOWN` | wyłączenie systemu |
| 27 | `BWS_DZWIEK_TON` | testowy ton Intel HDA |
| 28 | `BWS_DNS` | DNS |
| 29 | `BWS_HTTP` | HTTP |
| 30 | `BWS_HTTPS` | HTTPS |
| 31 | `BWS_TLS_CERTYFIKAT` | status certyfikatu TLS |
| 32 | `BWS_ZAKONCZ_PROCES` | zakończenie procesu |
| 33 | `BWS_GUI_UTWORZ_WARSTWE` | utworzenie / zastąpienie własnej warstwy |
| 34 | `BWS_GUI_PRZESUN_WARSTWE` | przesunięcie własnej warstwy |
| 35 | `BWS_ALOKUJ_STERTE` | powiększenie prywatnej sterty Ring 3 |
| 36 | `BWS_GUI_CZY_ZAMKNIETO_POWLOKE` | legacy stan zamknięcia powłoki |
| 37 | `BWS_ZDARZENIE_POLL` | nieblokujące pobranie zdarzenia |
| 38 | `BWS_ZDARZENIE_CZEKAJ` | blokujące oczekiwanie na zdarzenie |
| 39 | `BWS_GUI_CAPTURE` | capture wejścia |
| 40 | `BWS_GUI_SYSTEM_OVERLAY` | overlay systemowy |
| 41 | `BWS_GUI_MINIMALIZUJ` | minimalizacja własnego okna |
| 42 | `BWS_GUI_OKNA_SNAPSHOT` | snapshot listy okien |
| 43 | `BWS_GUI_AKTYWUJ_OKNO` | aktywacja okna |
| 44 | `BWS_ROZMIAR_PLIKU` | rozmiar pliku |
| 45 | `BWS_ARGUMENT_STARTOWY` | argument startowy procesu |
| 46 | `BWS_UTWORZ_KATALOG` | utworzenie katalogu |
| 47 | `BWS_METADANE_PLIKU` | metadane pliku |
| 48 | `BWS_PRZENIES_TWOR` | przeniesienie obiektu |
| 49 | `BWS_GUI_CELE_DROP` | rejestracja celów drop |
| 50 | `BWS_GUI_AKTUALIZUJ_DRAG` | aktualizacja / zakończenie drag |
| 51 | `BWS_GUI_WYSOKOSC_FONTU` | wysokość fontu |
| 52 | `BWS_SCHOWEK_PLIK_USTAW` | legacy schowek plików — ustaw |
| 53 | `BWS_SCHOWEK_PLIK_POBIERZ` | legacy schowek plików — pobierz |
| 54 | `BWS_SCHOWEK_PLIK_WYCZYSC` | legacy schowek plików — wyczyść |
| 55 | `BWS_KOPIUJ_TWOR` | kopiowanie obiektu |
| 56 | `BWS_GUI_POPUP_APLIKACJI` | popup aplikacji |
| 57 | `BWS_GUI_BLIT` | GUI blit |
| 58 | `BWS_GUI_TYTUL_OKNA` | tytuł okna |
| 59 | `BWS_CZYTAJ_FRAGMENT_PLIKU` | zakresowy odczyt pliku |
| 60 | `BWS_CZAS_MONOTONICZNY_NS` | monotoniczny czas HPET w ns |
| 61 | `BWS_GUI_GEOMETRIA` | transakcyjna geometria własnego okna |
| 62 | `BWS_GUI_MONITOR_INFO` | monitor / work area / skala |
| 63 | `BWS_GUI_USTAW_WORK_AREA` | zaufana publikacja work area |
| 64 | `BWS_SHM_CREATE` | utworzenie SHM |
| 65 | `BWS_SHM_MAP` | mapowanie SHM |
| 66 | `BWS_SHM_UNMAP` | odmapowanie SHM |
| 67 | `BWS_SHM_CLOSE` | zamknięcie SHM |
| 68 | `BWS_IPC_CREATE` | utworzenie endpointu IPC |
| 69 | `BWS_IPC_SEND` | wysłanie IPC |
| 70 | `BWS_IPC_RECEIVE` | odbiór IPC |
| 71 | `BWS_IPC_CLOSE` | zamknięcie endpointu IPC |
| 72 | `BWS_FILE_WATCH` | watch / unwatch plików |
| 73 | `BWS_CLIPBOARD_ENUM` | enumeracja formatów clipboard |
| 74 | `BWS_CLIPBOARD_SIZE` | rozmiar formatu clipboard |
| 75 | `BWS_CLIPBOARD_READ` | odczyt formatu clipboard |
| 76 | `BWS_CLIPBOARD_SET` | zapis formatu clipboard |
| 77 | `BWS_GUI_CURSOR_SHAPE` | systemowy kształt kursora |
| 78 | `BWS_GUI_CURSOR_CUSTOM` | własny kursor procesu |
| 79 | `BWS_SOCKET_CREATE` | utworzenie socketu TCP/IPv4 |
| 80 | `BWS_SOCKET_CONNECT` | connect |
| 81 | `BWS_SOCKET_SEND` | send |
| 82 | `BWS_SOCKET_RECV` | receive |
| 83 | `BWS_SOCKET_CLOSE` | close |
| 84 | `BWS_OTA_STATUS` | stan aktualizacji A/B |
| 85 | `BWS_OTA_POLECENIE` | polecenie podsystemu OTA |
| 86 | `BWS_MOTYW_POBIERZ` | odczyt aktywnego motywu |
| 87 | `BWS_MOTYW_POLECENIE` | operacje na motywach |
| 88 | `BWS_PIERWSZY_WOLNY` | pierwszy numer dostępny dla następnego ABI |

---

# 6.15 BWS 61 — transakcyjna geometria okna

BWS 61 zastępuje konieczność składania zmiany rozmiaru z kilku historycznych wywołań.

Obsługuje geometrię:

```text
x
y
width
height
```

własnej warstwy bieżącego procesu.

Wymaga:

```text
PRAWO_GUI
```

i nie pozwala procesowi zmieniać warstwy obcego PID.

Ważny model bezpieczeństwa:

```text
validate
→ allocate/prepare new surface
→ publish new geometry
→ dirty old + new region
→ release old backing resource
```

Nie należy niszczyć starego bufora przed upewnieniem się, że nowy został poprawnie przygotowany.

---

# 6.16 BWS 62–63 — monitor, work area i skala

BWS 62 zwraca wersjonowaną strukturę opisującą monitor.

Aktualna implementacja jest przygotowana do rozszerzenia na wiele monitorów, ale obecny system używa jednego monitora logicznego.

Rekord zawiera między innymi informacje potrzebne do:

- fizycznej geometrii,
- work area,
- skali interfejsu.

BWS 63 służy do publikowania globalnego work area.

Ta operacja jest zaufana i wymaga odpowiedniego prawa systemowego, m.in.:

```text
PRAWO_SYSTEM_CONFIG
```

Zwykła aplikacja GUI nie może samodzielnie przesuwać globalnego obszaru roboczego.

---

# 6.17 BWS 64–67 — SHM

BWS 64–67 implementują natywną pamięć współdzieloną.

Operacje:

```text
64 — create
65 — map
66 — unmap
67 — close
```

Aktualne limity:

```text
strona       = 4 KiB
max / obiekt = 2 MiB
max globalny = 8 MiB
```

Właściwości bezpieczeństwa:

- ramki są zerowane przed udostępnieniem,
- Ring 3 nie otrzymuje fizycznych adresów,
- uchwyty posiadają generację,
- dostęp jest kontrolowany per PID,
- prawa mogą być przekazywane jawnie przez IPC,
- cleanup procesu usuwa jego mapowania i prawa.

---

# 6.18 BWS 68–71 — IPC

IPC udostępnia bounded endpointy.

Operacje:

```text
68 — endpoint create
69 — send
70 — receive
71 — close
```

Aktualne limity:

```text
512 B  — maksymalny payload wiadomości
16     — maksymalna liczba oczekujących wiadomości / endpoint
32     — maksymalna liczba endpointów
```

Uchwyty zawierają generację.

Odbiór i zamknięcie endpointu są kontrolowane przez właściciela.

IPC może przenosić prawo dostępu do SHM.

Gotowość IPC może być sygnalizowana przez Event ABI.

---

# 6.19 BWS 72 — file watch

BWS 72 implementuje:

```text
watch
unwatch
```

przez jeden numer syscalla i kontrolowany uchwyt.

Mechanizm obejmuje:

- owner PID,
- generację,
- maskę akcji,
- bounded event queue,
- ponowną kontrolę praw.

Aktualny stan jest nadal **częściowy**.

Nie wszystkie mutacje wewnętrzne BSP2 mają jeszcze centralne hooki watcherów. Brakuje również części bogatszych danych dla rename/move/delete.

---

# 6.20 BWS 73–76 — clipboard MIME

Nowoczesny clipboard jest oddzielony od legacy schowka plikowego 52–54.

Operacje:

```text
73 — enumerate
74 — size
75 — read
76 — set
```

Aktualne limity:

```text
8 formatów
1 MiB / format
2 MiB łącznie
```

Dane są kopiowane przez jądro.

Kernel nie przechowuje surowego pointera do bufora Ring 3.

---

# 6.21 BWS 77–78 — kursory

BWS 77 pozwala wybrać systemowy kształt kursora dla procesu.

BWS 78 pozwala przekazać własny obraz kursora.

Dla custom cursor obowiązuje:

```text
maksymalnie 64 × 64
BGRA8888 lub RGBA8888
walidowany hotspot
kernel-owned copy
```

Stary obraz jest zwalniany dopiero po udanym przygotowaniu nowego.

---

# 6.22 BWS 79–83 — socket facade

Nowy interfejs socketów obejmuje:

```text
79 — create
80 — connect
81 — send
82 — recv
83 — close
```

Każdy socket posiada:

- właściciela PID,
- generacyjny uchwyt,
- kontrolę `PRAWO_SIEC`,
- walidację buforów użytkownika.

### Aktualne ograniczenie

Bieżący bazowy stos TCP nadal obsługuje:

```text
jedną systemową sesję TCP jednocześnie
```

Dlatego obecność BWS 79–83 nie oznacza jeszcze pełnego modelu równoczesnych socketów znanego z dojrzałych systemów.

Brakuje także pełnego readiness API dla wielu socketów.

---

# 6.23 BWS 84–85 — OTA

BWS 84:

```text
BWS_OTA_STATUS
```

kopiuje do Ring 3 wersjonowany stan podsystemu aktualizacji.

Aktualna struktura `BwsOtaStatus` ma w dokumentowanej wersji:

```text
384 B
```

i zawiera m.in.:

- wersję ABI,
- stan,
- kod błędu,
- postęp,
- aktywny i nieaktywny slot,
- informację o rollbacku,
- status podpisu,
- status SHA-256,
- informacje o wersji i kanale.

BWS 85:

```text
BWS_OTA_POLECENIE
```

wykonuje kontrolowane operacje podsystemu OTA.

BWS OTA podlegają PZB i maskom praw.

Operacje administracyjne wymagają zaufanego procesu oraz odpowiednich praw, w szczególności:

```text
PRAWO_SYSTEM_CONFIG
```

---

# 6.24 BWS 86 — pobranie motywu

Aktualny source tree definiuje:

```text
BWS_MOTYW_POBIERZ = 86
```

BWS kopiuje aktywny motyw do bufora Ring 3.

Obsługiwane są dwie wersje ABI:

```text
BursztynMotywV1 — 144 B
BursztynMotyw   — ABI v2, 172 B
```

Aktualna wersja ABI motywu:

```text
2
```

BWS sprawdza, czy docelowy zakres użytkownika jest poprawny i zapisywalny.

Wersja v1 jest zachowywana dla kompatybilności odczytu.

---

# 6.25 BWS 87 — polecenia motywów

BWS:

```text
BWS_MOTYW_POLECENIE = 87
```

obsługuje między innymi polecenia:

```text
MOTYW_CMD_USTAW       = 1
MOTYW_CMD_PODGLAD     = 2
MOTYW_CMD_ZATWIERDZ   = 3
MOTYW_CMD_PRZYWROC    = 4
MOTYW_CMD_LISTUJ      = 5
```

Mutacje motywu wymagają zaufanego procesu.

Aktualny predykat autoryzacyjny wymaga logicznie:

```text
PZB <= PZB_ZAUFANE
+
PRAWO_SYSTEM_CONFIG
```

ID motywu jest walidowane i nie może być dowolną ścieżką pliku.

Dozwolony format identyfikatora ogranicza się do bezpiecznego zestawu znaków, dzięki czemu przykłady typu:

```text
../etc/passwd
/system/motywy/grafit.motyw
GRAFIT
motyw ze spacja
```

są odrzucane.

Podgląd motywu posiada właściciela procesu i token transakcji.

---

# 6.26 Status testów BWS 86–87

Kod źródłowy zawiera testy:

- błędnego wskaźnika dla BWS 86,
- ABI v2,
- kompatybilności v1,
- kontroli uprawnień BWS 87,
- walidacji ID,
- listowania motywów,
- stres-testu odczytu i accessorów.

Należy jednak rozróżnić **obecność testów w kodzie** od **aktualnego wyniku całej suity runtime**.

W najnowszym pełnym przebiegu projektu zgłaszane były jeszcze błędy asercji związane z grupą motywów BWS 86–87, mimo że lokalny komunikat sekcji testowej potrafił raportować `OK`.

Do czasu poprawienia raportowania i uzyskania nowego pełnego przebiegu bez `FAIL` BWS 86–87 należy opisywać jako:

> **zaimplementowane i aktywnie stabilizowane, ale nie jako bezwarunkowo produkcyjnie zweryfikowane.**

---

# 6.27 Event ABI

Nie wszystkie nowe funkcje Ring 3 wymagają nowego numeru BWS.

BWS 37–38 dostarczają kolejkę zdarzeń procesu, której typy zostały rozszerzone.

Nowsze Event ABI obejmuje między innymi:

```text
event 23 — fizyczny klawisz
event 24 — Unicode TEXT_INPUT
event 25 — vertical/horizontal mouse wheel
event 26 — precision pointer
event 27 — file watcher
event 28 — IPC readiness
```

Struktura:

```text
bws_zdarzenie
```

zachowuje rozmiar:

```text
40 B
```

co pozwala rozszerzać semantykę zdarzeń bez tworzenia osobnego syscalla dla każdego typu wejścia.

---

# 6.28 BWS 60 — monotoniczny HPET

BWS 60 zwraca:

```text
monotonic HPET nanoseconds
```

albo:

```text
0
```

gdy HPET nie jest dostępny.

Nie ujawnia:

- MMIO,
- adresu fizycznego,
- rejestrów HPET.

Jest odpowiednim źródłem dla:

- timeoutów,
- animacji,
- pomiaru czasu trwania,
- profilowania.

Czas kalendarzowy nadal należy do RTC.

---

# 6.29 Kontrola PZB i `PRAWO_*`

BWS nie ufa procesowi tylko dlatego, że działa w Ring 3 z poprawnego obrazu `.bur`.

Każdy wrażliwy syscall może sprawdzać:

```text
PZB
+
PRAWO_*
+
PID
+
właściciela zasobu
+
generację uchwytu
+
argumenty
```

Podstawowe maski projektu obejmują między innymi:

```cpp
#define PRAWO_PLIKI_CZYTAJ      (1ULL << 0)
#define PRAWO_PLIKI_ZAPISZ      (1ULL << 1)
#define PRAWO_SIEC              (1ULL << 2)
#define PRAWO_GUI               (1ULL << 3)
#define PRAWO_URUCHOM_PROGRAM   (1ULL << 4)
#define PRAWO_SYSTEM_CONFIG     (1ULL << 5)
#define PRAWO_STEROWNIK         (1ULL << 6)
#define PRAWO_DEBUG             (1ULL << 7)
```

Szczegółowa polityka zależy od danego BWS.

---

# 6.30 Checked user-copy

Wskaźnik Ring 3 jest niezaufanym wejściem.

Nowoczesne BWS kopiują struktury i dane przez kontrolowane helpery.

Walidacja obejmuje między innymi:

- kanoniczność adresu,
- dolną granicę user space,
- sprawdzenie całego zakresu,
- obecność mapowania,
- wymagane prawa strony,
- ochronę przed overflow,
- właściwy kierunek kopiowania.

Koncepcyjnie:

```text
user pointer
    ↓
range validation
    ↓
PTE validation
    ↓
copy_from_user / copy_to_user
    ↓
kernel-owned structure
```

Kernel nie powinien zachowywać surowego pointera Ring 3 do późniejszego użycia.

---

# 6.31 Sprawdzanie overflow

Nowoczesna warstwa BWS sprawdza:

- dodawanie,
- mnożenie,
- zaokrąglanie liczby stron.

Przykładowe ryzykowne operacje:

```text
count * sizeof(entry)
width * height * 4
offset + length
size + PAGE_SIZE - 1
```

Przepełnienie musi zakończyć operację błędem przed alokacją lub dostępem do pamięci.

---

# 6.32 Uchwyty generacyjne

SHM, IPC, watchery i sockety używają niezerowych generacji uchwytów.

Koncepcja:

```text
handle = slot + generation
```

Jeżeli slot zostanie użyty ponownie, generacja się zmienia.

Stary handle nie może zacząć wskazywać nowego obiektu.

Chroni to przed stale-handle i częścią klas use-after-close.

---

# 6.33 Własność zasobów per PID

Uchwyt nie jest automatycznie globalnym prawem.

Kernel kontroluje właściciela dla zasobów takich jak:

- IPC endpoint,
- watcher,
- socket,
- custom cursor,
- GUI layer.

SHM posiada dodatkowo jawny zestaw PID mających prawo dostępu.

Proces nie może wykonywać owner-only operacji na zasobie obcego PID tylko dlatego, że zna jego numer uchwytu.

---

# 6.34 Cleanup procesu

Zakończenie procesu musi usunąć jego zasoby Ring 0.

Aktualny cleanup obejmuje m.in.:

- warstwy GUI,
- kursory,
- mapowania SHM,
- prawa SHM,
- wiadomości IPC,
- endpointy IPC,
- watchery,
- socket należący do procesu.

Shared PTE posiada marker, dzięki któremu zwykłe niszczenie przestrzeni adresowej nie zwalnia drugi raz ramek SHM.

---

# 6.35 Boolean wrappers

Istotna zasada SDK:

```text
sukces boolean = wynik == 1
```

Nie należy używać:

```cpp
if (bws_wywolaj(...))
```

dla wywołania, w którym `UINT64_MAX` oznacza nieznany numer, ponieważ taka wartość również jest niezerowa.

Poprawny wzorzec:

```cpp
return bws_wywolaj(...) == 1ULL;
```

---

# 6.36 Kody wyniku

Nie istnieje jedna globalna reguła:

```text
0 = każdy możliwy błąd
1 = każdy możliwy sukces
```

Historyczne BWS mają różne kontrakty.

Przykładowo wynik może oznaczać:

- boolean,
- handle,
- rozmiar,
- status,
- czas,
- token,
- `UINT64_MAX` dla nieznanego BWS.

Dlatego aplikacja powinna korzystać z wrappera odpowiadającego konkretnemu wywołaniu, a nie interpretować wszystkie BWS jednakowo.

---

# 6.37 Historyczne BWS GUI

BWS 14–24 pochodzą z wczesnego modelu GUI.

Część z nich pozostaje dla zgodności, ale nowe aplikacje powinny preferować współczesne mechanizmy tam, gdzie istnieją.

Przykłady:

```text
BWS 18 — legacy mouse state
BWS 22 — legacy mouse ownership
```

nie są docelowym modelem input routing.

Współczesna architektura używa:

- per-process event queue,
- focus,
- capture,
- hit-test,
- BWS 37–39,
- nowszego Event ABI.

---

# 6.38 Historyczne BWS sieciowe

BWS 12–13 pozostają jako legacy.

Później dodano:

```text
28 — DNS
29 — HTTP
30 — HTTPS
31 — TLS certificate status
```

a następnie:

```text
79–83 — owned TCP/IPv4 socket facade
```

Nie należy jednak przedstawiać socket facade jako pełnego nowoczesnego stacku socketów, ponieważ bazowy TCP nadal ma jedną systemową sesję.

---

# 6.39 Audio

BWS 27 nadal służy do:

```text
HDA test tone
```

Nie jest to pełne API audio.

Obecnie brakuje publicznego zestawu BWS dla:

- tworzenia własnego streamu,
- wyboru formatu,
- zapisu ramek,
- query capacity,
- miksowania wielu aplikacji.

Dlatego BWS 27 należy dokumentować jako funkcję testową/ograniczoną.

---

# 6.40 Async I/O

Obecność:

- event queue,
- IPC,
- watcherów

nie oznacza, że Bursztyn OS ma już pełne asynchroniczne I/O.

Aktualnie PSF i część networking/TLS nadal używają operacji synchronicznych.

Brakuje kompletnego modelu:

```text
submit
→ request handle
→ WAITING_IO
→ worker
→ completion
→ cancellation/lifetime
```

Dokumentacja nie powinna tworzyć fikcyjnego BWS „async” przed zaimplementowaniem całej semantyki.

---

# 6.41 Dialogi i powiadomienia

Nowoczesny audyt BWS celowo nie dodawał kernelowego syscalla „pokaż dialog systemowy”.

Docelowa architektura zakłada komponenty Ring 3 wykorzystujące:

- IPC,
- BursztynUI,
- popup infrastructure.

W późniejszym rozwoju BursztynUI powstały m.in.:

- `DialogPliku`,
- `DialogKoloru`,
- `DialogKomunikatu`,
- `Powiadomienie`.

Są to komponenty frameworka Ring 3, a nie osobne numery BWS.

To rozróżnienie należy zachować w dokumentacji ABI.

---

# 6.42 Zasada: funkcja GUI ≠ nowy syscall

BWS powinno udostępniać mechanizm systemowy, a nie każdą kontrolkę.

Przykładowo nie potrzebujemy osobnych syscalli dla:

```text
Przycisk
Tabela
Drzewo
DialogPliku
SelektorKoloru
Toast
```

Te elementy należą do BursztynUI.

Kernel powinien udostępniać niższe mechanizmy:

- powierzchnie,
- eventy,
- IPC,
- clipboard,
- kursory,
- file API,
- SHM.

---

# 6.43 Bezpieczeństwo BWS OTA

BWS 84–85 operują na szczególnie wrażliwym podsystemie.

Aplikacja aktualizacji nie powinna posiadać bezpośredniego dostępu do:

- surowych struktur boot state,
- dowolnego LBA,
- arbitralnego zapisu do ESP.

Ring 3 wydaje kontrolowane polecenie, a zaufany kod systemowy wykonuje operację po walidacji.

System aktualizacji dodatkowo wykorzystuje:

- A/B,
- SHA-256,
- RSA-2048,
- CRC32,
- generacyjny boot state,
- rollback.

---

# 6.44 Bezpieczeństwo BWS motywów

BWS 86 jest odczytem i stosuje sprawdzane kopiowanie do Ring 3.

BWS 87 rozróżnia operacje odczytowe i mutujące.

Mutacje wymagają zaufanego procesu oraz `PRAWO_SYSTEM_CONFIG`.

Podgląd jest transakcyjny i przypisany do procesu.

Inny proces nie powinien móc przejąć aktywnego podglądu.

---

# 6.45 Testy BWS 61–83

Audyt z 2026-08-30 dodał smoke suite Ring 3 obejmującą m.in.:

- monitor/work area,
- monotoniczny czas,
- geometrię,
- SHM,
- IPC,
- brak praw plikowych/sieciowych,
- clipboard MIME,
- system/custom cursor.

Audyt zaznaczał jednocześnie, że przed nazwaniem ABI „production-stable” wymagane są dodatkowe wieloprocesowe i destrukcyjne przypadki graniczne.

Ta ostrożność nadal obowiązuje.

---

# 6.46 Minimalna macierz bezpieczeństwa dla nowego BWS

Każdy nowy syscall powinien mieć testy:

1. poprawnego przypadku,
2. `NULL`,
3. błędnego pointera Ring 3,
4. zakresu przechodzącego przez niezmapowaną stronę,
5. braku wymaganego `PRAWO_*`,
6. złego właściciela,
7. stale handle,
8. minimalnego rozmiaru,
9. maksymalnego rozmiaru,
10. `MAX + 1`,
11. integer overflow,
12. cleanup po śmierci procesu.

Dla uchwytów:

```text
create
→ close
→ reuse same slot
→ old handle must fail
```

Dla zasobów:

```text
baseline
→ process allocates resources
→ process exits
→ resource count returns to baseline
```

---

# 6.47 Wymagania przy dodawaniu BWS 88+

Przed przydzieleniem kolejnego numeru należy:

1. sprawdzić `BWS_PIERWSZY_WOLNY`;
2. dopisać numer wyłącznie w `bws_api.h`;
3. nie zmieniać semantyki istniejącego numeru;
4. zdefiniować wersjonowaną strukturę ABI, jeśli BWS wymienia dane binarne;
5. dodać `static_assert(sizeof(...))`;
6. dodać checked user-copy;
7. zdefiniować PZB / `PRAWO_*`;
8. zdefiniować limity;
9. zdefiniować ownership/lifetime;
10. dodać cleanup procesu;
11. dodać testy negatywne;
12. zaktualizować SDK i dokumentację.

Jeżeli daną funkcję można poprawnie zbudować w Ring 3 na istniejących mechanizmach, nie należy automatycznie dodawać nowego BWS.

---

# 6.48 Stabilność ABI

BWS jest publicznym kontraktem aplikacji, dlatego zmiany struktur wymagają wersjonowania.

Dobrym przykładem jest podsystem motywów:

```text
BursztynMotywV1 = 144 B
BursztynMotyw v2 = 172 B
```

BWS 86 potrafi zachować kompatybilny odczyt starszej wersji.

Podobny model powinien być stosowany do przyszłych struktur:

```cpp
struct BwsFoo {
    uint16_t wersja;
    uint16_t rozmiar_struktury;
    ...
};
```

---

# 6.49 Aktualny status jakości ABI

Aktualny stan można opisać następująco:

### Zaimplementowane

- BWS 1–87,
- brama `SYSCALL`,
- per-process kernel stack,
- checked user-copy,
- generacyjne uchwyty,
- SHM,
- IPC,
- file watch,
- clipboard MIME,
- kursory,
- socket facade,
- OTA,
- motywy.

### Nadal ograniczone / częściowe

- async I/O,
- pełne file-watch hooks,
- wiele równoczesnych socketów TCP,
- pełne audio stream API,
- pełne IME,
- pełny high-resolution HID,
- uniwersalne system-service discovery.

### Aktywnie stabilizowane

- BWS 86–87 / testy motywów,
- raportowanie błędów grup testowych,
- wybrane integracyjne ścieżki GUI korzystające z nowego ABI.

---

# 6.50 Czego nie należy już pisać

### Nieaktualne

```text
BWS zawiera 26 wywołań.
```

### Aktualne

```text
Aktualne publiczne ABI używa BWS 1–87,
a BWS_PIERWSZY_WOLNY ma wartość 88.
```

---

### Niepoprawne

```text
RAX zawiera numer BWS.
```

### Aktualne ABI Bursztyn OS

```text
R8  = numer BWS
R9  = arg1
R10 = arg2
R12 = arg3
R13 = arg4
RAX = wynik
```

---

### Niepoprawne

```text
TSS automatycznie przełącza RSP przy SYSCALL.
```

### Poprawne

```text
SYSCALL nie zmienia RSP.
Bursztyn OS przełącza stos jawnie w syscall.S
po SWAPGS, używając prywatnego stosu Ring 0 procesu.
```

---

### Niepoprawne

```text
BZL
```

jako aktualna nazwa polityki.

### Aktualne

```text
PZB — Poziom Zaufania Bursztyna
```

---

### Niepoprawne

```text
BWS gwarantuje absolutną szczelność systemu.
```

### Rzetelne

```text
BWS jest warstwową granicą bezpieczeństwa wykorzystującą
Ring 3, VMM, checked user-copy, PZB, PRAWO_*,
ownership i generacyjne uchwyty. Odporność implementacji
jest stale testowana i rozwijana.
```

---

# 6.51 Podsumowanie

BWS rozwinęło się z niewielkiego zestawu podstawowych syscalli do pełniejszego ABI Ring 3.

Aktualna ewolucja:

```text
1–26
podstawowy terminal/pliki/sieć/GUI
        ↓
27–60
procesy, eventy, warstwy GUI, heap,
drag&drop, clipboard, popup, blit, HPET
        ↓
61–83
geometria, monitor, SHM, IPC,
file watch, MIME clipboard, kursory, sockety
        ↓
84–85
OTA / A-B updates
        ↓
86–87
motywy
```

Najważniejsze właściwości współczesnego BWS:

- jawne publiczne numery w `bws_api.h`,
- kompatybilność addytywna,
- własne ABI rejestrów,
- prywatny stos Ring 0 procesu,
- `SWAPGS`,
- kontrolowane RFLAGS,
- checked user-copy,
- sprawdzanie overflow,
- PZB i `PRAWO_*`,
- generacyjne uchwyty,
- własność per PID,
- bounded allocations,
- cleanup procesu.

BWS nie jest jeszcze kompletnym odpowiednikiem dojrzałego ABI desktopowego OS. Najważniejsze przyszłe obszary to async I/O, równoległe sockety, pełne audio streams oraz dalsze testy destrukcyjne i wieloprocesowe.

---

## 6.52 Źródła i dokumenty powiązane

Najważniejsze źródła prawdy dla ABI:

1. `bws_api.h` — kanoniczna numeracja BWS,
2. `syscall.S` — rzeczywiste ABI rejestrów i brama `SYSCALL`,
3. `syscalls.cpp` — dispatcher BWS 1–60,
4. `bws_nowoczesne.h/.cpp` — BWS 61–87,
5. `motyw.h/.cpp` — ABI BWS 86–87,
6. `bws_ota.h` i podsystem aktualizacji — BWS 84–85,
7. SDK Ring 3 — wrappery publiczne.

Dokumenty powiązane:

- `02_architektura_systemu.md` — Ring 0/Ring 3, PZB i stos BWS,
- `04_audyt_gui_schedulera.md` — event queue i BWS 37–39,
- `05_hpet_i_pacing.md` — BWS 60,
- `07_ekosystem_i_formaty.md` — `.bur`, `.cebula`, manifesty,
- `11_audyt_bws_nowoczesne.md` — BWS 61–83,
- `aktualizacja-systemu.md` — BWS 84–85.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod,
2. `bws_api.h`,
3. `syscall.S`,
4. publiczne struktury ABI,
5. aktualne testy runtime,
6. najnowsze audyty,
7. starsza dokumentacja opisowa.
