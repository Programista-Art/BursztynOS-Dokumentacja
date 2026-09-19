# Audyt instalatora Bursztyn OS

**Stan dokumentu:** aktualizacja 2026-09-19  
**Pierwotny audyt:** 2026-09-09  
**Zakres:** instalacja UEFI, SATA AHCI, GPT, ESP FAT32, BSP2/PSF, tryb Live, bezpieczeństwo wyboru dysku, odporność na awarię zasilania oraz integracja z aktualizacjami A/B  
**Status sprzętu fizycznego:** **NOT MANUALLY VERIFIED ON REAL HARDWARE**  
**Status testów:** dedykowany audyt instalatora z 2026-09-09 raportował `4491/4491 PASS`; po późniejszym dodaniu pełniejszego mechanizmu A/B/OTA wynik ten należy traktować jako historyczny wynik konkretnej rewizji i ponowić przed wydaniem

---

## 1. Cel dokumentu

Pierwotna wersja tego audytu opisywała zarówno stan **przed implementacją instalatora**, jak i pierwszą działającą wersję instalacji na GPT.

Od tego czasu architektura Bursztyn OS została rozszerzona o:

- trwały model instalacji UEFI,
- BSP2 działający względem partycji,
- bezpieczny tryb Live bez automatycznego formatowania dysku,
- dwukopiowy znacznik instalacji,
- aktualizacje systemowe A/B,
- dwa obrazy jądra na ESP,
- autorytatywny stan rozruchu w LBA 2046/2047,
- `grubenv` jako pochodny stan bootloadera,
- BWS OTA,
- aplikację `Aktualizacje Systemu`.

Dlatego wcześniejszy rozdział „Audyt **przed** implementacją instalatora” należy obecnie traktować jako materiał historyczny.

Aktualny dokument opisuje **stan po implementacji i późniejszym rozwoju architektury rozruchu**.

---

# 2. Podsumowanie aktualnego stanu

Aktualna pierwsza obsługiwana konfiguracja instalacyjna:

```text
architektura          = x86-64
tryb firmware         = UEFI
kontroler dysku       = SATA AHCI
protokół ATA          = LBA48
sektor logiczny       = 512 B / 512e
tablica partycji      = GPT
partycja rozruchowa   = ESP FAT32
filesystem systemowy  = BSP2 / PSF
Secure Boot           = musi być wyłączony
NVMe                  = nieobsługiwane jako target
4Kn                   = nieobsługiwane
USB Mass Storage      = nieobsługiwane jako target
Intel RST/VMD/RAID    = nieobsługiwane
```

Instalator działa jako aplikacja Ring 3:

```text
/programy/instalator.cebula/instalator.bur
```

uruchamiana wyłącznie w jawnym trybie instalacyjnym:

```text
bursztyn.install=1
```

---

# 3. Zwykły Live boot a tryb instalacyjny

Najważniejsza korekta względem bardzo starej architektury:

> **Normalny start Live nie wykonuje automatycznego formatowania fizycznego dysku.**

Zwykły Live:

```text
boot
→ BSP2 RAM-only
→ brak destrukcyjnych zapisów instalacyjnych
```

Tryb instalacyjny:

```text
boot z bursztyn.install=1
→ uruchom Instalator Bursztyn OS
→ wybór i potwierdzenie dysku
→ ponowna walidacja urządzenia
→ dopiero wtedy pierwszy zapis
```

Historyczny mechanizm:

```text
brak sygnatury na stałym LBA
→ automatyczny format
```

został usunięty ze ścieżki normalnego działania.

---

# 4. Obsługiwany storage

## 4.1 SATA AHCI

Aktualna instalacja wymaga obsługiwanego kontrolera:

```text
SATA AHCI
```

Sterownik używa:

- ATA,
- LBA48,
- DMA,
- IDENTIFY DEVICE,
- odczytu,
- zapisu,
- ATA FLUSH.

W przypadku braku obsługiwanego AHCI instalator powinien zakończyć ścieżkę destrukcyjną komunikatem w rodzaju:

```text
Nie wykryto obsługiwanego kontrolera SATA AHCI.
```

i wykonać:

```text
0 zapisów
```

na dysku.

---

# 5. Nieobsługiwane kontrolery i nośniki

Aktualny installer target **nie obsługuje**:

```text
NVMe
IDE / ATA PIO
USB Mass Storage
Intel RST
Intel VMD
RAID
4Kn
```

### NVMe

PCI może wykryć urządzenie klasy NVMe, ale nie istnieje jeszcze backend blokowy umożliwiający instalację.

### IDE

Nazwa urządzenia QEMU typu `ide-hd` nie oznacza, że Bursztyn OS posiada sterownik kontrolera IDE.

### USB

Aktualny USB/xHCI obsługuje przede wszystkim HID.

Nie ma jeszcze stosu:

```text
USB BOT / UAS
→ SCSI
→ block device
```

potrzebnego do instalacji na pamięci USB.

---

# 6. Sektory 512 B a bloki BSP2

Należy rozdzielić:

```text
logiczny sektor urządzenia
```

od:

```text
blok filesystemu BSP2
```

Aktualnie:

```text
sektor AHCI = 512 B
BSP2 block   = 4096 B
```

Urządzenia 512e są dopuszczone.

Urządzenia z logicznym sektorem:

```text
4096 B — 4Kn
```

są obecnie jawnie odrzucane przez instalator.

---

# 7. BSP2 / PSF

BSP2 i PSF to **jeden system plików**, a nie dwa różne formaty.

Aktualne potwierdzone właściwości:

```text
blok filesystemu = 4096 B
superblok         = 44 B w audytowanym layoutcie
adresy bloków     = 64-bit
direct
single indirect
double indirect
```

Historyczne informacje o blokach 512 B są nieaktualne.

---

# 8. Partycjonowanie zamiast stałego LBA

Stary BSP2 używał historycznej stałej bazy:

```text
LBA 40000
```

oraz części legacy związanych z surowym:

```text
LBA 10
```

Aktualna ścieżka instalacyjna używa abstrakcji partycji:

```text
physical_lba
=
partition.start_lba + relative_lba
```

Filesystem musi walidować zakres względem własnej partycji, a nie względem samego końca całego dysku.

---

# 9. GPT

Instalator implementuje GPT zawierające:

- Protective MBR,
- główny nagłówek GPT,
- główną tablicę wpisów,
- zapasową tablicę wpisów,
- zapasowy nagłówek GPT,
- CRC32,
- read-back verification.

GPT używa 64-bitowych LBA.

Limit bieżącego BSP2/instalatora nie wynika więc z ograniczenia samego GPT.

---

# 10. Typ partycji Bursztyn OS

Aktualny identyfikator typu partycji BSP2 używany przez ABI instalatora:

```text
b21fe73c-0414-4585-bd07-8ab4980f9c88
```

Jeżeli GUID ma pozostać publiczną częścią layoutu dysku, powinien zostać traktowany jako stabilny kontrakt i chroniony testami.

---

# 11. Aktualny układ pełnego dysku

Podstawowy układ po instalacji:

```text
LBA 0
└── PMBR

LBA 1..33
└── Primary GPT

LBA 34..2045
└── obszar wyrównania

LBA 2046
└── Bursztyn Boot State — kopia 0

LBA 2047
└── Bursztyn Boot State — kopia 1

LBA 2048...
└── ESP FAT32 — 512 MiB

następnie
└── BSP2 — maksymalnie 2 GiB w bieżącym wydaniu

koniec dysku
└── Backup GPT

pozostała przestrzeń
└── nieprzydzielona
```

Partycje są wyrównywane do granicy:

```text
1 MiB
```

---

# 12. Ważna aktualizacja: LBA 2046 i 2047

Pierwszy audyt instalatora powstał przed pełnym rozwojem aktualizacji A/B.

Aktualny Bursztyn OS rezerwuje:

```text
LBA 2046
LBA 2047
```

dla redundantnego stanu rozruchu.

Sektory te:

- nie należą do ESP,
- nie należą do BSP2,
- nie mogą być używane przez formatter,
- muszą pozostać poza deklarowanymi partycjami,
- są walidowane przed uruchomieniem A/B.

Instalator musi upewnić się, że:

```text
GPT array end < 2046
ESP start >= 2048
żadna partycja nie nachodzi na 2046..2047
```

---

# 13. Ograniczenie obecnego modelu sektorów boot-state

LBA 2046/2047 są zarezerwowane przez layout Bursztyn OS, ale nie stanowią osobnej partycji GPT.

Dla własnego instalatora jest to kontrolowane.

Zewnętrzne narzędzie partycjonujące może jednak interpretować ten fragment jako część nieprzydzielonego obszaru przed ESP.

Dlatego przed stabilnym publicznym wydaniem warto rozważyć:

```text
małą dedykowaną partycję GPT
dla Boot State
```

z własnym GUID.

Obecny model pozostaje aktualnym kontraktem systemu i nie powinien być zmieniany bez migracji formatu.

---

# 14. ESP FAT32

ESP ma obecnie:

```text
512 MiB
```

i zawiera co najmniej:

```text
/EFI/BOOT/BOOTX64.EFI
/boot/jadro_a.bin
/boot/jadro_b.bin
/boot/grub/grub.cfg
/boot/grub/grubenv
```

Standardowa ścieżka:

```text
/EFI/BOOT/BOOTX64.EFI
```

pozwala uruchomić zainstalowany system bez tworzenia własnej zmiennej bootującej UEFI NVRAM.

---

# 15. Instalowany system a hybrydowe ISO

Należy odróżniać dwie rzeczy.

## Live / ISO

Obraz ISO jest hybrydowy i historycznie zawiera ścieżki:

```text
BIOS El Torito
UEFI El Torito
```

## System zainstalowany na dysku

Aktualna pierwsza ścieżka instalacyjna jest:

```text
UEFI x86-64
```

z:

```text
/EFI/BOOT/BOOTX64.EFI
```

Nie należy więc pisać, że pierwszy instalator instaluje natywną ścieżkę BIOS i UEFI na dysku.

---

# 16. Secure Boot

Aktualny:

```text
BOOTX64.EFI
```

nie posiada łańcucha podpisu potrzebnego do Secure Boot.

Brakuje m.in. wdrożonego modelu:

```text
Microsoft-signed shim
własny zaufany certyfikat firmware
podpisany EFI image
```

Dlatego aktualne wymaganie:

> **Secure Boot musi być wyłączony.**

Podpisy kryptograficzne aktualizacji OTA nie zmieniają tego faktu.

---

# 17. A/B po instalacji

Po późniejszym rozwoju podsystemu aktualizacji dysk jest przygotowany do pracy ze slotami:

```text
A
B
```

Na ESP:

```text
jadro_a.bin
jadro_b.bin
```

W BSP2 aktualizacja systemowa może używać:

```text
/sloty/a/
/sloty/b/
```

dla systemowych drzew wersjonowanych.

Stan slotów jest kontrolowany przez Boot State i `grubenv`.

---

# 18. `grubenv`

Plik:

```text
/boot/grub/grubenv
```

jest stanem używanym przez GRUB przed startem kernela.

Nie jest autorytatywnym źródłem prawdy.

Autorytatywny stan A/B znajduje się w:

```text
LBA 2046
LBA 2047
```

oraz posiada lustrzane kopie w BSP2.

Jeżeli `grubenv` i Boot State się różnią, kernel może odrzucić nieautoryzowany slot i naprawić stan pochodny.

---

# 19. Znacznik instalacji

Instalator posiada własny redundantny znacznik instalacji rozróżniający:

```text
in-progress
complete
```

Znacznik wykorzystuje:

- dwie kopie,
- numer sekwencji,
- CRC.

Instalacja nie jest uznawana za zakończoną tylko dlatego, że część plików pojawiła się na dysku.

---

# 20. Kolejność trwałego zapisu

Krytyczne etapy używają:

```text
write
→ ATA FLUSH
→ read-back
→ verify
```

Dopiero po prawidłowym zakończeniu wcześniejszego etapu można publikować kolejny stan.

Dotyczy to szczególnie:

- GPT,
- filesystemu,
- ESP,
- znacznika instalacji,
- późniejszych metadanych A/B.

---

# 21. ATA FLUSH

Pierwotny stan AHCI nie posiadał wymaganego `FLUSH CACHE`.

Pierwsza implementacja instalatora wprowadziła wymóg:

```text
ATA FLUSH
```

dla krytycznych etapów.

Jest to istotne dla odporności na utratę zasilania.

Samo zakończenie funkcji `write()` nie jest wystarczającym dowodem trwałości danych na urządzeniu.

---

# 22. Ponowna walidacja dysku przed pierwszym zapisem

Między ekranem potwierdzenia a pierwszym zapisem instalator ponownie wykonuje:

```text
IDENTIFY DEVICE
```

i porównuje dane wybranego urządzenia.

Weryfikowane są m.in.:

- port,
- model,
- numer seryjny,
- liczba sektorów,
- rozmiar logicznego sektora,
- fingerprint.

Jeżeli urządzenie zmieni się w międzyczasie:

```text
ABORT
```

przed pierwszym zapisem.

Chroni to przed klasą błędów:

```text
TOCTOU / zmiana targetu
```

---

# 23. Ekran bezpieczeństwa

Przed zatwierdzeniem instalacji interfejs powinien jawnie pokazywać:

```text
UEFI
Secure Boot: wymagane OFF
model dysku
identyfikację kontrolera
SATA AHCI
sektor 512 B / 512e
rozmiar ESP
rozmiar BSP2
informację o usunięciu danych
informację o nieprzydzielonej reszcie dysku
```

Użytkownik powinien widzieć **dokładnie ten dysk**, na którym zostanie wykonana operacja destrukcyjna.

---

# 24. „Własny rozmiar”

Opcja własnego rozmiaru nie powinna być aktywowana dopóki system nie potrafi bezpiecznie:

- analizować istniejących filesystemów,
- wykrywać wolnego miejsca wewnątrz partycji,
- zmniejszać filesystemu,
- przesuwać partycji,
- wykonywać recovery po przerwaniu resize.

Aktualny model:

```text
pełny kontrolowany layout dysku
```

jest prostszy i bezpieczniejszy niż udawanie obsługi instalacji obok dowolnego systemu.

---

# 25. Pozostałe miejsce dysku

Bieżący instalator tworzy:

```text
ESP 512 MiB
+
BSP2 maks. 2 GiB
```

a resztę pozostawia:

```text
nieprzydzieloną
```

Nie jest to ograniczenie GPT.

Jest to ograniczenie obecnego wydania instalatora/BSP2.

---

# 26. Limit BSP2

Nie należy mylić:

```text
maks. 2 GiB partycji w obecnym installerze
```

z:

```text
maksymalną pojemnością wynikającą z GPT
```

GPT używa 64-bitowych LBA.

Dodatkowo historyczny layout pliku BSP2 miał zakres adresowania około:

```text
(490 + 512 + 512²) * 4096
= 1 077 846 016 B
```

a część publicznych pól rozmiaru pliku pozostawała 32-bitowa.

Przed zwiększeniem limitów należy ujednolicić ABI filesystemu.

---

# 27. RAM-only Live BSP2

Normalne Live używa BSP2 w RAM.

Historyczny audyt przed instalatorem wykrył, że stary cache o 256 blokach mógł utracić brudny blok po eviction w RAM-only.

Aktualne materiały potwierdzają bezpieczną politykę:

```text
normal Live
→ brak automatycznego formatowania fizycznego dysku
```

ale wewnętrzną implementację cache RAM-only warto nadal traktować jako osobny temat do ponownego audytu aktualnego `psf.cpp`.

Nie należy mieszać:

```text
brak zapisu na fizyczny dysk
```

z:

```text
gwarancją trwałości danych Live po restarcie
```

---

# 28. Propagacja błędów BSP2

Historyczny kod miał niebezpieczne zachowania:

```text
błąd odczytu
→ zero-filled block
```

oraz niepełną kontrolę części flush.

Pierwsza implementacja instalatora dodała rygorystyczniejsze:

```text
FLUSH
read-back
fail-closed
```

dla ścieżek instalacji.

Nie oznacza to automatycznie, że każda wewnętrzna operacja BSP2 przeszła pełny fault-injection audit.

Przed produkcyjnym wydaniem należy ponownie przetestować:

```text
read failure
write failure
flush failure
partial write
corrupted metadata
```

---

# 29. Instalacja aplikacji i zasobów

System może ponownie wykorzystać mechanizmy tworzące drzewo systemowe i wbudowane aplikacje:

```text
utworz_drabine_katalogow()
instaluj_wbudowane_programy()
```

Instalacja obejmuje m.in.:

- programy `.bur`,
- paczki `.cebula`,
- manifesty,
- fonty,
- zasoby systemowe.

Aktualny ekosystem BursztynUI i aplikacji jest większy niż w pierwszym audycie, więc lista payloadów instalacyjnych powinna być generowana z bieżącego builda, a nie utrzymywana ręcznie w starej dokumentacji.

---

# 30. Tapety i zasoby boot

Historyczne ISO ładowało moduły takie jak:

```text
tapety.pak
test8k.bmp
```

W aktualnej architekturze instalator musi dopilnować, aby payload potrzebny do trwałej instalacji nie został zwolniony przed zakończeniem jego kopiowania.

Legacy odczyt surowego:

```text
LBA 10
```

nie powinien wrócić do aktywnej ścieżki runtime.

---

# 31. Świeża konfiguracja ekranu

Instalator nie powinien bezwarunkowo kopiować konfiguracji ekranu środowiska Live.

Przykład:

```text
QEMU Bochs 2560x1440
```

nie musi odpowiadać fizycznemu monitorowi użytkownika.

Świeża instalacja powinna startować z bezpiecznej konfiguracji, a późniejszy runtime może zapisać:

```text
/ustawienia/ekran.cfg
```

dopiero po potwierdzeniu działającego trybu.

---

# 32. Tożsamość instalatora i PZB

Instalator jest aplikacją Ring 3, ale operacje destrukcyjne wymagają zaufanej tożsamości nadanej przez kernel.

Manifest nie może sam sobie przyznać praw do formatowania dysku.

Krytyczne operacje wymagają m.in.:

```text
PRAWO_SYSTEM_CONFIG
```

oraz polityki PZB.

Nie należy wiązać dokumentacji instalatora ze starym założeniem:

```text
BWS 0..83 = cały aktualny ABI
```

ponieważ późniejsze rozszerzenia dodały co najmniej BWS OTA 84–85 oraz dalsze numery związane z motywami.

Kanonicznym źródłem numeracji BWS powinien być zawsze:

```text
bws_api.h
```

---

# 33. IPC instalatora

Historyczny audyt wspominał BWS 69–71 jako istniejące IPC.

Aktualne nowoczesne ABI obejmuje:

```text
68 — IPC create
69 — IPC send
70 — IPC receive
71 — IPC close
```

Instalator może korzystać z usług Ring 3, ale sam fakt istnienia IPC nie daje mu prawa do operacji administracyjnych.

Autoryzacja pozostaje po stronie kernela.

---

# 34. Integracja z aktualizacjami A/B

Po wdrożeniu mechanizmu aktualizacji instalator musi tworzyć taki layout, aby późniejsze OTA mogło bezpiecznie działać.

Wymagane elementy:

```text
ESP z jadro_a.bin i jadro_b.bin
grub.cfg / grubenv
zarezerwowane LBA 2046/2047
BSP2 z wymaganym namespace A/B
zgodny GUID BSP2
poprawny Boot State
```

Instalator jest więc obecnie również **inicjalizatorem przyszłego środowiska aktualizacji**, nie tylko kopierem plików.

---

# 35. Pierwszy stan A/B po instalacji

Świeża instalacja powinna posiadać jednoznaczny stan początkowy.

Koncepcyjnie:

```text
slot potwierdzony = A
slot oczekujący   = brak
licznik prób      = 0
```

Drugi slot nie może zostać przypadkowo uznany za zatwierdzony tylko dlatego, że istnieje plik:

```text
jadro_b.bin
```

Autoryzację określa Boot State.

---

# 36. Znaczniki instalacji a Boot State

Są to dwa różne mechanizmy.

### Installation marker

Odpowiada na pytanie:

```text
Czy instalacja dysku zakończyła się poprawnie?
```

### A/B Boot State

Odpowiada na pytanie:

```text
Który slot wolno uruchomić i który został potwierdzony?
```

Nie należy scalać tych struktur ani ich semantyki tylko dlatego, że oba mechanizmy używają redundancji i CRC.

---

# 37. Odporność na awarię zasilania

Minimalne oczekiwania:

### Awaria podczas GPT

```text
instalacja nie może zostać uznana za complete
```

### Awaria podczas ESP

```text
complete nie może zostać ustawione
```

### Awaria podczas BSP2

```text
complete nie może zostać ustawione
```

### Awaria między kopiami installation marker

```text
wybierana jest poprawna/nowsza kopia
```

### Awaria podczas inicjalizacji Boot State

```text
co najmniej jedna poprawna kopia musi zostać zachowana
```

Normalny boot nie może automatycznie sformatować systemu „naprawiając” niepełną instalację.

---

# 38. Reinstalacja

Dedykowane testy z 2026-09-09 obejmowały:

```text
fresh install
UEFI boot bez ISO
reinstall
kolejny boot bez ISO
```

Reinstalacja jest operacją destrukcyjną i powinna ponownie wymagać:

- wyboru dysku,
- potwierdzenia,
- ponownej walidacji tożsamości,
- zgodności sektora,
- zgodności kontrolera.

Nie należy traktować wykrycia starego Bursztyn OS jako zgody na automatyczne nadpisanie.

---

# 39. Ochrona drugiego dysku

Historyczny `installer-test` weryfikował SHA-256 drugiego dysku przed i po instalacji.

Invariant:

```text
wybrany target
→ może zostać zmodyfikowany

każdy inny dysk
→ musi pozostać identyczny
```

To powinno pozostać obowiązkowym release gate.

Test powinien obejmować również sytuacje:

```text
dwa podobne modele
zmiana kolejności PCI/portów
re-enumeracja
```

aby potwierdzić znaczenie fingerprintu, a nie tylko indeksu dysku.

---

# 40. Test hybrydowego ISO

Historyczny audyt potwierdził test uruchomienia hybrydowego ISO jako USB w środowisku QEMU.

Należy rozróżnić:

```text
ISO hybrid boot test
```

od:

```text
USB Mass Storage jako target instalacji
```

Pierwsze może działać dzięki firmware/bootloaderowi.

Drugie nadal wymaga natywnego stosu USB block storage, którego Bursztyn OS obecnie nie ma.

---

# 41. Wynik `4491/4491`

Audyt z 2026-09-09 raportował:

```text
make installer-test
4491 / 4491 PASS
```

Był to wynik bardzo wartościowy, ale dotyczy konkretnego stanu kodu.

Od tego czasu pojawiły się późniejsze elementy architektury:

- A/B,
- LBA 2046/2047,
- `grubenv`,
- OTA,
- kolejne rozszerzenia BWS.

Dlatego przed aktualnym wydaniem należy ponownie uruchomić:

```text
make installer-test
```

razem z testami rozruchu A/B.

Nie należy przepisywać historycznego `4491/4491` jako świeżego wyniku bieżącego `HEAD`, jeśli test nie został ponowiony.

---

# 42. Testy A/B powiązane z instalatorem

Po utworzeniu dysku przez installer należy dodatkowo testować:

```text
boot slot A
install/update B
trial boot B
confirm B
rollback B→A
stale grubenv
one bad Boot State copy
split write między 2046 i 2047
unauthorized slot boot
```

Instalator i updater dzielą część krytycznego layoutu dyskowego, dlatego testy obu podsystemów nie powinny być całkowicie odseparowane.

---

# 43. Status Secure Boot w testach

Test UEFI bez Secure Boot nie jest testem Secure Boot.

Release documentation musi jawnie mówić:

```text
Secure Boot OFF
```

dopóki projekt nie posiada:

- podpisanego obrazu EFI,
- kontrolowanego łańcucha zaufania,
- procedury zarządzania kluczami.

Podpis OTA RSA-2048 nie zastępuje podpisu bootloadera EFI.

---

# 44. Manualna weryfikacja sprzętu

Aktualny status pozostaje:

```text
NOT MANUALLY VERIFIED ON REAL HARDWARE
```

Przed zmianą tego statusu należy przetestować co najmniej:

```text
UEFI machine
Secure Boot disabled
native SATA AHCI
512n lub 512e SSD/HDD
boot z nośnika instalacyjnego
instalację
boot bez ISO
reboot
reinstall
A/B update
rollback
power-cycle
```

Warto również przetestować maszynę, na której firmware wystawia:

```text
Intel RST/VMD
```

i potwierdzić bezpieczną odmowę instalacji bez zapisu.

---

# 45. Brak obsługi dual boot

Aktualny installer nie powinien być dokumentowany jako pełny instalator:

```text
obok Windows
obok Linux
obok istniejącego GPT
```

jeżeli nie posiada bezpiecznego:

- shrink,
- move,
- filesystem-aware resize,
- boot manager integration.

Obecny tryb pełnego kontrolowanego layoutu jest właściwszy dla pierwszego wydania.

---

# 46. Bezpieczna arytmetyka LBA

Każda operacja instalatora powinna sprawdzać overflow dla:

```text
start_lba + sector_count
partition.start_lba + relative_lba
size_bytes + sector_size - 1
entry_count * entry_size
```

Nie wolno wykonywać obliczenia i dopiero później sprawdzać, czy wynik „wygląda poprawnie”.

Preferowane są helpery:

```cpp
checked_add(...)
checked_mul(...)
```

---

# 47. Walidacja GPT

Instalator powinien przy read-back sprawdzać:

- signature,
- revision,
- header size,
- header CRC,
- partition-array CRC,
- `current_lba`,
- `backup_lba`,
- usable range,
- entry size,
- liczbę wpisów,
- położenie backup array/header.

Dodatkowo layout Bursztyn OS musi sprawdzać rezerwację:

```text
LBA 2046..2047
```

niezależnie od ogólnej poprawności GPT.

---

# 48. Ochrona przed pomyłką urządzenia

Do operacji destrukcyjnej nie wystarcza:

```text
„pierwszy dysk”
```

ani:

```text
„port 0”
```

Wymagane jest powiązanie zatwierdzonego wyboru z fingerprintem urządzenia.

Jeżeli serial jest pusty lub nieużyteczny, fingerprint powinien korzystać z dostępnego zestawu stabilnych właściwości i działać fail-closed przy niejednoznaczności.

---

# 49. Logowanie instalacji

Log powinien zawierać wystarczające informacje diagnostyczne, ale nie powinien eksponować niepotrzebnie danych użytkownika.

Warto logować:

```text
etap instalacji
target port
model
logical sector size
capacity
GPT write/read-back status
ESP status
BSP2 status
flush status
installation marker generation
Boot State init status
```

Dla każdego błędu:

```text
pierwszy nieudany etap
```

powinien być jednoznaczny.

---

# 50. Release gate instalatora

Przed wydaniem należy wymagać:

```text
[ ] normal Live = 0 destructive writes
[ ] brak auto-format
[ ] brak targetu AHCI = 0 writes
[ ] 4Kn reject before write
[ ] NVMe reject before write
[ ] RST/VMD reject before write
[ ] wrong disk fingerprint abort before write
[ ] GPT primary PASS
[ ] GPT backup PASS
[ ] GPT CRC PASS
[ ] ESP FAT32 PASS
[ ] BOOTX64.EFI present
[ ] jadro_a.bin present
[ ] jadro_b.bin present / poprawnie zainicjalizowane wg bieżącego layoutu A/B
[ ] grubenv valid
[ ] Boot State LBA2046 valid
[ ] Boot State LBA2047 valid
[ ] BSP2 mount PASS
[ ] installation marker complete
[ ] second disk SHA unchanged
[ ] boot without ISO PASS
[ ] reinstall PASS
[ ] A/B trial boot PASS
[ ] rollback PASS
[ ] power-loss tests PASS
[ ] Secure Boot requirement visible
[ ] manual QEMU visual pass
```

Przed deklaracją wsparcia fizycznego sprzętu:

```text
[ ] real hardware pass
```

---

# 51. Aktualny status funkcji

| Element | Status |
|---|---|
| Instalator Ring 3 | **zaimplementowany** |
| `bursztyn.install=1` | **zaimplementowany** |
| Normal Live bez auto-format | **zaimplementowany** |
| GPT primary/backup | **zaimplementowany** |
| CRC GPT | **zaimplementowany** |
| ESP FAT32 512 MiB | **zaimplementowany** |
| Standardowy `BOOTX64.EFI` | **zaimplementowany** |
| NVRAM boot variable | **niewymagana** |
| BSP2 względem partycji | **zaimplementowany** |
| AHCI/LBA48 | **zaimplementowane** |
| ATA FLUSH | **zaimplementowany dla krytycznych ścieżek instalacji** |
| 512 B / 512e | **obsługiwane** |
| 4Kn | **nieobsługiwane** |
| NVMe target | **nieobsługiwany** |
| USB Mass Storage target | **nieobsługiwany** |
| RST/VMD/RAID | **nieobsługiwane** |
| Full-disk controlled layout | **zaimplementowany** |
| Własny resize / dual boot | **niezaimplementowany** |
| Installation marker in-progress/complete | **zaimplementowany** |
| Ponowna walidacja targetu | **zaimplementowana** |
| A/B boot-state LBA2046/2047 | **zaimplementowany w obecnym systemie** |
| Secure Boot | **nieobsługiwany** |
| Real hardware | **NOT MANUALLY VERIFIED** |

---

# 52. Co jest już nieaktualne z sekcji „przed implementacją”

Poniższych stwierdzeń nie należy już przedstawiać jako obecnego stanu:

```text
BSP2 ma stałą bazę LBA 40000
brak GPT
brak abstrakcji partycji
brak FLUSH CACHE
normalny boot może automatycznie formatować fizyczny dysk
legacy LBA 10 jest aktywną ścieżką
instalator nie ma gotowego ESP
system ma tylko jeden obraz jądra
BWS kończy się na 83
```

Są to dane historyczne opisujące stan sprzed pierwszej implementacji.

---

# 53. Co nadal pozostaje ograniczeniem

Aktualne ograniczenia:

```text
tylko UEFI dla zainstalowanego systemu
Secure Boot OFF
SATA AHCI
512 B / 512e
brak NVMe
brak USB storage
brak RST/VMD/RAID
brak 4Kn
BSP2 limit 2 GiB w bieżącym installerze
brak bezpiecznego shrink/resize innych filesystemów
brak pełnego dual boot installera
real hardware niezweryfikowany manualnie
```

---

# 54. Podsumowanie

Aktualny instalator Bursztyn OS nie jest już eksperymentem opartym na stałym LBA.

Obecna architektura:

```text
Live RAM-only
      ↓
bursztyn.install=1
      ↓
Instalator Ring 3
      ↓
AHCI IDENTIFY + wybór targetu
      ↓
potwierdzenie
      ↓
ponowna walidacja fingerprintu
      ↓
GPT
      ↓
ESP FAT32
      ↓
BSP2 / PSF
      ↓
installation marker
      ↓
Boot State A/B
      ↓
ATA FLUSH + read-back
      ↓
boot UEFI z dysku bez ISO
```

Najważniejsze mocne strony obecnego modelu:

- brak automatycznego formatowania w normalnym Live,
- jawna ścieżka instalacyjna,
- GPT z kopią zapasową,
- read-back,
- ATA FLUSH,
- walidacja targetu przed pierwszym zapisem,
- ochrona drugiego dysku,
- standardowa ścieżka `BOOTX64.EFI`,
- przygotowanie pod aktualizacje A/B.

Najważniejsze rzeczy wymagające dalszego rozwoju:

- real-hardware verification,
- NVMe,
- USB Mass Storage,
- 4Kn,
- Secure Boot,
- resize/dual boot,
- ponowne wykonanie pełnych testów installera razem z aktualnym layoutem A/B.

---

## 55. Dokumenty powiązane

Aktualny stan instalatora należy interpretować razem z:

- `03_proces_rozruchu.md` — UEFI, GRUB i Multiboot2,
- `04_zarzadzanie_sprzetem.md` — AHCI i storage,
- `05_bursztynowy_system_plikow.md` — BSP2 / PSF,
- `06_wywolania_systemowe.md` — BWS i polityka Ring 3,
- `07_ekosystem_i_formaty.md` — `.bur` / `.cebula`,
- `aktualizacja-systemu.md` — A/B, LBA 2046/2047 i `grubenv`.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod instalatora,
2. aktualny storage/AHCI,
3. aktualny GPT/partition layer,
4. aktualny BSP2,
5. aktualny kod A/B Boot State,
6. aktualne testy,
7. starszy audyt z 2026-09-09.

---

## 56. Zalecana następna weryfikacja

Po obecnych zmianach projektu należy wykonać jeden zbiorczy test:

```text
fresh disk
→ Live boot
→ verify zero disk writes
→ install
→ boot A without ISO
→ verify filesystem
→ verify Boot State
→ install/update B
→ trial boot B
→ confirm
→ reboot
→ force failed trial next update
→ verify rollback
→ reinstall
→ verify second disk unchanged
```

oraz zebrać końcowe liczniki:

```text
installer assertions
boot-state assertions
OTA assertions
QEMU UEFI result
second disk SHA-256
real hardware status
```

Dopiero taki przebieg będzie aktualnym, spójnym wynikiem dla obecnego layoutu instalacji + A/B.
