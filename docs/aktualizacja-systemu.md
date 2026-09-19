# Aktualizacja systemu Bursztyn OS

**Stan dokumentu:** aktualizacja 2026-09-19  
**Zakres:** aktualizacje A/B, ESP, BSP2, stan rozruchu, `grubenv`, rollback, pakiety `.akt`, weryfikacja kryptograficzna, BWS OTA i aplikacja „Aktualizacje Systemu”  
**Status:** mechanizm zaimplementowany i zweryfikowany w dedykowanych testach QEMU/host dla audytowanej wersji; **niezweryfikowany manualnie na realnym sprzęcie**

> **Zasada rzetelności:** ten rozdział oddziela stan potwierdzony przez kod i dedykowane testy podsystemu aktualizacji od wyników całego bieżącego systemu. Historyczne wyniki `140/140`, `52/52` i testy QEMU są snapshotem konkretnych rewizji. Po późniejszych zmianach BWS, GUI, BSP2 lub bootloadera należy ponownie uruchomić pełną suitę OTA przed określeniem bieżącego `HEAD` jako w pełni zweryfikowanego.

---

## 1. Cel mechanizmu A/B

Aktualizacje Bursztyn OS nie nadpisują bezpośrednio działającego slotu systemowego.

Podstawowa zasada:

```text
slot potwierdzony
      ↓
pozostaje nienaruszony

slot nieaktywny
      ↓
otrzymuje nową wersję
      ↓
jest oznaczany jako oczekujący
      ↓
rozruch próbny
      ↓
health checkpoint
      ├── sukces → zatwierdzenie
      └── błąd  → powrót do slotu potwierdzonego
```

Mechanizm ma ograniczyć ryzyko unieruchomienia systemu przez uszkodzony pakiet, niepełny zapis, utratę zasilania, wadliwe nowe jądro lub brak osiągnięcia punktu gotowości.

---

## 2. Aktualny układ dysku

Dla zainstalowanego systemu aktualna ścieżka rozruchowa jest **UEFI x86-64**.

```text
DYSK SYSTEMOWY
│
├── PMBR + GPT
│
├── LBA 2046
│   └── kopia 0 stanu rozruchu A/B
│
├── LBA 2047
│   └── kopia 1 stanu rozruchu A/B
│
├── ESP FAT32
│   ├── /EFI/BOOT/BOOTX64.EFI
│   ├── /boot/jadro_a.bin
│   ├── /boot/jadro_b.bin
│   ├── /boot/grub/grub.cfg
│   └── /boot/grub/grubenv
│
└── BSP2 / PSF
    ├── /sloty/a/
    ├── /sloty/b/
    ├── /aktualizacje/
    └── wspólne dane użytkownika
```

Aktualny instalator tworzy:

```text
ESP FAT32 = 512 MiB
BSP2      = maks. 2 GiB w bieżącym wydaniu
```

Pozostała przestrzeń dysku może pozostać nieprzydzielona. BSP2 używa logicznych bloków `4096 B`, natomiast aktualny storage instalatora obsługuje logiczne sektory urządzenia `512 B`, w tym 512e.

---

## 3. Ograniczenia platformy zainstalowanego systemu

Aktualna ścieżka instalacyjna wymaga:

```text
UEFI x86-64
SATA AHCI
ATA LBA48
sektor logiczny 512 B / 512e
```

Nieobsługiwane jako docelowy storage w obecnym wydaniu:

```text
NVMe
Intel RST
Intel VMD
RAID
USB Mass Storage
4Kn
```

Dodatkowo:

> **Secure Boot musi być wyłączony.**

`/EFI/BOOT/BOOTX64.EFI` nie jest obecnie podpisany w sposób wymagany przez Secure Boot. Bursztyn OS nie wymaga zmiennej startowej NVRAM, ponieważ korzysta ze standardowej ścieżki `/EFI/BOOT/BOOTX64.EFI`.

---

## 4. ESP i jądra A/B

Partycja ESP zawiera dwa niezależne obrazy jądra:

```text
/boot/jadro_a.bin
/boot/jadro_b.bin
```

oraz:

```text
/EFI/BOOT/BOOTX64.EFI
/boot/grub/grub.cfg
/boot/grub/grubenv
```

Obrazy A/B na ESP są oddzielone od drzew systemowych zapisanych w BSP2.

```text
ESP
→ kod przedjądrowy + jądra A/B

BSP2
→ system, programy, konfiguracja i dane
```

Aktualizacja nie powinna nadpisywać obrazu jądra aktualnie potwierdzonego slotu.

---

## 5. Sloty BSP2

Aktualna architektura aktualizacji używa logicznych drzew:

```text
/sloty/a/
/sloty/b/
```

dla wersjonowanych danych systemowych.

Przykładowo:

```text
/sloty/a/system/
/sloty/a/programy/

/sloty/b/system/
/sloty/b/programy/
```

Aktywny slot wpływa na translację ścieżek systemowych:

```text
aktywny A:
/system/...   → /sloty/a/system/...
/programy/... → /sloty/a/programy/...

aktywny B:
/system/...   → /sloty/b/system/...
/programy/... → /sloty/b/programy/...
```

Dane użytkownika powinny pozostać wspólne i niezależne od zmiany slotu, np. `/uzytkownicy/`, `/dokumenty/`, `/obrazy/` oraz inne przestrzenie określone przez aktualny layout jako dane trwałe użytkownika.

---

## 6. Autorytatywny stan rozruchu

Najważniejsza zasada architektury:

> **`grubenv` nie jest źródłem prawdy o zatwierdzeniu wersji systemu.**

Autorytatywny stan znajduje się w redundantnych rekordach:

```text
LBA 2046
LBA 2047
```

i jest lustrzany do:

```text
/aktualizacje/stan_rozruchu.k0
/aktualizacje/stan_rozruchu.k1
```

Rekord przechowuje m.in. slot potwierdzony, slot oczekujący, stan slotu oczekującego, licznik prób, limit prób, wersje, skróty SHA-256 jąder, numer generacji i CRC32.

---

## 7. Ważna uwaga o LBA 2046/2047

LBA 2046 i 2047 są **zarezerwowane przez specyfikację układu Bursztyn OS**, ale nie są obecnie osobną partycją GPT.

Aktualny układ dla sektorów 512 B:

```text
LBA 0        → PMBR
LBA 1        → primary GPT header
LBA 2..33    → primary GPT entries
LBA 34..2045 → obszar wyrównania
LBA 2046     → Bursztyn Boot State copy 0
LBA 2047     → Bursztyn Boot State copy 1
LBA 2048     → początek ESP
```

Narzędzia Bursztyn OS wiedzą, że `2046..2047` nie są wolną przestrzenią. Zewnętrzne narzędzie do partycjonowania może jednak traktować przestrzeń przed ESP jako nieprzydzieloną, ponieważ sektory te nie są reprezentowane osobnym wpisem GPT.

Dlatego przy przyszłej stabilizacji publicznego layoutu warto rozważyć dedykowaną partycję boot-state z własnym GUID. Obecny model pozostaje działającym kontraktem aktualnego instalatora i aktualizacji.

---

## 8. Walidacja układu przed użyciem A/B

Przed zapisaniem boot state należy zweryfikować, że:

```text
GPT entries kończą się przed LBA 2046
ESP rozpoczyna się >= LBA 2048
żadna partycja nie nachodzi na 2046..2047
istnieje ESP
istnieje partycja BSP2
```

Każdy konflikt powinien działać `fail-closed`:

```text
niepewny layout
→ brak zapisu boot state
→ brak aktywacji OTA
```

---

## 9. Rekord `bursztyn_rozruch_stan`

Audytowana wersja rekordu zajmuje `512 B` i jest chroniona przez:

```text
CRC32
+
walidację semantyczną
```

Rekord zawiera generację 64-bitową pozwalającą wybrać nowszą kopię.

Stany logiczne obejmują:

```text
NIEAKTYWNY
OCZEKUJACY
POTWIERDZONY
NIEUDANY
```

Sloty:

```text
A
B
NIEZNANY
```

Dokładne wartości magic, layout pól i `sizeof` powinny pozostać synchronizowane z aktualnym nagłówkiem źródłowym, jeżeli format zostanie zamrożony jako trwałe ABI dyskowe.

---

## 10. Walidacja semantyczna rekordu

Samo poprawne CRC nie wystarcza. Walidator powinien odrzucać m.in.:

```text
zły magic
nieobsługiwaną wersję
generacja == 0
nieznany identyfikator slotu
slot_oczekujacy == slot_potwierdzony
niespójny slot i stan
licznik_prob > maks_prob
niewłaściwy maks_prob
niedozwolony downgrade
brak SHA-256 oczekującego jądra
złe CRC32
```

Dzięki temu:

```text
poprawna suma kontrolna
≠
automatycznie poprawny stan logiczny
```

---

## 11. Wybór najlepszej kopii stanu

Odczyt analizuje obie kopie `LBA 2046` i `LBA 2047`.

Reguły:

1. jeżeli tylko jedna kopia jest poprawna — wybierz ją;
2. jeżeli obie są poprawne — wybierz nowszą generację;
3. przy równych generacjach zastosuj deterministyczne reguły fail-safe;
4. przy uszkodzeniu obu — wejście w kontrolowany stan awaryjny.

Dla porównywania generacji używana jest arytmetyka odporna na wrap licznika 64-bitowego.

---

## 12. Ping-pong write ordering

Stan nie powinien być zapisywany „naraz” do obu kopii.

```text
odczytaj obie kopie
      ↓
wybierz starszą / uszkodzoną
      ↓
zapisz nowy rekord
      ↓
ATA FLUSH
      ↓
read-back / weryfikacja
      ↓
zapisz drugą kopię
      ↓
ATA FLUSH
      ↓
read-back / weryfikacja
```

Dopiero później aktualizowane są kopie BSP2 i `grubenv`. Celem jest zachowanie co najmniej jednej poprawnej kopii przy utracie zasilania.

---

## 13. `grubenv`

Plik:

```text
/boot/grub/grubenv
```

jest 1024-bajtowym blokiem środowiska GRUB.

Zawiera pochodne dane potrzebne przed startem kernela:

```text
bursztyn_potwierdzony=A|B
bursztyn_oczekujacy=BRAK|A|B|NIEUDANY
bursztyn_proby=0|1
```

Jego rola to `bootloader scratchpad`, a nie autorytatywny magazyn polityki aktualizacji.

---

## 14. Dlaczego `grubenv` jest tylko stanem pochodnym

GRUB potrzebuje prostego formatu, który może odczytać i zmienić przed uruchomieniem jądra. Nie interpretuje pełnej struktury Bursztyn Boot State z LBA 2046/2047.

```text
LBA boot state
→ decyzja systemu / autoryzacja

grubenv
→ praktyczny wybór obrazu przed uruchomieniem jądra
```

Jeżeli oba źródła są niespójne, jądro traktuje boot-state jako nadrzędny i może naprawić `grubenv`.

---

## 15. Statyczna logika bootloadera

`BOOTX64.EFI` zawiera statyczną logikę GRUB potrzebną do wyboru slotu.

Skrypt:

```text
grub-embedded.cfg
```

nie powinien być traktowany jako magazyn dynamicznego stanu.

Dynamiczne są:

```text
grubenv
LBA boot state
```

a nie kod logiki bootloadera.

---

## 16. Próbny rozruch

Po instalacji aktualizacji:

```text
slot A = potwierdzony
slot B = oczekujący
```

przykładowy `grubenv`:

```text
bursztyn_potwierdzony=A
bursztyn_oczekujacy=B
bursztyn_proby=0
```

GRUB przed pierwszym startem B ustawia:

```text
bursztyn_proby=1
```

i zapisuje zmianę przed przekazaniem sterowania. Następnie ładuje `/boot/jadro_b.bin` z parametrem `bursztyn.slot=B`.

---

## 17. Limit prób

Aktualna polityka używa:

```text
maks_prob = 1
```

Oznacza to jedną próbę uruchomienia nowego slotu przed automatycznym wycofaniem.

Jeżeli system zostanie ponownie uruchomiony bez osiągnięcia health checkpointu:

```text
bursztyn_proby >= 1
```

GRUB wraca do potwierdzonego slotu.

---

## 18. Autoryzacja już uruchomionego kernela

Sam fakt, że bootloader załadował jądro, nie oznacza jeszcze, że jądro może kontynuować pełny start.

Wczesna logika aktualizacji porównuje:

```text
slot uruchomiony
```

z:

```text
slot_potwierdzony
slot_oczekujacy
stan_oczekujacego
```

z autorytatywnego boot-state.

Dozwolone są slot potwierdzony lub autoryzowany slot oczekujący. Inny przypadek jest błędem polityki rozruchu.

---

## 19. Nieautoryzowany start

Jeżeli `grubenv` został zmanipulowany lub jest przedawniony i uruchomi nieautoryzowany slot:

```text
GRUB uruchamia B
LBA mówi:
B nie jest potwierdzony
i nie jest oczekujący
```

jądro powinno:

```text
odrzucić dalszy start
naprawić grubenv
wskazać slot potwierdzony
wykonać reset
```

Nie należy uruchamiać normalnych procesów Ring 3 po wykryciu takiej niespójności.

---

## 20. Health checkpoint

Nowy slot nie jest potwierdzany natychmiast po wejściu do `kernel_main`.

Checkpoint następuje dopiero po osiągnięciu stanu uznanego przez system za operacyjny. Audytowany model obejmował m.in. działający PMM/VMM, dostęp do storage i BSP2, aktywny właściwy slot, działającą grafikę, uruchomiony Menedżer Okien, działającą powłokę, uruchomiony scheduler oraz stan „system gotowy”.

Dokładna kolejność tych warunków powinna pozostać zgodna z bieżącym kodem.

---

## 21. Potwierdzenie slotu

Po health checkpoint:

```text
slot_potwierdzony = aktywny
slot_oczekujacy   = NIEZNANY
licznik_prob      = 0
```

Aktualizowane są:

```text
LBA 2046
LBA 2047
/aktualizacje/stan_rozruchu.k0
/aktualizacje/stan_rozruchu.k1
/boot/grub/grubenv
```

oraz generacja i CRC rekordu.

Od tej chwili nowa wersja jest stabilnym punktem odniesienia dla kolejnej aktualizacji.

---

## 22. Rollback

Jeżeli próbny slot panikuje, resetuje się, traci zasilanie, nie osiąga checkpointu lub zostaje przerwany przed potwierdzeniem, to podczas następnego startu:

```text
GRUB widzi wykorzystaną próbę
→ uruchamia poprzedni slot potwierdzony
```

Poprzedni slot pozostaje nienaruszony.

Rollback jest celowo dozwolony nawet wtedy, gdy oznacza powrót do wersji starszej niż właśnie instalowana.

---

## 23. Downgrade a rollback

### Niedozwolony downgrade OTA

Pakiet:

```text
wersja_docelowa <= wersja_bieżąca
```

powinien być odrzucony jako nowa aktualizacja.

### Dozwolony rollback

Powrót do poprzedniego potwierdzonego slotu po nieudanym uruchomieniu nowej wersji jest podstawową funkcją bezpieczeństwa.

---

## 24. Pakiet aktualizacji `.akt`

Podsystem aktualizacji wykorzystuje własny format pakietu:

```text
.akt
```

oraz magic:

```text
BAKT
```

Pakiet przechodzi etap:

```text
download
→ weryfikacja
→ staging
→ commit do slotu nieaktywnego
→ aktywacja boot-state
```

Aktywny slot nie jest celem zwykłego procesu instalacji aktualizacji.

---

## 25. Manifest aktualizacji

Proces zaczyna się od podpisanego wykazu:

```text
wykazaktualizacji.wykaz
```

Sprawdzane są m.in. format, wersja, minimalna wersja, podpis i relacja `docelowa > bieżąca`.

Nieprawidłowy wykaz nie powinien prowadzić do żadnego zapisu w aktywnym systemie.

---

## 26. Pobieranie

Pakiet jest pobierany do kontrolowanej lokalizacji tymczasowej:

```text
/aktualizacje/pobieranie.tmp
```

Dopiero po pobraniu wykonywana jest pełna weryfikacja.

Utrata zasilania w tej fazie nie powinna zmieniać aktywnego slotu, potwierdzonego jądra ani boot-state.

---

## 27. Weryfikacja kryptograficzna

Mechanizm aktualizacji używa:

```text
SHA-256
RSA-2048
PKCS#1 v1.5
```

Weryfikowane są podpis wykazu, integralność pakietu zgodnie z formatem, SHA-256 pakietu, SHA-256 poszczególnych wpisów oraz SHA-256 jądra zapisanego na ESP.

Pakiet z błędnym podpisem lub hashem jest odrzucany.

---

## 28. Klucze RELEASE i TEST

Architektura oddziela:

```text
RELEASE
TEST
```

Build wydaniowy nie powinien akceptować pakietów podpisanych kluczem testowym.

Prywatny klucz wydania nie powinien znajdować się w repozytorium, obrazie systemu ani na publicznym serwerze aktualizacji. W obrazie kernela występuje wyłącznie materiał publiczny potrzebny do weryfikacji.

Dokładne odciski kluczy należy synchronizować z aktualnym buildem przed publikacją dokumentacji wydaniowej, zamiast traktować historyczny fingerprint jako wieczny kontrakt.

---

## 29. Staging

Zweryfikowane wpisy są przygotowywane w katalogu:

```text
/aktualizacje/przygotowanie/
```

Każdy zapis powinien być sprawdzony przez read-back i SHA-256 przed commit. Dopiero po poprawnym stagingu rozpoczyna się modyfikacja nieaktywnego slotu.

---

## 30. Commit do nieaktywnego slotu

Jeżeli:

```text
A = potwierdzony
```

to aktualizacja trafia do:

```text
B
```

Commit obejmuje:

```text
/sloty/b/...
/boot/jadro_b.bin
/sloty/b/system/wersja
```

Po zapisach wykonywane są odpowiednie flush/read-back. Slot A pozostaje punktem rollback.

---

## 31. Kolejność aktywacji

Bezpieczna kolejność:

```text
1. pobierz
2. zweryfikuj
3. staging
4. zapisz nieaktywny BSP2
5. zapisz nieaktywne jądro ESP
6. odczytaj i zweryfikuj
7. flush
8. zapisz autorytatywny boot-state LBA
9. zapisz kopie BSP2
10. na końcu ustaw grubenv
11. zażądaj restartu
```

Nie należy wskazywać w `grubenv` nowego slotu przed zakończeniem krytycznego commit.

---

## 32. Odporność na utratę zasilania

Jeżeli awaria nastąpi podczas downloadu, weryfikacji, stagingu lub commit do nieaktywnego slotu, potwierdzony slot pozostaje źródłem normalnego rozruchu.

Podczas zapisu boot-state dwie kopie i generacja pozwalają wybrać poprawny rekord po torn write. Podczas zapisu `grubenv` autorytatywny LBA state pozwala wykryć niezgodność przy kolejnym uruchomieniu.

---

## 33. `grubenv` uszkodzony lub brakujący

Aktualny model posiada fallback i mechanizm samonaprawy.

Jeżeli GRUB uruchomi bezpieczny slot domyślny, jądro po odczytaniu autorytatywnego stanu może odbudować `grubenv`.

Nie należy jednak traktować samego fallbacku `A` jako dowodu, że A zawsze jest najnowszym potwierdzonym slotem — o tym decyduje stan autorytatywny.

---

## 34. Aplikacja „Aktualizacje Systemu”

System posiada natywną aplikację Ring 3:

```text
/programy/aktualizacje.cebula/aktualizacje.bur
```

Aplikacja prezentuje bieżącą wersję, aktywny slot, slot docelowy, dostępną wersję, kanał, rozmiar pakietu, stan podpisu, stan SHA-256, postęp, potrzebę restartu i informację o rollbacku.

Interfejs korzysta z UTF-8 i BursztynUI.

---

## 35. Ścisły przepływ GUI

Model aplikacji:

```text
Sprawdź
   ↓
Pobierz
   ↓
Zainstaluj
   ↓
Uruchom ponownie
```

Każdy przycisk powinien być aktywny tylko w dozwolonym stanie automatu.

Przykładowe polecenia:

```text
BWS_OTA_CMD_SPRAWDZ
BWS_OTA_CMD_POBIERZ
BWS_OTA_CMD_INSTALUJ
BWS_OTA_CMD_REBOOT
```

Podsystem nie powinien automatycznie wykonać restartu tylko dlatego, że pakiet został pobrany.

---

## 36. Powiadomienie o rollbacku

Jeżeli poprzednia próba nie powiodła się, stan OTA może zwrócić:

```text
rollback_wykryty = 1
```

GUI powinno wyświetlić jednoznaczne ostrzeżenie, np.:

> **Wykryto automatyczny powrót po nieudanym rozruchu. Przywrócono potwierdzoną wersję systemu.**

Taki komunikat powinien zawierać stan, ale nie ukrywać faktu nieudanej próby aktualizacji.

---

## 37. BWS OTA

Publiczne BWS aktualizacji:

```c
#define BWS_OTA_STATUS     84
#define BWS_OTA_POLECENIE  85
```

BWS 84 kopiuje stan OTA do kontrolowanego bufora Ring 3. BWS 85 wykonuje dozwolone polecenie aktualizacji.

### Aktualna numeracja BWS

BWS 84–85 nie są już końcem całego ABI projektu. W późniejszym rozwoju pojawiły się również BWS 86–87 związane z motywami.

> **`bws_api.h` jest kanonicznym źródłem aktualnej numeracji BWS.**

Nie należy w nowej dokumentacji pisać, że ABI kończy się na 85.

---

## 38. `BwsOtaStatus`

Audytowana struktura:

```text
BwsOtaStatus
```

ma:

```text
384 B
```

i zawiera m.in.:

```text
wersja_abi
stan
kod_bledu
postep_procent
restart_wymagany
aktywny_slot
nieaktywny_slot
rollback_wykryty
podpis_ok
sha256_ok
rozmiar_pakietu
biezaca_wersja
dostepna_wersja
kanal
odcisk_klucza
komunikat
CRC32
```

Publiczne ABI powinno zachować:

```cpp
static_assert(sizeof(BwsOtaStatus) == 384);
```

dopóki wersja ABI 1 jest wspierana. Przyszłe rozszerzenia powinny korzystać z wersjonowania zamiast cichej zmiany layoutu.

---

## 39. Checked user-copy

Wskaźniki Ring 3 przekazane do BWS OTA są niezaufane.

Przed dostępem kernel powinien sprawdzić kanoniczność, pełny zakres, mapowanie, prawa zapisu/odczytu i overflow.

Dane publiczne są kopiowane przez controlled user-copy. Kernel nie powinien przechowywać surowego pointera aplikacji do późniejszego użycia.

---

## 40. PZB i prawa

Operacje aktualizacji są wrażliwe i wymagają zaufanej aplikacji systemowej.

Kluczowe prawa obejmują m.in.:

```text
PRAWO_SIEC
PRAWO_PLIKI_CZYTAJ
PRAWO_PLIKI_ZAPISZ
PRAWO_SYSTEM_CONFIG
```

### Korekta dokumentacyjna

Nie należy w tym rozdziale na stałe wiązać aplikacji aktualizacji z:

```text
PZB_USLUGI = poziom 2
```

bez potwierdzenia aktualnego enumu PZB w kodzie. Starsze dokumenty projektu nie są w pełni spójne co do nazwy funkcjonalnej poziomu PZB-2.

Poprawna zasada brzmi:

> **Aplikacja aktualizacji otrzymuje zaufaną tożsamość systemową nadaną przez kernel oraz niezbędne prawa, w szczególności `PRAWO_SYSTEM_CONFIG`.**

Manifest sam nie może nadać sobie takich uprawnień.

---

## 41. Restart

Polecenie restartu jest operacją uprzywilejowaną.

Powinno być dostępne dopiero po:

```text
pełnym commit
+
poprawnym boot-state
+
poprawnym grubenv
+
flush storage
```

Restart nie może maskować błędu zapisu. Jeżeli aktywacja nie została ukończona, system powinien pozostać w działającym potwierdzonym slocie.

---

## 42. Aktualizacje a synchroniczna sieć

Aktualny stos HTTP/HTTPS/TLS posiada nadal istotne ścieżki synchroniczne Ring 0.

Oznacza to, że etap:

```text
Sprawdź / Pobierz
```

może przy długiej operacji sieciowej czasowo pogorszyć responsywność GUI.

Nie jest to błąd modelu A/B, lecz ograniczenie obecnej architektury sieci.

Docelowo pobieranie OTA powinno przejść w kierunku:

```text
request
→ WAITING_IO
→ worker/state machine
→ progress events
→ completion
```

zamiast długiego synchronicznego BWS.

---

## 43. Aktualizacje a BSP2

BSP2 i PSF to jeden system plików.

Aktualny filesystem używa:

```text
blok 4096 B
```

i I/O względem:

```text
partition.start_lba + relative_lba
```

Nie należy stosować historycznych stałych baz LBA do danych slotów. System aktualizacji musi respektować granice partycji BSP2.

---

## 44. Dane użytkownika

Aktualizacja systemowa nie powinna nadpisywać danych użytkownika przechowywanych poza slotowanym system namespace.

Testy powinny kontrolować co najmniej:

```text
/uzytkownicy
/dokumenty
/obrazy
```

i inne współdzielone katalogi zgodnie z aktualnym layoutem.

Ważny invariant:

```text
zmiana A ↔ B
≠
migracja / kasowanie danych użytkownika
```

---

## 45. Testy parsera i pakietu

Historyczny audyt dokumentuje osobne testy:

```text
make test-aktualizacje
```

obejmujące m.in. parser, podpis RSA, SHA-256, traversal, integralność wpisów i scenariusze sieciowe.

Wartości liczbowe z danego audytu należy traktować jako wynik tej konkretnej rewizji kodu.

---

## 46. Testy automatu rozruchu

Audytowana suita:

```text
make test-rozruch
```

raportowała:

```text
52 / 52 PASS
```

Obejmowała m.in. instalację A→B, B→A, potwierdzanie, rollback, korupcję jednej i dwóch kopii, power loss, downgrade rejection, split-write, generacje, `grubenv`, semantycznie niepoprawne rekordy i nienaruszalność danych użytkownika.

To jest test podsystemu boot-state, nie dowód, że wszystkie pozostałe podsystemy bieżącego Bursztyn OS są wolne od regresji.

---

## 47. QEMU UEFI

Dedykowany test:

```text
make test-rozruch-qemu
```

w audytowanej wersji wykonywał pełny scenariusz UEFI:

```text
instalacja
→ Slot A
→ aktualizacja B
→ próbny start B
→ checkpoint
→ trwały B
→ kontrolowana awaria
→ rollback
→ split-write
→ self-healing
→ unauthorized boot rejection
```

To jest najważniejszy test integracyjny mechanizmu A/B.

---

## 48. Wyniki instalatora

Osobny audyt instalatora z 2026-09-09 raportował:

```text
4491 / 4491 PASS
```

dla testów hostowych i scenariuszy QEMU, obejmujących m.in. instalację na pustym dysku, UEFI boot bez ISO, reinstalację, kolejny start i ochronę drugiego dysku.

Status:

```text
NOT MANUALLY VERIFIED ON REAL HARDWARE
```

Ten status pozostaje istotny także dla aktualizacji A/B, ponieważ mechanizm zależy od tej samej warstwy storage i boot.

---

## 49. Wynik `140/140`

Dokument aktualizacji raportował:

```text
140 / 140 HOST UNIT TESTS PASSING
make test-ota
```

oraz:

```text
ZERO COMPILER WARNINGS
-Wall -Wextra -Wpedantic
```

### Aktualna interpretacja

Te wyniki należy opisywać jako:

> **wynik audytowanej rewizji podsystemu OTA**

A nie jako automatyczną gwarancję obecnego całego drzewa źródłowego.

Bieżący projekt posiada późniejsze zmiany m.in. w BWS i GUI oraz osobne regresje innych podsystemów. Dlatego przed wydaniem należy ponownie uruchomić:

```text
make test-ota
make test-rozruch
make test-rozruch-qemu
```

na aktualnym `HEAD`.

---

## 50. Minimalny release gate OTA

Przed oznaczeniem konkretnego buildu jako gotowego do aktualizacji OTA powinny przejść:

```text
1. build bez nowych warnings
2. parser .akt
3. RSA/SHA tests
4. boot-state unit tests
5. disk-layout tests
6. installer tests
7. UEFI QEMU A/B test
8. rollback test
9. unauthorized boot test
10. split-write/power-loss simulation
11. stale grubenv recovery
12. user-data invariant
```

Dla wydania na fizyczny sprzęt dodatkowo:

```text
manual real-hardware verification
```

---

## 51. Brak Secure Boot

Istotne rozróżnienie:

```text
pakiety OTA są weryfikowane kryptograficznie
```

nie oznacza:

```text
platforma ma działający Secure Boot
```

Aktualnie:

```text
OTA:
RSA-2048 + SHA-256

UEFI Secure Boot:
nieobsługiwany
```

Użytkownik musi mieć Secure Boot wyłączony.

---

## 52. Granica zaufania

A/B pomaga chronić przed wadliwą aktualizacją, uszkodzonym zapisem i niepełnym bootem. Nie zastępuje wszystkich mechanizmów bezpieczeństwa platformy.

W szczególności obecny system:

- nie ma Secure Boot,
- nie posiada sprzętowego measured boot,
- nie deklaruje TPM-based attestation,
- nadal ufa własnemu kernelowi i bootloaderowi zapisanym na dysku.

Dokumentacja nie powinna utożsamiać podpisu OTA z pełnym hardware root of trust.

---

## 53. Zalecany przyszły rozwój

Najważniejsze dalsze kierunki:

```text
dedykowana partycja GPT dla boot-state
Secure Boot / podpisany bootloader
bezpieczny key rotation dla OTA
wersjonowanie formatu .akt
async download
resume pobierania
delta updates
per-file atomic replacement
pełniejsze lokalne logi update failure
manual testy fizycznego sprzętu
```

Dodatkowo warto rozważyć osobne wersjonowanie BSP2 slotów, migracje danych kompatybilne z rollback, limit maksymalnej liczby nieudanych wersji i narzędzie diagnostyczne do odczytu obu rekordów boot-state.

---

## 54. Aktualny status funkcji

| Element | Stan |
|---|---|
| Dwa sloty A/B | **zaimplementowane** |
| Jądra A/B na ESP | **zaimplementowane** |
| Sloty systemowe w BSP2 | **zaimplementowane według dokumentacji aktualizacji** |
| LBA 2046/2047 boot-state | **zaimplementowane** |
| CRC32 + generacja | **zaimplementowane** |
| `grubenv` | **zaimplementowany** |
| Rozruch próbny | **zaimplementowany** |
| Health checkpoint | **zaimplementowany** |
| Automatyczny rollback | **zweryfikowany w QEMU dla audytowanej rewizji** |
| Self-healing `grubenv` | **zweryfikowany w QEMU dla audytowanej rewizji** |
| Odrzucenie nieautoryzowanego slotu | **zweryfikowane w QEMU dla audytowanej rewizji** |
| RSA-2048 + SHA-256 OTA | **zaimplementowane** |
| BWS 84–85 | **zaimplementowane** |
| Aplikacja Ring 3 „Aktualizacje Systemu” | **zaimplementowana** |
| Secure Boot | **nieobsługiwany** |
| Real hardware OTA | **NOT MANUALLY VERIFIED** |
| Async OTA download | **nieukończony / obecna sieć ma ścieżki synchroniczne** |
| Dedykowana partycja GPT boot-state | **planowana / do rozważenia** |

---

## 55. Najważniejsze korekty względem poprzedniej wersji

W tej aktualizacji dokumentu:

1. zachowano autorytatywny model LBA 2046/2047;
2. zachowano `grubenv` jako pochodny scratchpad;
3. zachowano A/B i jeden próbny boot;
4. dodano aktualny kontekst instalatora: UEFI, AHCI, 512/512e, ESP 512 MiB, BSP2 do 2 GiB;
5. dodano jawne ograniczenie Secure Boot;
6. doprecyzowano, że LBA 2046/2047 nie są osobną partycją GPT;
7. usunięto zbyt absolutne stwierdzenie, że testy historyczne automatycznie opisują aktualny `HEAD`;
8. BWS 84–85 pozostają OTA, ale dokument nie twierdzi już, że całe ABI kończy się na 85;
9. usunięto niepewne przypisanie aplikacji do `PZB_USLUGI = 2` jako stałego kontraktu;
10. dodano checked user-copy i granicę zaufania Ring 3;
11. dodano wpływ synchronicznego stosu sieciowego na responsywność pobierania aktualizacji;
12. oddzielono kryptograficzną weryfikację OTA od Secure Boot;
13. dodano release gate wymagający ponownego uruchomienia testów na aktualnym kodzie.

---

## 56. Podsumowanie

Aktualny model aktualizacji Bursztyn OS:

```text
sprawdź wykaz
      ↓
zweryfikuj podpis
      ↓
pobierz .akt
      ↓
SHA-256 / RSA
      ↓
staging
      ↓
zapis nieaktywnego slotu BSP2
      ↓
zapis nieaktywnego jądra na ESP
      ↓
read-back + flush
      ↓
LBA 2046/2047
      ↓
grubenv
      ↓
restart
      ↓
próbny slot
      ↓
health checkpoint
      ├── PASS → zatwierdź
      └── FAIL → rollback
```

Najważniejszymi właściwościami projektu są działający model A/B, redundantny boot-state, deterministyczny wybór kopii, rollback, self-healing `grubenv`, weryfikacja RSA-2048/SHA-256, aplikacja Ring 3 i BWS OTA 84/85.

Najważniejsze ograniczenia to brak Secure Boot, brak manualnej weryfikacji real hardware, storage instalacyjny tylko SATA AHCI 512/512e, boot-state poza osobną partycją GPT, synchroniczne ścieżki sieciowe oraz konieczność ponawiania historycznych testów na aktualnym `HEAD`.

---

## 57. Dokumenty powiązane

Aktualny stan powinien być interpretowany razem z:

- `03_proces_rozruchu.md` — UEFI/GRUB/Multiboot2 i A/B,
- `05_bursztynowy_system_plikow.md` — BSP2/PSF,
- `06_wywolania_systemowe.md` — BWS, w tym OTA 84/85,
- `07_ekosystem_i_formaty.md` — aplikacja `.cebula`,
- `10_siec.md` — HTTP/HTTPS/TLS,
- `instalator_audyt.md` — GPT, ESP, AHCI i ograniczenia sprzętowe.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod boot-state i aktualizacji,
2. `bws_api.h`,
3. aktualny skrypt GRUB/`grubenv`,
4. bieżący kod instalatora i layoutu GPT,
5. aktualny BSP2,
6. najnowsze testy runtime,
7. dokumentacja audytowa.

---

## 58. Testy wymagane przed publicznym wydaniem

Minimalna końcowa macierz:

```text
fresh install
boot A
install B
boot B
confirm B
reboot B
update back to A
confirm A

power loss:
- during download
- during BSP2 staging
- during ESP kernel write
- between LBA copies
- between LBA and grubenv
- during health confirmation

corruption:
- LBA 2046
- LBA 2047
- both LBA
- grubenv missing
- grubenv truncated
- grubenv stale
- wrong kernel SHA
- wrong package SHA
- wrong RSA signature

security:
- TEST key on RELEASE
- downgrade attempt
- unauthorized slot boot
- Ring3 bad pointer to BWS84
- missing PRAWO_SYSTEM_CONFIG for BWS85
- path traversal in .akt

invariants:
- confirmed slot unchanged until activation
- user data unchanged
- second disk unchanged
- rollback returns to bootable confirmed slot
```

Dopiero po przejściu tej macierzy na aktualnym kodzie można przypisać wynik do konkretnego wydania Bursztyn OS.
