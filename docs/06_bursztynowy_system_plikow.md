# 05. Bursztynowy System Plików — BSP2 / PSF

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** format BSP2/PSF, bloki, metadane, ścieżki, warstwa partycji, AHCI, tryb RAM-only, integracja z BWS/PZB oraz ograniczenia bieżącej implementacji  
**Status:** aktualny opis zastępujący starszy wariant dokumentacji z blokami 512 B, wolumenem 2 MiB i stałą bazą LBA

---

## 5.1 Nazwa i rola systemu plików

Aktualny system plików Bursztyn OS jest określany jako:

**BSP2 / PSF**

Nazwy te odnoszą się do **jednego systemu plików**, implementowanego w kodzie m.in. przez `psf.cpp` i `psf.h`. Nie należy przedstawiać BSP2 i PSF jako dwóch niezależnych formatów.

BSP2 jest autorskim, hierarchicznym systemem plików rozwijanym specjalnie dla Bursztyn OS.

Zapewnia podstawę dla:

- katalogów i plików,
- programów `.bur`,
- paczek `.cebula`,
- manifestów `opis.aplikacji`,
- ustawień systemowych,
- logów,
- zasobów aplikacji,
- danych użytkownika.

BSP2 nie jest oparty na FAT, ext, NTFS ani innym zewnętrznym formacie.

Jednocześnie Bursztyn OS wykorzystuje **FAT32 na partycji ESP**, ponieważ jest to osobna warstwa rozruchowa UEFI, a nie system plików danych Bursztyn OS.

---

## 5.2 Najważniejsze korekty względem starej dokumentacji

Starszy dokument zawierał kilka informacji historycznych, które nie opisują już aktualnego systemu.

### Nieaktualne

```text
blok BSP = 512 B
wolumen BSP = 2 MiB
stała baza BSP na konkretnym LBA
automatyczne formatowanie przy braku sygnatury
cały BSP mapowany pod 0x130000000ULL
```

### Aktualny stan

```text
blok BSP2 = 4096 B
partycja BSP2 instalatora = do 2 GiB w bieżącym wydaniu
I/O względem partycji: partition.start_lba + relative_lba
brak automatycznego formatowania fizycznego dysku przy zwykłym starcie
zwykły tryb Live używa BSP2 w trybie RAM-only
```

Rozmiaru **2 GiB** nie należy mylić ze strukturą formatu ani maksymalną pojemnością dysku. Jest to jawny limit bieżącego wydania instalatora.

---

## 5.3 Warstwy storage

Aktualną ścieżkę trwałego zapisu można przedstawić jako:

```text
aplikacja Ring 3
      ↓
BWS plikowe
      ↓
walidacja PZB / PRAWO_*
      ↓
BSP2 / PSF
      ↓
abstrakcja partycji
      ↓
SATA AHCI
      ↓
ATA LBA48
      ↓
dysk 512 B / 512e
```

BSP2 nie operuje bezpośrednio na absolutnych sektorach całego dysku.

Aktualna ścieżka I/O używa:

```text
partition.start_lba + relative_lba
```

Dzięki temu filesystem znajduje się wewnątrz swojej partycji GPT i nie zależy od historycznych stałych adresów dyskowych.

---

## 5.4 SATA AHCI

Aktualna trwała implementacja BSP2 korzysta z natywnego sterownika SATA AHCI.

Obsługiwane są obecnie:

- ATA,
- LBA48,
- DMA,
- odczyt sektorów,
- zapis sektorów,
- IDENTIFY DEVICE,
- ATA FLUSH w krytycznych ścieżkach instalatora,
- logiczny sektor 512 B,
- dyski 512e.

Obecny instalator nie obsługuje jako dysku docelowego:

- NVMe,
- IDE/ATA PIO,
- USB Mass Storage,
- Intel RST,
- Intel VMD,
- RAID,
- dysków 4Kn.

Brak obsługiwanego kontrolera SATA AHCI powoduje przerwanie instalacji bez wykonywania zapisu.

---

## 5.5 Sektor dysku a blok BSP2

Ważne jest rozdzielenie dwóch pojęć.

### Logiczny sektor urządzenia

Aktualnie:

```text
512 B
```

Dopuszczone są urządzenia 512e.

### Logiczny blok BSP2

Aktualnie:

```text
4096 B
```

Jeden blok BSP2 odpowiada więc logicznie wielu sektorom urządzenia 512 B.

Nie należy już pisać:

> „blok BSP ma 512 B i jest mapowany 1:1 na sektor AHCI”.

To opis historycznej wersji projektu.

---

## 5.6 Podstawowe właściwości formatu BSP2

Nowszy audyt kodu określa BSP2 następująco:

- blok danych: **4096 B**,
- superblok: **44 B** w audytowanej wersji,
- adresy bloków: **64-bitowe**,
- wskaźniki bezpośrednie,
- pojedynczy poziom pośredni,
- podwójny poziom pośredni.

Część starszych komentarzy w nagłówkach źródłowych była już podczas audytu oznaczona jako nieaktualna, dlatego dokumentacja powinna odwoływać się do rzeczywistego layoutu aktualnego `psf.h`, a nie do historycznych komentarzy.

---

## 5.7 Węzeł indeksowy

BSP2 korzysta ze struktury indeksowej opisującej plik lub katalog.

Koncepcyjnie węzeł przechowuje informacje takie jak:

- typ obiektu,
- rozmiar,
- wskaźniki do bloków,
- informacje potrzebne do odnalezienia danych.

Nazwa pliku jest wiązana z numerem/identyfikatorem węzła przez wpis katalogowy.

Nie należy jednak bez aktualnego `static_assert(sizeof(...))` deklarować w dokumentacji, że sam `wezel_indeksowy` ma dokładnie 4096 B.

Rozmiar bloku filesystemu i rozmiar struktury węzła to dwie różne rzeczy.

---

## 5.8 Adresowanie danych pliku

Audyt BSP2 dokumentuje trzy klasy wskaźników:

```text
direct
single indirect
double indirect
```

Dla audytowanego layoutu wyliczenie zakresu adresowania pliku wynosiło:

```text
(490 + 512 + 512²) * 4096
```

czyli:

```text
1 077 846 016 B
```

około 1,00 GiB danych adresowalnych przez ten układ wskaźników.

Jest to limit wynikający z bieżącego layoutu struktur, a nie z 2 GiB limitu partycji instalatora.

Dodatkowo audyt wskazywał, że publiczne rozmiary plików nadal używały pól 32-bitowych. Przed podniesieniem limitów woluminu lub pliku należy najpierw ujednolicić typy rozmiarów w całym ABI.

---

## 5.9 Wpis katalogowy

Katalog jest strukturą mapującą nazwę na obiekt filesystemu.

Koncepcyjny wpis:

```text
nazwa
→ identyfikator węzła
```

Pozwala to oddzielić:

- nazwę widoczną w ścieżce,
- metadane obiektu,
- fizyczne bloki danych.

Dokładny layout `teczka_wpis` pozostaje detalem bieżącego formatu `psf.h` i powinien być dokumentowany na podstawie aktualnego kodu, jeżeli format ma zostać zamrożony jako trwałe ABI dyskowe.

---

## 5.10 Ścieżki

BSP2 używa hierarchicznych ścieżek rozpoczynających się od `/`.

Przykład:

```text
/programy/notatnik.cebula/notatnik.bur
```

Parser wykonuje logicznie:

1. normalizację i walidację wejścia,
2. podział ścieżki na elementy,
3. start od katalogu głównego,
4. wyszukanie wpisu odpowiadającego pierwszemu komponentowi,
5. przejście do kolejnego węzła,
6. powtarzanie aż do obiektu końcowego.

Przykład:

```text
/
└── programy
    └── notatnik.cebula
        └── notatnik.bur
```

Dokumentacja nie powinna na stałe deklarować numeru konkretnego root inode bez potwierdzenia w aktualnym `psf.h` i formatterze.

---

## 5.11 Bezpieczeństwo parsera ścieżek

Ścieżka przekazana z Ring 3 jest danymi niezaufanymi.

Przed użyciem należy wykonać:

```text
copy_string_from_user_checked
      ↓
limit długości
      ↓
normalizacja
      ↓
walidacja komponentów
      ↓
polityka PZB / PRAWO_*
      ↓
operacja BSP2
```

Parser musi chronić się przed:

- zbyt długą ścieżką,
- brakiem terminatora,
- przepełnieniem długości,
- niepoprawnymi komponentami,
- wyjściem poza dozwolony namespace,
- użyciem surowego wskaźnika Ring 3 po zmianie kontekstu.

---

## 5.12 Struktura katalogów systemu

Bursztyn OS tworzy podczas przygotowania środowiska własną drabinę katalogów i instaluje wbudowane aplikacje, manifesty oraz zasoby.

Typowy logiczny układ obejmuje m.in.:

```text
/
├── programy/
│   ├── notatnik.cebula/
│   ├── kalkulator.cebula/
│   ├── ...
│
├── ustawienia/
├── logi/
├── uzytkownicy/
├── piaskownica/
├── tymczasowe/
└── zasoby systemowe / usługi zależnie od bieżącej wersji
```

Wewnętrzne ścieżki techniczne używają nazw bez polskich znaków diakrytycznych.

Nie należy jednak mylić drzewa BSP2 z zawartością partycji ESP.

---

## 5.13 Jądra A/B nie są zwykłymi plikami BSP2

Aktualny system aktualizacji A/B przechowuje obrazy rozruchowe w partycji ESP FAT32.

Przykładowe pliki:

```text
/boot/jadro_a.bin
/boot/jadro_b.bin
/boot/grub/grub.cfg
/boot/grub/grubenv
```

Dlatego katalog logiczny `/jadro` w BSP2 — jeśli występuje w bieżącej drabinie katalogów — nie jest autorytatywnym magazynem dwóch bootowalnych obrazów A/B.

Trzeba rozróżniać:

```text
ESP / FAT32
→ bootloader i obrazy startowe

BSP2
→ właściwe dane systemu i aplikacji
```

---

## 5.14 `.bur`

Plik `.bur` jest natywnym plikiem wykonywalnym programu Ring 3.

BSP2 przechowuje go jak zwykły plik danych, natomiast loader Bursztyn OS interpretuje jego format podczas uruchamiania procesu.

Przykład:

```text
/programy/notatnik.cebula/notatnik.bur
```

Filesystem nie powinien samodzielnie nadawać procesowi uprawnień tylko na podstawie rozszerzenia pliku.

---

## 5.15 `.cebula`

`.cebula` jest strukturą paczki aplikacji.

Przykładowo:

```text
/programy/notatnik.cebula/
├── opis.aplikacji
├── notatnik.bur
├── ikona...
└── zasoby...
```

Paczka może zawierać:

- program `.bur`,
- manifest,
- ikonę,
- zasoby aplikacji.

BSP2 zapewnia trwałe przechowywanie struktury katalogowej, natomiast politykę uruchomienia i uprawnień egzekwują loader, BWS i PZB.

---

## 5.16 `opis.aplikacji`

Manifest aplikacji opisuje m.in.:

- tożsamość aplikacji,
- deklarowane możliwości,
- wymagane prawa.

Manifest jest wejściem do polityki bezpieczeństwa, ale nie może samodzielnie przyznać aplikacji dowolnych uprawnień.

Ostateczna decyzja należy do jądra.

Koncepcyjnie:

```text
opis.aplikacji
      ↓
parser
      ↓
polityka systemowa
      ↓
PZB + PRAWO_*
      ↓
proces Ring 3
```

---

## 5.17 PZB i dostęp do plików

Aktualną nazwą mechanizmu bezpieczeństwa jest:

**PZB — Poziom Zaufania Bursztyna**

Nazwa **BZL** występująca w starszych dokumentach jest historyczna i nie powinna być używana w nowej dokumentacji.

Dostęp do filesystemu powinien uwzględniać jednocześnie:

```text
PZB
+
PRAWO_PLIKI_CZYTAJ
lub
PRAWO_PLIKI_ZAPISZ
+
politykę ścieżki
+
tożsamość procesu
```

---

## 5.18 Ochrona obszarów systemowych

Samo posiadanie:

```text
PRAWO_PLIKI_ZAPISZ
```

nie powinno oznaczać prawa do dowolnej modyfikacji systemu.

Chronione namespace mogą obejmować np.:

```text
/system
/sterowniki
```

oraz inne obszary wskazane przez aktualną politykę.

Decyzja bezpieczeństwa jest **logicznie egzekwowana przez jądro** podczas BWS i operacji filesystemu.

Nie należy opisywać ochrony katalogu jako „sprzętowej”, ponieważ CPU nie zna pojęcia ścieżki BSP2.

---

## 5.19 Piaskownica PZB-5

Model PZB przewiduje najbardziej ograniczoną klasę:

```text
PZB-5
```

Docelowy namespace piaskownicy obejmuje dedykowane dane aplikacji i przestrzeń tymczasową.

Historyczna dokumentacja używała przykładu:

```text
/piaskownica/[nazwa_aplikacji]
/tymczasowe
```

oraz blokady dostępu do danych innych użytkowników.

Ponieważ automatyczne, kernel-enforced provisioning namespace per aplikacja pozostaje nadal częściowo rozwijane, nie należy obecnie przedstawiać pełnej piaskownicy filesystemowej jako ukończonej i formalnie zweryfikowanej dla wszystkich ścieżek.

---

## 5.20 BWS plikowe

Aktualna mapa BWS zawiera między innymi:

| BWS | Operacja |
|---:|---|
| 2 | create file |
| 3 | write file |
| 5 | read file |
| 6 | list directory |
| 7 | delete object |
| 8 | rename object |
| 44 | file size |
| 46 | create directory |
| 47 | file metadata |
| 48 | move object |
| 52–54 | legacy file clipboard |
| 55 | copy object |
| 59 | ranged file read |
| 72 | file watch / unwatch |

BWS związane z plikami muszą stosować:

- checked user-copy,
- limity rozmiarów,
- kontrolę praw,
- politykę PZB,
- walidację ścieżek,
- propagację błędów.

---

## 5.21 File watch

BWS 72 udostępnia mechanizm obserwowania zmian plików.

Aktualny watcher posiada:

- uchwyt generacyjny,
- właściciela PID,
- maskę zdarzeń,
- ograniczoną kolejkę,
- ponowną kontrolę praw dla istniejącego watchera.

Mechanizm pozostaje częściowy — nie wszystkie mutacje wewnętrzne BSP2 muszą jeszcze emitować kompletny zestaw watcher events.

Nie należy więc przedstawiać go jako pełnego odpowiednika dojrzałych systemów typu inotify/FSEvents.

---

## 5.22 Partycja BSP2

Aktualny instalator tworzy układ:

```text
GPT
├── ESP FAT32 — 512 MiB
└── BSP2      — maks. 2 GiB w bieżącym wydaniu
```

Pozostałe miejsce na dysku pozostaje obecnie nieprzydzielone.

Partycje są wyrównywane do granicy 1 MiB.

Typ partycji BSP2 używany przez aktualne ABI instalatora:

```text
b21fe73c-0414-4585-bd07-8ab4980f9c88
```

GPT używa 64-bitowych LBA, więc limit 2 GiB wynika z bieżącego wydania BSP2/instalatora, a nie z samego GPT.

---

## 5.23 Zarezerwowane sektory stanu rozruchu

Bursztyn OS rezerwuje przed ESP:

```text
LBA 2046
LBA 2047
```

dla dwóch rekordów stanu rozruchu A/B.

Te sektory:

- nie należą do BSP2,
- nie są zwykłą wolną przestrzenią,
- nie mogą zostać użyte przez formatter BSP2,
- są walidowane pod kątem braku kolizji z GPT i partycjami.

ESP rozpoczyna się od:

```text
LBA 2048
```

w aktualnym układzie instalatora dla sektora 512 B.

---

## 5.24 Tryb Live a trwały BSP2

Zwykły start środowiska Live używa BSP2 w trybie **RAM-only**.

Zasada bezpieczeństwa po wdrożeniu instalatora:

> zwykły start Live nie formatuje automatycznie fizycznego dysku tylko dlatego, że nie znaleziono sygnatury BSP2.

Trwały zapis na docelowym dysku jest wykonywany przez kontrolowaną ścieżkę instalacji.

Tryb instalacyjny uruchamiany jest jawnie przez:

```text
bursztyn.install=1
```

i uruchamia:

```text
/programy/instalator.cebula/instalator.bur
```

---

## 5.25 Formatter

Formatter BSP2 nie powinien być uruchamiany przypadkowo podczas zwykłego bootu.

W aktualnej architekturze formatowanie dysku jest elementem kontrolowanej instalacji.

Przed wykonaniem destrukcyjnej operacji instalator sprawdza między innymi:

- wybrany dysk,
- port ATA,
- model urządzenia,
- numer seryjny,
- liczbę sektorów,
- rozmiar sektora logicznego,
- fingerprint urządzenia.

Między potwierdzeniem użytkownika i pierwszym zapisem tożsamość dysku jest sprawdzana ponownie.

Zmiana urządzenia powoduje przerwanie operacji.

---

## 5.26 Trwałość instalacji i flush

Aktualny instalator używa ścieżki zapisu z kontrolą trwałości.

Elementy obejmują:

- ATA FLUSH,
- odczyt kontrolny po krytycznych etapach,
- dwukopiowy znacznik stanu instalacji,
- numer sekwencji,
- CRC,
- stany `in-progress` i `complete`.

Niepełna instalacja nie powinna zostać zaakceptowana jako prawidłowy system.

To mechanizm instalatora i layoutu dysku, a nie journaling całego BSP2.

Nie należy z tego wyciągać wniosku, że sam BSP2 ma już pełny journal transakcyjny dla każdej operacji plikowej.

---

## 5.27 Cache i RAM-only

Historyczny audyt sprzed implementacji instalatora wykazał, że wcześniejszy cache danych miał 256 bloków i w trybie RAM-only mógł utracić brudne dane po wyrzuceniu cache entry.

Ponieważ dostępne dokumenty po implementacji instalatora nie opisują wystarczająco szczegółowo nowej wewnętrznej polityki cache BSP2, ten punkt nie powinien być przedstawiany jako definitywnie rozwiązany bez ponownego audytu aktualnego `psf.cpp`.

Rzetelny status:

> zwykły Live boot używa BSP2 RAM-only, ale szczegóły aktualnego cache, eviction i backing store wymagają synchronizacji dokumentacji z bieżącym kodem.

---

## 5.28 Propagacja błędów I/O

Audyt przed implementacją instalatora wykrył w starszym kodzie niebezpieczny wzorzec:

- błąd odczytu bloku mógł być zastępowany zerami,
- część ścieżek formattera nie propagowała poprawnie błędu flush.

Aktualny instalator ma bardziej rygorystyczne:

- ATA FLUSH,
- read-back verification,
- fail-closed przy zmianie tożsamości dysku.

Nie oznacza to automatycznie, że każda wewnętrzna ścieżka BSP2 ma już idealną propagację błędów.

Przed oznaczeniem filesystemu jako produkcyjnie odpornego na I/O fault należy ponownie przeprowadzić fault-injection audit całego `psf.cpp`.

---

## 5.29 Brak automatycznego zapisu w normalnym Live

Po wdrożeniu instalatora architektura została rozdzielona:

```text
normalny Live
→ brak automatycznego formatowania fizycznego dysku

tryb instalacyjny
→ jawnie zatwierdzony zapis na wybrane urządzenie
```

Jest to ważna korekta względem historycznego stanu, w którym brak sygnatury w określonym miejscu dysku mógł inicjować automatyczne formatowanie.

---

## 5.30 Ustawienia przechowywane w BSP2

System korzysta z plików BSP2 do trwałego przechowywania konfiguracji.

Przykładem jest:

```text
/ustawienia/ekran.cfg
```

Aplikacja ustawień ekranu zapisuje potwierdzoną konfigurację i wykonuje:

- `psf_flush_cache()`,
- odczyt weryfikujący.

Niepotwierdzona zmiana rozdzielczości nie powinna nadpisywać ostatniego zatwierdzonego pliku konfiguracyjnego.

---

## 5.31 Zasoby i tapety

Wbudowane aplikacje, manifesty, fonty i inne zasoby są instalowane przez systemowe ścieżki przygotowania drzewa BSP2.

Tapety wykorzystują własny format `.btp`.

Importer zasobów powinien:

- identyfikować zawartość,
- kontrolować integralność tam, gdzie przewidziano CRC,
- nie niszczyć istniejących danych tylko dlatego, że brakuje opcjonalnego modułu.

Stare ścieżki polegające na surowym odczycie danych z arbitralnego LBA należy traktować jako legacy, nie jako aktualny kontrakt BSP2.

---

## 5.32 Granice partycji

Każda operacja BSP2 na trwałym nośniku musi być ograniczona do:

```text
partition.start_lba
...
partition.end_lba
```

Nie wystarczy sprawdzenie względem całkowitego rozmiaru dysku.

Dla operacji:

```text
relative_lba + sector_count
```

należy sprawdzić overflow i przekroczenie końca partycji przed dodaniem `partition.start_lba`.

Zasada:

```text
najpierw walidacja względnego zakresu
dopiero potem translacja na LBA urządzenia
```

---

## 5.33 Bezpieczna arytmetyka

BSP2 operuje na rozmiarach, indeksach i adresach bloków pochodzących z metadanych lub żądań Ring 3.

Należy kontrolować operacje:

```text
block * BLOCK_SIZE
index * sizeof(entry)
offset + length
relative_lba + sector_count
file_size + append_size
```

Przepełnienie nie może prowadzić do:

- zapisu poza partycją,
- złej alokacji,
- złego bufora,
- nadpisania metadanych.

Preferowane są jawne helpery typu:

```cpp
checked_add(...)
checked_mul(...)
```

---

## 5.34 Integralność struktur

Przed użyciem struktur odczytanych z dysku należy walidować co najmniej:

- magic/version,
- rozmiary,
- liczbę bloków,
- zakresy wskaźników,
- granice partycji,
- typ obiektu,
- indeksy wpisów.

Nie wolno ufać metadanym tylko dlatego, że pochodzą z partycji oznaczonej jako BSP2.

Uszkodzony filesystem jest również niezaufanym wejściem dla kernela.

---

## 5.35 Zasada fail-closed

Przy błędzie krytycznej walidacji BSP2 bezpieczniejszą reakcją jest odmowa operacji niż kontynuowanie na niespójnym stanie.

Przykłady:

```text
zły blok
→ błąd

blok poza partycją
→ błąd

integer overflow
→ błąd

nieprawidłowy wpis katalogu
→ błąd

brak PRAWO_PLIKI_ZAPISZ
→ odmowa

błąd I/O podczas krytycznej instalacji
→ przerwanie instalacji
```

Nie należy automatycznie formatować nośnika jako reakcji na uszkodzony lub nieznany filesystem podczas zwykłego startu systemu.

---

## 5.36 Czego BSP2 obecnie nie deklaruje

Na podstawie aktualnych dokumentów nie należy przypisywać BSP2 funkcji, które nie zostały potwierdzone.

Nie ma podstaw, aby obecnie deklarować jako gotowe:

- pełny journaling wszystkich operacji plikowych,
- copy-on-write,
- snapshots,
- checksums każdego bloku danych,
- transparentną kompresję,
- szyfrowanie woluminu,
- RAID,
- online resize,
- bezpieczne zmniejszanie istniejących partycji,
- pełny POSIX filesystem semantics,
- pełną odporność na dowolną awarię zasilania.

Są to potencjalne przyszłe kierunki, nie obecne cechy formatu.

---

## 5.37 Aktualne ograniczenia

Najważniejsze ograniczenia bieżącej implementacji i instalatora:

- instalowana partycja BSP2 ma obecnie limit 2 GiB,
- publiczne rozmiary plików w audytowanej wersji pozostają 32-bitowe,
- bieżący layout adresowania pliku daje około 1,078 GB zakresu danych,
- storage instalatora wymaga SATA AHCI,
- logiczny sektor musi mieć 512 B / 512e,
- brak 4Kn,
- brak NVMe,
- brak USB Mass Storage,
- brak resize istniejących filesystemów,
- brak potwierdzonego pełnego journalingu,
- część szczegółów cache RAM-only wymaga ponownego audytu,
- pełna propagacja wszystkich klas błędów I/O wymaga ponownego audytu.

---

## 5.38 Testy

Audyt instalatora z 2026-09-09 raportował:

```text
4491 / 4491 PASS
```

dla swojej suity hostowej, obejmującej m.in.:

- instalację na pustym dysku,
- start UEFI bez ISO,
- reinstalację,
- ponowny start bez ISO,
- ochronę drugiego dysku,
- start hybrydowego ISO jako USB w środowisku testowym.

To potwierdza konkretną ścieżkę instalacyjną, ale nie oznacza pełnej walidacji BSP2 na wszystkich klasach sprzętu i błędów.

Status sprzętu fizycznego w tym audycie:

```text
NOT MANUALLY VERIFIED ON REAL HARDWARE
```

---

## 5.39 Testy wymagane dla BSP2

Przed uznaniem BSP2 za stabilny format produkcyjny należy posiadać co najmniej testy:

### Format i montowanie

```text
fresh format
mount existing
wrong magic
wrong version
truncated metadata
out-of-range block
```

### Pliki

```text
0 B
1 B
4095 B
4096 B
4097 B
direct boundary
single-indirect boundary
double-indirect boundary
maximum supported file
MAX + 1
```

### Katalogi

```text
empty directory
many entries
duplicate name
rename
move
delete non-empty
deep path
invalid component
```

### I/O fault injection

```text
read failure
write failure
flush failure
short operation
corrupted metadata
power loss during update
```

### Bezpieczeństwo

```text
PZB denial
missing PRAWO_PLIKI_CZYTAJ
missing PRAWO_PLIKI_ZAPISZ
bad Ring3 pointer
path overflow
path traversal attempt
sandbox namespace
```

### Partycja

```text
first block
last block
one-block-past-end
overflowed LBA
wrong start_lba
```

---

## 5.40 Kierunki dalszego rozwoju

Naturalne następne kroki BSP2 obejmują:

1. ujednolicenie wszystkich rozmiarów plików do 64 bitów;
2. ponowny audyt formatu on-disk i dodanie `static_assert` dla struktur;
3. wersjonowanie superbloku i struktur;
4. pełny fault-injection dla warstwy I/O;
5. formalny model cache RAM-only;
6. atomowe zastępowanie plików konfiguracyjnych;
7. rozszerzenie file watch;
8. przygotowanie pod większe partycje;
9. backend NVMe pod istniejącą abstrakcję block/partition;
10. docelowe mechanizmy integralności danych;
11. rozważenie journalingu lub innego mechanizmu crash consistency;
12. szyfrowanie danych użytkownika.

Rozszerzenia powinny zachowywać kompatybilność z istniejącym formatem tylko wtedy, gdy jest ona jawnie zaprojektowana i testowana.

---

## 5.41 Podsumowanie

Aktualny BSP2/PSF różni się znacząco od pierwszej wersji opisanej w starej dokumentacji.

Najważniejsze obecne właściwości:

```text
BSP2 / PSF = jeden system plików
blok = 4096 B
adresy bloków = 64-bit
direct + single indirect + double indirect
trwały backend = SATA AHCI
sektor urządzenia = 512 B / 512e
I/O = względem partycji GPT
instalowany wolumen = do 2 GiB w bieżącym wydaniu
normalny Live = RAM-only
brak automatycznego formatowania fizycznego dysku
```

Najważniejsze korekty względem starego dokumentu:

- 4096 B zamiast 512 B dla bloku BSP2,
- 2 GiB zamiast historycznego wolumenu 2 MiB,
- partycja GPT zamiast stałej absolutnej bazy LBA,
- BSP2 i PSF opisane jako jeden filesystem,
- rozdzielono sektor urządzenia od bloku filesystemu,
- usunięto stały adres VMM `0x130000000ULL` jako definicję całego BSP,
- zastąpiono nazwę BZL przez PZB,
- uwzględniono aktualne `.bur`, `.cebula`, BWS plikowe i file watch,
- rozdzielono BSP2 od ESP i obrazów A/B,
- dodano aktualny tryb Live RAM-only i bezpieczny model instalacji,
- dodano ograniczenia i obszary wymagające ponownego audytu zamiast deklarowania niepotwierdzonej pełnej stabilności.

---

## 5.42 Dokumenty powiązane

Aktualny stan BSP2 należy interpretować razem z:

- `02_architektura_systemu.md` — PZB i maski `PRAWO_*`,
- `04_zarzadzanie_sprzetem.md` — AHCI i ograniczenia storage,
- `06_wywolania_systemowe.md` — BWS plikowe,
- `07_ekosystem_i_formaty.md` — `.bur`, `.cebula`, `opis.aplikacji`,
- `11_audyt_bws_nowoczesne.md` — file watch i współczesne zabezpieczenia BWS,
- `instalator_audyt.md` — GPT, partycja BSP2 i bezpieczna instalacja,
- `aktualizacja-systemu.md` — rozdział ESP/BSP2 oraz stan A/B,
- `ustawienia_ekranu_audyt.md` — przykład trwałego zapisu `/ustawienia/ekran.cfg`.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny `psf.cpp` i `psf.h`,
2. aktualna abstrakcja partycji/storage,
3. bieżący kod instalatora,
4. aktualne testy,
5. nowsze audyty,
6. starsza dokumentacja historyczna.
