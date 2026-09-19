# 03. Proces Rozruchu (Boot Process) i Sekwencja Long Mode

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** GRUB / Multiboot2, start BIOS/UEFI, wybór slotu A/B, wejście w Long Mode, bootstrap paging, przekazanie danych do `kernel_main` i inicjalizacja wczesnego środowiska jądra

---

## 3.1 Zakres procesu rozruchowego

Proces rozruchu Bursztyn OS obejmuje kilka kolejnych warstw:

```text
Firmware
   ↓
BIOS lub UEFI
   ↓
GRUB
   ↓
wybór obrazu jądra / slotu A-B
   ↓
Multiboot2
   ↓
32-bitowy bootstrap jądra
   ↓
wczesne stronicowanie
   ↓
Long Mode x86-64
   ↓
własny stos bootstrap
   ↓
kernel_main(...)
   ↓
PMM / VMM / GDT / TSS / IDT / APIC / HPET
   ↓
sterowniki i podsystemy systemowe
   ↓
Ring 3 i aplikacje
```

Bursztyn OS używa jednego własnego obrazu jądra zgodnego z Multiboot2. Bootloader odpowiada za załadowanie obrazu jądra i przekazanie struktury informacji Multiboot2, natomiast właściwa inicjalizacja x86-64 jest wykonywana przez kod bootstrapowy Bursztyn OS.

---

## 3.2 Dwie główne ścieżki startu

Aktualny projekt ma dwie praktycznie istotne ścieżki rozruchu.

### 3.2.1 Hybrydowy obraz ISO

Obraz generowany przez `grub-mkrescue` zawiera ścieżki umożliwiające rozruch w środowisku BIOS i UEFI.

Audyt obrazu instalacyjnego wykazał strukturę hybrydową obejmującą:

- BIOS El Torito,
- UEFI El Torito,
- ochronny MBR,
- GPT/APM obrazu ISO.

Ta ścieżka jest używana przede wszystkim podczas:

- uruchamiania Live,
- testów QEMU,
- uruchamiania instalatora.

### 3.2.2 System zainstalowany na dysku

Aktualna ścieżka instalacyjna jest przeznaczona dla:

```text
UEFI x86-64
+
GPT
+
ESP FAT32
+
SATA AHCI
```

Partycja ESP zawiera m.in.:

```text
/EFI/BOOT/BOOTX64.EFI
/boot/jadro_a.bin
/boot/jadro_b.bin
/boot/grub/grub.cfg
/boot/grub/grubenv
```

`BOOTX64.EFI` jest obrazem GRUB x86_64-efi przygotowanym dla Bursztyn OS.

W aktualnym wydaniu Secure Boot musi być wyłączony, ponieważ `BOOTX64.EFI` nie posiada pełnego łańcucha podpisu wymaganego przez Secure Boot.

---

## 3.3 Rozruch A/B przed wejściem do jądra

Zainstalowany system wykorzystuje dwa sloty jądra:

```text
Slot A
Slot B
```

GRUB odczytuje stan z:

```text
/boot/grub/grubenv
```

i na jego podstawie wybiera:

```text
/boot/jadro_a.bin
```

albo:

```text
/boot/jadro_b.bin
```

`grubenv` jest buforem stanu używanym w fazie przedjądrowej. Nie jest jednak głównym źródłem prawdy systemu aktualizacji.

Autorytatywny stan A/B jest utrzymywany przez Bursztyn OS w redundantnych rekordach boot state oraz ich kopiach systemowych.

W uproszczeniu:

```text
GRUB
  ↓
odczytaj grubenv
  ↓
czy istnieje slot oczekujący?
  ├─ NIE → uruchom slot potwierdzony
  └─ TAK
       ↓
    pierwsza próba?
       ├─ TAK → uruchom slot oczekujący
       └─ NIE → fallback do slotu potwierdzonego
```

Nowy slot jest uznawany za stabilny dopiero po osiągnięciu przez jądro punktu kontrolnego gotowości.

Jeżeli system zrestartuje się lub ulegnie awarii przed potwierdzeniem, GRUB może wrócić do poprzedniego slotu.

---

## 3.4 Multiboot2

Bursztyn OS używa formatu Multiboot2.

Nagłówek Multiboot2 musi znajdować się wystarczająco wcześnie w obrazie jądra i być wyrównany zgodnie ze specyfikacją.

Podstawowe pola nagłówka:

| Pole | Wartość / znaczenie |
|---|---|
| `magic` | `0xE85250D6` |
| `architecture` | `0` — i386 protected mode |
| `header_length` | długość nagłówka |
| `checksum` | suma 32-bitowa pól podstawowych musi dać 0 |

Nagłówek jest zwykle umieszczony w dedykowanej sekcji, np.:

```text
.multiboot
```

wyrównanej do 8 bajtów.

### Ważne: istnieją dwa różne „magic numbers”

Starsza dokumentacja mogła prowadzić do pomylenia dwóch różnych wartości.

#### Magic nagłówka obrazu

```text
0xE85250D6
```

identyfikuje **nagłówek Multiboot2 znajdujący się w pliku jądra**.

#### Magic przekazywany przez bootloader w EAX

Przy wejściu do 32-bitowego punktu startowego zgodnego z Multiboot2:

```text
EAX = 0x36D76289
```

Ta wartość potwierdza, że kontrolę przekazał bootloader zgodny z Multiboot2.

Nie należy używać `0xE85250D6` jako oczekiwanej wartości wejściowej `EAX`.

---

## 3.5 Rejestry wejściowe Multiboot2

W typowej ścieżce i386 Multiboot2 GRUB przekazuje:

```text
EAX = 0x36D76289
EBX = fizyczny adres struktury Multiboot2 Information
```

Pozostałe rejestry nie powinny być traktowane jako zachowujące użyteczne dane, jeśli nie wymaga tego konkretny kontrakt.

Kod bootstrapowy musi zachować:

- magic z `EAX`,
- pointer MBI z `EBX`,

zanim zacznie używać rejestrów do konfiguracji CRx, MSR i tablic stron.

---

## 3.6 Tagi Multiboot2 używane przez Bursztyn OS

Bursztyn OS korzysta z informacji dostarczanych przez Multiboot2, w szczególności z danych potrzebnych do:

- rozpoznania pamięci RAM,
- rezerwacji obszarów używanych przez bootloader i moduły,
- inicjalizacji grafiki,
- wykrycia modułów GRUB,
- dalszego uruchomienia PMM i VMM.

### Mapa pamięci

Mapa pamięci przekazana przez bootloader jest jednym z kluczowych źródeł dla inicjalizacji PMM.

Kernel nie powinien uznawać całego fizycznego RAM za wolny. Musi zarezerwować co najmniej obszary zajęte przez:

- obraz jądra,
- bootstrapowe tablice stron,
- strukturę Multiboot2,
- moduły Multiboot2,
- framebuffer i inne zakresy sprzętowe, jeśli wymagają rezerwacji,
- późniejsze struktury jądra.

### Moduły

GRUB może załadować moduły potrzebne do startu systemu, np. pakiet tapet lub zasoby testowe.

PMM musi wiedzieć, że fizyczne ramki zajmowane przez moduł nie są wolne.

Moduł może zostać zwolniony dopiero po zakończeniu jego importu lub skopiowaniu danych do docelowego storage.

---

## 3.7 Framebuffer Multiboot2

Nagłówek Multiboot2 może zawierać framebuffer header tag typu 5.

W aktualnym projekcie preferowany tryb startowy to:

```text
1280 × 720 × 32 bpp
```

Należy jednak rozumieć tę wartość jako **preferencję**, a nie gwarancję sprzętową.

Bootloader może przekazać inny dostępny tryb.

Dlatego jądro musi zweryfikować rzeczywiście otrzymany framebuffer info:

```text
adres framebufferu
pitch
szerokość
wysokość
bpp
typ framebufferu
```

i dopiero na tej podstawie uruchamiać właściwy backend HAL.

Nie należy zakładać:

```text
pitch == width * 4
```

bez sprawdzenia wartości przekazanej przez bootloader.

---

## 3.8 GOP, VESA i Bochs VBE w kontekście bootu

Aktualna warstwa grafiki posiada trzy backendy:

- UEFI GOP,
- VESA VBE,
- Bochs VBE.

### UEFI GOP

W ścieżce UEFI GRUB otrzymuje framebuffer z firmware i przekazuje jego opis do jądra przez Multiboot2.

Po wejściu do Bursztyn OS jądro nie wymaga runtime EFI ani NVRAM do zwykłego renderowania obrazu.

Aktualny backend GOP należy traktować jako **statyczny tryb startowy** — zmiana rozdzielczości w runtime nie jest wykonywana tą samą ścieżką co Bochs VBE.

### VESA VBE

VESA jest ścieżką dla środowisk legacy/BIOS, w których bootloader dostarcza liniowy framebuffer.

### Bochs VBE

Bochs VBE jest używany głównie w QEMU i emulatorach.

W przeciwieństwie do statycznego GOP, backend Bochs umożliwia aktualnemu podsystemowi ekranu przełączanie trybów podczas działania systemu.

Boot mode dla aktualnych testów Bochs to 1280×720.

---

## 3.9 Wczesne stronicowanie jako trampolina

Wejście w 64-bitowy Long Mode wymaga przygotowania odpowiednich tablic stron.

Kod bootstrapowy tworzy minimalny zestaw struktur umożliwiający bezpieczne wykonanie przejścia:

```text
PML4
 ↓
PDPT
 ↓
PD
```

Bootstrap używa identity mapping:

```text
virtual address == physical address
```

dla zakresu potrzebnego do kontynuowania rozruchu.

W tej fazie mogą być używane 2 MiB huge pages przez ustawienie bitu `PS` w PDE.

### Korekta względem starego dokumentu

Starszy dokument opisywał dokładnie **jedną mapowaną stronę 2 MiB**.

Nie należy traktować tej liczby jako trwałego kontraktu aktualnej architektury.

Współczesny obraz jądra i dane rozruchowe mogą zajmować więcej niż 2 MiB, dlatego rzeczywisty bootstrap musi mapować **cały zakres potrzebny do wykonania kodu, stosu i struktur używanych przed przejęciem pamięci przez docelowy VMM**.

Dokumentacja nie powinna na sztywno deklarować liczby bootstrapowych PDE bez ponownego audytu aktualnego `boot.S`.

Poprawna zasada architektoniczna brzmi:

> wczesne identity mapping ma być możliwie małe, ale wystarczające dla całego aktywnego zakresu bootstrapu.

---

## 3.10 Bootstrapowe tablice stron

Tablice rozruchowe muszą:

- być wyrównane do 4 KiB,
- zostać wyzerowane przed zapisaniem wpisów,
- zawierać poprawne fizyczne adresy kolejnych poziomów,
- ustawiać tylko wymagane flagi,
- istnieć w pamięci przez cały okres ich używania w `CR3`.

Typowa relacja:

```text
PML4[0]
  ↓
PDPT

PDPT[0]
  ↓
PD

PD[n]
  ↓
2 MiB identity-mapped region
```

Dla 2 MiB PDE istotne są między innymi:

```text
Present
Writable
Page Size
```

Bootstrapowe mapowanie nie jest końcowym układem pamięci Bursztyn OS.

---

## 3.11 Wejście w Long Mode

Sekwencja przejścia procesora obejmuje następujące etapy:

### 1. Przygotowanie tablic stron

Bootstrap tworzy struktury PML4/PDPT/PD i odpowiednie identity mapping.

### 2. Załadowanie `CR3`

Do `CR3` trafia fizyczny adres PML4.

### 3. Włączenie PAE

Ustawiany jest:

```text
CR4.PAE = 1
```

czyli bit 5 rejestru `CR4`.

### 4. Włączenie Long Mode Enable

Rejestr:

```text
IA32_EFER = 0xC0000080
```

jest odczytywany przez `RDMSR`.

Następnie ustawiany jest:

```text
EFER.LME = 1
```

i wartość zapisywana przez `WRMSR`.

### 5. Włączenie paging

Ustawiany jest:

```text
CR0.PG = 1
```

Po spełnieniu warunków procesor znajduje się w stanie umożliwiającym aktywację Long Mode po załadowaniu odpowiedniego segmentu kodu.

### 6. Załadowanie GDT 64-bit

Bootstrap wykonuje:

```asm
lgdt gdt64_pointer
```

### 7. Daleki skok

Następuje far jump do selektora 64-bitowego kodu jądra, np.:

```asm
ljmp $0x08, $long_mode_start
```

To ładuje `CS` selektorem kodu Long Mode i przełącza wykonywanie na 64-bitowy kod.

---

## 3.12 Wczesna GDT a docelowa GDT

GDT używana przy wejściu do Long Mode jest strukturą bootstrapową.

Jej zadaniem jest umożliwienie poprawnego przejścia do 64-bitowego kodu.

Później jądro inicjalizuje pełną architekturę GDT/TSS potrzebną do:

- Ring 0,
- Ring 3,
- obsługi wyjątków,
- stosów Ring 0,
- TSS,
- IST,
- BWS i przejść poziomów uprzywilejowania.

Nie należy utożsamiać minimalnej `gdt64` z pełną strukturą GDT używaną po inicjalizacji systemu.

---

## 3.13 Wejście do `.code64`

Po far jump kod wykonuje się w 64-bitowym trybie.

Bootstrap przygotowuje minimalne środowisko wymagane przed wywołaniem C++:

- poprawny stos,
- wymagane rejestry segmentowe,
- zachowane dane Multiboot2,
- odpowiednie wyrównanie stosu dla ABI.

### Stos bootstrapowy

Starszy dokument podawał na sztywno:

```text
16 KiB
```

jako rozmiar początkowego stosu.

Tego stosu nie należy mylić z późniejszymi stosami Ring 0 procesów, które mają własny model ochrony i większe wydzielone przestrzenie.

Dokumentacja architektoniczna powinna traktować rozmiar stosu bootstrap jako detal implementacji definiowany w aktualnym `boot.S`, chyba że jest on utrzymywany jako publiczny kontrakt.

Najważniejsze wymagania:

- stos musi być dostępny w bootstrapowym mapowaniu,
- `RSP` musi wskazywać prawidłowy adres,
- stos musi być odpowiednio wyrównany przed `call kernel_main`.

---

## 3.14 System V AMD64 ABI

Kod C/C++ jądra jest wywoływany zgodnie z używanym przez toolchain ABI x86-64.

Dla typowego System V AMD64 ABI pierwsze argumenty całkowitoliczbowe przekazuje się przez:

```text
RDI
RSI
RDX
RCX
R8
R9
```

Dlatego wartości Multiboot2 zachowane w fazie 32-bitowej mogą zostać przygotowane jako pierwsze argumenty:

```text
RDI = multiboot magic
RSI = multiboot info pointer
```

Następnie bootstrap wykonuje:

```asm
call kernel_main
```

Koncepcyjna deklaracja:

```cpp
extern "C" void kernel_main(
    uint64_t multiboot_magic,
    uint64_t multiboot_info_ptr);
```

`extern "C"` zapobiega C++ name mangling punktu wejścia wywoływanego z assemblera.

---

## 3.15 Zachowanie EAX/EBX

Jeżeli bieżący `boot.S` kopiuje dane w stylu:

```asm
movl %eax, %edi
movl %ebx, %esi
```

należy pamiętać, że zapis do 32-bitowego rejestru w trybie x86-64 zeruje górną połowę odpowiadającego rejestru 64-bitowego.

Jest to właściwe dla danych Multiboot2, których klasyczna ścieżka i386 używa 32-bitowych wartości wejściowych.

Kod nie może jednak polegać na przypadkowym przetrwaniu tych danych w rejestrach używanych później przez `RDMSR`, `WRMSR`, `CR3` i pozostałą konfigurację bootstrapu.

---

## 3.16 `kernel_main` — przejęcie sterowania

Po wejściu do `kernel_main` kończy się najniższa asemblerowa faza rozruchu.

Od tego momentu kod C/C++ może rozpocząć inicjalizację właściwego środowiska systemowego.

W uproszczeniu:

```text
kernel_main
   ↓
walidacja Multiboot2
   ↓
analiza memory map / framebuffer / modules
   ↓
PMM
   ↓
VMM
   ↓
pełna GDT / TSS
   ↓
IDT i wyjątki
   ↓
APIC / HPET / scheduler
   ↓
storage / BSP2
   ↓
USB / sieć / audio
   ↓
grafika / kompozytor
   ↓
BWS
   ↓
Ring 3
```

Rzeczywista kolejność części podsystemów może ewoluować i powinna być zgodna z zależnościami w aktualnym `kernel.cpp`.

---

## 3.17 Inicjalizacja PMM

PMM nie może rozpocząć od założenia, że wszystkie ramki raportowane jako RAM są wolne.

Podczas budowania bitmapy należy uwzględnić co najmniej:

```text
kernel image
bootstrap paging
Multiboot2 Information
Multiboot2 modules
obszary firmware / reserved
struktury jądra
MMIO i inne rezerwacje wymagane przez platformę
```

Kolejność ma znaczenie:

```text
odczytaj mapę pamięci
      ↓
zbuduj stan ramek
      ↓
zarezerwuj zajęte obszary
      ↓
dopiero wtedy zezwól na zwykłe alokacje
```

---

## 3.18 Przejście do docelowego VMM

Bootstrapowe tablice stron są tylko trampoliną.

Docelowy VMM tworzy pełny układ przestrzeni adresowej używany przez Bursztyn OS.

Aktualna architektura obejmuje między innymi:

- 4-poziomowe stronicowanie x86-64,
- mapowania jądra,
- HHDM,
- prywatne przestrzenie Ring 3,
- SHM,
- wydzielone obszary stosów Ring 0,
- osobne obszary dla dużych buforów grafiki PMM/VMM.

W docelowym VMM mogą być używane 2 MiB huge pages tam, gdzie są uzasadnione, natomiast nie wszystkie typy mapowań muszą korzystać z huge pages.

---

## 3.19 Framebuffer podczas przejścia do VMM

Framebuffer może znajdować się pod fizycznym adresem niezwiązanym z normalnym RAM.

Dlatego przejście z bootstrap paging do docelowego VMM nie może zgubić mapowania potrzebnego do dalszego renderowania.

HAL powinien pracować na danych zweryfikowanych z framebuffer tagu, m.in.:

```text
physical address
pitch
width
height
bpp
pixel layout
```

Po utworzeniu finalnej przestrzeni jądra framebuffer musi posiadać poprawne mapowanie lub mechanizm dostępu zgodny z docelową architekturą VMM.

---

## 3.20 Rozruch trybu Live

Normalny rozruch Live nie powinien automatycznie niszczyć zawartości fizycznych dysków.

Po wdrożeniu instalatora ścieżka Live została rozdzielona od instalacji.

Tryb instalacyjny jest uruchamiany jawnie, np. przez parametr:

```text
bursztyn.install=1
```

W tym trybie uruchamiana jest aplikacja instalatora Ring 3.

Brak parametru instalacyjnego oznacza normalny start środowiska Live.

---

## 3.21 Rozruch po instalacji

Po instalacji boot chain wygląda koncepcyjnie tak:

```text
UEFI firmware
   ↓
ESP FAT32
   ↓
/EFI/BOOT/BOOTX64.EFI
   ↓
GRUB
   ↓
grubenv
   ↓
wybór Slot A / Slot B
   ↓
jądro Multiboot2
   ↓
bootstrap 32-bit
   ↓
Long Mode
   ↓
kernel_main
   ↓
BSP2 + usługi systemowe
```

Ta ścieżka nie wymaga utworzenia własnej zmiennej NVRAM, ponieważ używa standardowej ścieżki removable-media:

```text
/EFI/BOOT/BOOTX64.EFI
```

---

## 3.22 Rozruch A/B i rollback

Dla aktualizacji systemu proces rozruchu posiada dodatkową logikę:

```text
slot A potwierdzony
      ↓
aktualizacja instalowana do B
      ↓
B oznaczony jako oczekujący
      ↓
restart
      ↓
GRUB uruchamia B jako próbę
      ↓
czy kernel osiągnął health checkpoint?
      ├─ TAK → B zostaje potwierdzony
      └─ NIE → następny start wraca do A
```

Analogicznie następna aktualizacja może zostać zapisana ponownie do slotu A.

Mechanizm ten ma zapobiegać trwałemu unieruchomieniu systemu przez nieudany update.

---

## 3.23 Autorytatywny stan rozruchu

Bursztyn OS przechowuje dwa redundantne rekordy boot state:

```text
LBA 2046
LBA 2047
```

Znajdują się one przed ESP rozpoczynającą się od LBA 2048.

Rekordy wykorzystują:

- CRC32,
- licznik generacji,
- walidację semantyczną,
- stan slotów A/B.

`grubenv` jest synchronizowany z tym stanem, ale nie zastępuje autorytatywnej informacji zarządzanej przez jądro.

---

## 3.24 Bezpieczeństwo procesu boot

Najważniejsze zasady:

1. Nie ufać bezwarunkowo danym Multiboot2.
2. Walidować długości i granice tagów.
3. Zachowywać wymagane moduły, dopóki nie zostaną przetworzone.
4. Nie zwalniać ramek jądra ani boot info do PMM.
5. Nie wykonywać niekontrolowanego zapisu na dysk podczas zwykłego Live boot.
6. Nie aktywować niepotwierdzonego slotu jako trwałego bez health checkpoint.
7. Przy uszkodzonym stanie aktualizacji preferować fail-closed / znany dobry slot.
8. Nie zakładać, że żądany framebuffer mode został rzeczywiście ustawiony.
9. Nie zakładać konkretnego pitch bez odczytu informacji framebufferu.
10. Nie włączać przerwań przed przygotowaniem własnego IDT i wymaganych struktur.

---

## 3.25 Czego nie należy już pisać w dokumentacji

### Niepoprawne:

```text
Multiboot2 magic w EAX = 0xE85250D6
```

### Poprawne:

```text
header magic          = 0xE85250D6
bootloader magic EAX  = 0x36D76289
```

---

### Niepoprawne:

```text
Framebuffer tag gwarantuje dokładnie 1280x720.
```

### Poprawne:

```text
1280x720x32 jest preferowanym trybem startowym;
kernel musi zaakceptować i zweryfikować tryb faktycznie przekazany przez GRUB.
```

---

### Niepoprawne jako trwałe założenie:

```text
Bootstrap mapuje zawsze tylko pierwsze 2 MiB.
```

### Poprawne:

```text
Bootstrap identity-mapuje cały zakres wymagany do bezpiecznego wejścia
do Long Mode i przekazania kontroli docelowemu VMM.
```

---

### Niepoprawne:

```text
UEFI bezpośrednio wywołuje kernel_main.
```

### Poprawne dla aktualnej architektury:

```text
UEFI → BOOTX64.EFI/GRUB → Multiboot2 → bootstrap jądra → kernel_main
```

---

## 3.26 Diagnostyka rozruchu

Wczesna diagnostyka powinna być możliwie krótka i deterministyczna.

Przydatne punkty kontrolne:

```text
[BOOT] Multiboot2 OK
[BOOT] memory map OK
[BOOT] bootstrap paging OK
[BOOT] Long Mode OK
[BOOT] kernel stack OK
[BOOT] kernel_main
[PMM] ...
[VMM] ...
[GDT] ...
[IDT] ...
[APIC] ...
[HPET] ...
```

Nie należy pozostawiać bardzo szczegółowego spamu diagnostycznego w normalnym buildzie.

Rozbudowane śledzenie powinno być dostępne przez jawne flagi diagnostyczne.

---

## 3.27 Testy procesu rozruchu

Proces boot powinien być testowany oddzielnie dla co najmniej:

### Hybrydowe ISO

```text
QEMU BIOS
QEMU UEFI
```

### Dysk zainstalowany

```text
UEFI bez podłączonego ISO
```

### Aktualizacje

```text
A potwierdzony → próba B → potwierdzenie B
A potwierdzony → próba B → reset przed checkpointem → rollback do A
```

### Dane Multiboot2

```text
poprawny memory map
poprawny framebuffer
moduły
nieoczekiwany / brak opcjonalnego tagu
```

### Pamięć

Po przejściu do docelowego VMM należy sprawdzić, czy:

- kernel nadal jest poprawnie zmapowany,
- framebuffer nadal jest dostępny,
- struktura Multiboot2 nie została przedwcześnie zwolniona,
- nie ma kolizji bootstrapowych tablic z PMM.

---

## 3.28 Status testów aktualnej ścieżki instalacyjnej

Audyt instalatora raportuje pomyślne testy automatyczne obejmujące m.in.:

- instalację na pustym dysku,
- start UEFI bez ISO,
- reinstalację,
- ponowny start bez ISO,
- zachowanie drugiego dysku,
- start hybrydowego ISO jako USB w środowisku testowym.

Należy jednak zachować rozróżnienie:

> **testy QEMU/automatyczne nie są tym samym co pełna manualna walidacja na szerokiej gamie fizycznego sprzętu.**

Aktualny status fizycznego sprzętu instalatora pozostaje ograniczony względem środowiska wirtualnego.

---

## 3.29 Powiązanie z HAL ekranu

Po wejściu do jądra informacje framebufferu są przekazywane do warstwy grafiki.

HAL wybiera odpowiedni backend:

```text
UEFI GOP
VESA VBE
Bochs VBE
```

Dalsze przełączanie rozdzielczości zależy od możliwości aktywnego sterownika.

Aktualny podsystem ekranu:

- dynamicznie sprawdza dostępne tryby Bochs,
- posiada 15-sekundowy rollback niepotwierdzonej zmiany,
- utrzymuje generację ekranu dla zasobów zależnych od framebufferu.

Te mechanizmy należą już do fazy runtime, a nie do samego bootstrapu Long Mode.

---

## 3.30 Granica odpowiedzialności bootloadera i kernela

### GRUB odpowiada za:

- odnalezienie obrazu jądra,
- wybór slotu A/B w fazie pre-boot,
- załadowanie obrazu zgodnego z Multiboot2,
- przekazanie boot info,
- przygotowanie framebufferu zgodnie z możliwościami środowiska,
- załadowanie modułów.

### Bootstrap Bursztyn OS odpowiada za:

- zachowanie parametrów Multiboot2,
- przygotowanie minimalnego paging,
- ustawienie PAE,
- ustawienie EFER.LME,
- włączenie paging,
- załadowanie bootstrap GDT,
- wejście do Long Mode,
- przygotowanie stosu,
- wywołanie `kernel_main`.

### `kernel_main` i kolejne warstwy odpowiadają za:

- walidację informacji boot,
- inicjalizację PMM/VMM,
- pełną GDT/TSS/IDT,
- przerwania i timery,
- sterowniki,
- BSP2,
- BWS,
- GUI,
- uruchomienie Ring 3.

---

## 3.31 Podsumowanie

Aktualny proces rozruchu Bursztyn OS wykorzystuje GRUB i Multiboot2 jako kontrolowaną warstwę bootloadera, a następnie przechodzi do własnego kodu bootstrapowego i własnej architektury x86-64.

Kluczowe właściwości:

- jeden własny kernel Multiboot2,
- hybrydowy obraz ISO BIOS/UEFI,
- instalowany system UEFI z `BOOTX64.EFI`,
- boot A/B przez GRUB,
- własne wejście 32-bit → Long Mode,
- bootstrap identity paging,
- własny stos jądra przed wejściem do C++,
- dalsze przejęcie pamięci przez PMM/VMM,
- framebuffer przekazywany przez Multiboot2,
- HAL UEFI GOP / VESA VBE / Bochs VBE,
- automatyczny rollback aktualizacji A/B.

Najważniejsze korekty względem starszej wersji dokumentu:

- rozróżniono `0xE85250D6` od bootloader magic `0x36D76289`,
- tryb 1280×720 opisano jako preferencję Multiboot2, nie gwarancję,
- usunięto sztywne założenie, że bootstrap może mapować wyłącznie 2 MiB,
- rozdzielono bootstrapowy stos od późniejszych stosów Ring 0 procesów,
- dodano aktualną ścieżkę UEFI/GPT/ESP,
- dodano wybór slotów A/B i rollback,
- doprecyzowano rolę GOP, VESA i Bochs VBE,
- rozdzielono fazę boot od runtime zmiany rozdzielczości.

---

## 3.32 Źródła i dokumenty powiązane

Dokumenty Bursztyn OS:

- `01_wstep_i_filozofia.md`
- `02_architektura_systemu.md`
- `04_zarzadzanie_sprzetem.md`
- `09_tryb_graficzny.md`
- `instalator_audyt.md`
- `ustawienia_ekranu_audyt.md`
- `aktualizacja-systemu.md`

Specyfikacja zewnętrzna:

- GNU Multiboot2 Specification  
  https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html

Przy rozbieżnościach należy nadać pierwszeństwo:

1. aktualnemu `boot.S` i linker script,
2. aktualnemu kodowi `kernel.cpp`,
3. konfiguracji GRUB i aktualnemu obrazowi instalacyjnemu,
4. nowszym audytom,
5. starszej dokumentacji opisowej.
