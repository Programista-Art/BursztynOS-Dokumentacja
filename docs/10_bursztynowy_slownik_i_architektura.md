# 08. Bursztynowy Słownik i Lokalizacja Systemu

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** nazewnictwo Bursztyn OS, konwencje kodu, UTF-8, polskie API BursztynUI, nazwy formatów i podsystemów oraz zasady lokalizacji  
**Status:** aktualny słownik projektu; zastępuje starszą wersję używającą nazwy BZL, ograniczoną głównie do PS/2 i opisującą część niepotwierdzonych struktur jako stabilne ABI

---

## 8.1 Cel słownika

Bursztyn OS świadomie rozwija własne, polskie nazewnictwo techniczne.

Celem nie jest jednak zastępowanie wszystkich powszechnych nazw branżowych własnymi odpowiednikami. Projekt rozdziela dwa rodzaje terminologii:

1. **nazwy własne Bursztyn OS** — np. BWS, PZB, BursztynUI, `.bur`, `.cebula`;
2. **standardowe nazwy technologii i sprzętu** — np. x86-64, UEFI GOP, ACPI, HPET, APIC, USB HID, SATA AHCI, TCP/IPv4.

Zasada:

> **Własne elementy systemu otrzymują polskie nazwy, natomiast nazwy zewnętrznych standardów pozostają zgodne z dokumentacją techniczną ich producentów i specyfikacji.**

Dzięki temu Bursztyn OS zachowuje własną tożsamość bez utraty precyzji technicznej.

---

# 8.2 Zasady nazewnictwa

Aktualny projekt stosuje kilka równoległych konwencji.

### Warstwa użytkownika

W interfejsie użytkownika używany jest pełny język polski:

```text
Ustawienia ekranu
Aktualizacje Systemu
Eksplorator plików
Notatnik
Kalkulator
Tapety
Powłoka Bursztynowa
```

Tekst jest kodowany w UTF-8.

### Publiczne API BursztynUI

Nazwy klas i kontrolek są polskie, ale w identyfikatorach źródłowych nie używa się znaków diakrytycznych:

```cpp
Okno
Przycisk
PoleTekstowe
PasekNarzedzi
DialogPliku
SelektorKoloru
Powiadomienie
ZarzadcaPopupow
```

### Kod niskopoziomowy

W kodzie C/C++/ASM preferowane są identyfikatory ASCII:

```text
snake_case
```

np.:

```cpp
wylistuj_katalog()
utworz_katalog()
poziom_zaufania
wezel_indeksowy
menedzer_okien
```

### Ścieżki systemowe

Ścieżki techniczne są utrzymywane bez polskich znaków:

```text
/programy
/ustawienia
/uzytkownicy
/tymczasowe
/piaskownica
```

---

# 8.3 Oficjalne nazwy własne Bursztyn OS

| Nazwa | Znaczenie |
|---|---|
| **Bursztyn OS** | cały system operacyjny |
| **Jądro Bursztyna** | własne jądro Ring 0 |
| **BWS** | Bursztynowe Wywołania Systemowe |
| **PZB** | Poziom Zaufania Bursztyna |
| **BSP2 / PSF** | autorski system plików |
| **BursztynUI** | framework GUI dla aplikacji Ring 3 |
| **`.bur`** | natywny plik wykonywalny Ring 3 |
| **`.cebula`** | katalogowa paczka aplikacji |
| **`opis.aplikacji`** | manifest aplikacji |
| **`.btp`** | własny format tapet |
| **Powłoka Bursztynowa** | shell/terminal Bursztyn OS |
| **Menedżer Okien** | systemowy proces Ring 3 odpowiedzialny za pulpit i okna |
| **Avocado** | planowany własny język aplikacji Bursztyn OS |

---

# 8.4 PZB — poprawna nazwa modelu zaufania

Aktualną nazwą modelu bezpieczeństwa jest:

```text
PZB — Poziom Zaufania Bursztyna
```

Zakres:

```text
PZB 0–5
```

Nazwa:

```text
BZL
```

występująca w starszych dokumentach jest historyczna i nie powinna być używana w nowej dokumentacji.

PZB jest **logicznym modelem polityki kernela**.

Nie jest dodatkowym sprzętowym ringiem procesora.

Procesy użytkownika działają sprzętowo w Ring 3 niezależnie od tego, czy mają PZB-2, PZB-3, PZB-4 czy PZB-5.

---

# 8.5 BWS — Bursztynowe Wywołania Systemowe

BWS jest publicznym ABI pomiędzy Ring 3 i Ring 0.

Aktualny zakres:

```text
BWS 1–87
BWS 88 — pierwszy wolny numer
```

Starsze dokumenty mówiące o:

```text
26 BWS
```

opisują historyczny etap projektu.

W nowej dokumentacji należy używać pełnej nazwy przy pierwszym wystąpieniu:

> **Bursztynowe Wywołania Systemowe (BWS)**

a później skrótu:

```text
BWS
```

Nie ma potrzeby unikać terminu **ABI**, gdy jest technicznie przydatny. BWS jest własnym ABI Bursztyn OS.

---

# 8.6 BSP2 / PSF

Aktualny filesystem należy nazywać:

```text
BSP2 / PSF
```

Nazwy te opisują **jeden system plików**, a nie dwa oddzielne formaty.

W aktualnym kodzie format wykorzystuje:

```text
blok = 4096 B
```

oraz 64-bitowe adresowanie bloków.

Nieaktualne są starsze sformułowania:

```text
BSP ma bloki 512 B
```

oraz przedstawianie BSP i PSF jako dwóch osobnych systemów plików.

---

# 8.7 „Teczka” i „katalog”

W pierwotnej filozofii Bursztyn OS słowo:

```text
teczka
```

miało całkowicie zastąpić:

```text
katalog
folder
directory
```

W aktualnym kodzie projektu funkcjonują jednak utrwalone identyfikatory takie jak:

```cpp
wylistuj_katalog()
utworz_katalog()
```

oraz techniczne określenie katalogu w dokumentacji BSP2.

Dlatego aktualna konwencja powinna być praktyczna:

### Interfejs użytkownika

Można używać:

```text
teczka
```

jeżeli odpowiada stylistyce konkretnej aplikacji.

### Kod i dokumentacja techniczna

Preferowane jest:

```text
katalog
```

gdy odpowiada nazwie istniejącego API lub strukturze filesystemu.

Nie należy masowo zmieniać działającego ABI tylko w celu zastąpienia słowa `katalog` słowem `teczka`.

---

# 8.8 Paczka `.cebula`

`.cebula` jest natywną paczką aplikacji Bursztyn OS.

Aktualnie jest przede wszystkim **ustrukturyzowanym katalogiem**, np.:

```text
/programy/notatnik.cebula/
├── opis.aplikacji
├── notatnik.bur
├── ikona...
└── zasoby/
```

Nie należy opisywać `.cebula` jako odpowiednika:

```text
AppImage
APK
tarball
```

w sensie technicznej zgodności.

To własny model Bursztyn OS.

Nie ma także podstaw, aby obecnie twierdzić, że każda `.cebula` jest pojedynczym skompresowanym archiwum.

---

# 8.9 Plik `.bur`

`.bur` jest natywnym plikiem wykonywalnym Bursztyn OS dla:

```text
x86-64
Ring 3
```

Potwierdzony model obejmuje własny nagłówek i sygnaturę:

```text
BUR
```

oraz punkt wejścia potrzebny loaderowi.

Nie należy bez aktualnego audytu loadera zamrażać w słowniku:

- dokładnego `sizeof(NaglowekBur)`,
- wszystkich pól nagłówka,
- stałego `RING3_BASE`,
- liczby sekcji,
- dokładnych offsetów sekcji.

Te informacje należą do formalnej specyfikacji formatu `.bur`, a nie do słownika pojęć.

---

# 8.10 `opis.aplikacji`

`opis.aplikacji` jest manifestem paczki `.cebula`.

Opisuje m.in.:

- tożsamość aplikacji,
- wersję,
- program startowy,
- deklarowane potrzeby bezpieczeństwa.

Manifest nie jest źródłem bezwarunkowych uprawnień.

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

Aplikacja nie może uzyskać wyższych praw wyłącznie poprzez wpisanie ich do własnego manifestu.

---

# 8.11 `PRAWO_*`

Drobnoziarniste prawa procesu są określane przez 64-bitową maskę.

Podstawowe nazwy:

```cpp
PRAWO_PLIKI_CZYTAJ
PRAWO_PLIKI_ZAPISZ
PRAWO_SIEC
PRAWO_GUI
PRAWO_URUCHOM_PROGRAM
PRAWO_SYSTEM_CONFIG
PRAWO_STEROWNIK
PRAWO_DEBUG
```

Nazwy te są elementem aktualnego modelu bezpieczeństwa i powinny być używane konsekwentnie w dokumentacji.

---

# 8.12 BursztynUI

**BursztynUI** jest oficjalną nazwą natywnego frameworka GUI Bursztyn OS.

Nie należy zamiennie nazywać go:

```text
Bursztyn GUI
GUI Library
Widget Toolkit
```

jeżeli dokument dotyczy konkretnego frameworka.

Można oczywiście użyć ogólnego terminu „framework GUI” jako opisu jego roli.

Aktualny inwentarz projektu obejmuje około:

```text
62 publiczne kontrolki, dialogi i elementy infrastruktury
```

---

# 8.13 Nazewnictwo BursztynUI

W BursztynUI przyjęto polskie klasy bez znaków diakrytycznych.

Przykłady podstawowych kontrolek:

```text
Okno
Panel
Kolumna
Wiersz
Etykieta
Przycisk
PoleWyboru
PrzyciskOpcji
PoleTekstowe
PoleWielowierszowe
Suwak
PasekPrzewijania
RamkaGrupy
ListaWyboru
ListaRozwijana
```

Rozbudowane elementy:

```text
Tabela
Drzewo
WidokIkon
WidokMiniatur
TabelaWirtualna
ListaWirtualna
EdytorFormatowany
PoleWyszukiwania
PoleSugestii
SelektorKoloru
PowierzchniaRysunkowa
Wykres
```

Dialogi:

```text
OknoDialogowe
DialogPliku
DialogKoloru
DialogKomunikatu
```

Infrastruktura:

```text
PolecenieUI
ListaPolecen
PowierzchniaPopup
ZarzadcaPopupow
```

Nazwy takie jak `DialogPliku` są właściwymi nazwami API. Tekst wyświetlany użytkownikowi może oczywiście zawierać pełne polskie znaki.

---

# 8.14 „Popup”, „toast”, „tooltip” i terminologia mieszana

Nie wszystkie terminy muszą być sztucznie tłumaczone w kodzie, jeśli projekt ma już utrwaloną nazwę.

Aktualne API posiada m.in.:

```text
PowierzchniaPopup
ZarzadcaPopupow
Powiadomienie
Tooltip / Podpowiedz
```

Zasada:

- publiczna nazwa klasy powinna pozostać stabilna;
- dokumentacja może podać polskie objaśnienie;
- nie należy wykonywać zmian ABI wyłącznie dla językowej czystości.

---

# 8.15 „Planista”

Oficjalnym polskim określeniem schedulera może być:

```text
Planista
```

Aktualny Bursztyn OS wykorzystuje:

```text
Round-Robin
```

i pozostaje systemem jednordzeniowym.

W dokumentacji technicznej można pisać:

> **Planista (scheduler)**

przy pierwszym wystąpieniu, aby zachować jednoznaczność dla osób znających standardową terminologię systemową.

---

# 8.16 „Włókno” — status terminu

Starszy słownik definiował:

```text
Włókno = thread
```

i pokazywał przykładową strukturę `WloknoKontrolne`.

Aktualny system ma procesy, prywatne przestrzenie adresowe, stosy Ring 0 oraz scheduler, ale dokumentacja nie potwierdza jeszcze kompletnego publicznego modelu wielu niezależnych wątków/włókien wewnątrz jednego procesu.

Dlatego:

> **„Włókno” pozostaje zarezerwowanym polskim terminem dla przyszłego modelu thread, ale nie należy dziś przedstawiać `WloknoKontrolne` jako istniejącego, stabilnego ABI jądra bez potwierdzenia w aktualnym kodzie.**

Obecny opis planisty powinien koncentrować się na procesach i kontekstach wykonywania, które rzeczywiście istnieją.

---

# 8.17 „Zarządca Pamięci”

W dokumentacji można używać polskiego określenia:

```text
Zarządca Pamięci Fizycznej
Zarządca Pamięci Wirtualnej
```

oraz standardowych skrótów:

```text
PMM
VMM
```

Preferowany zapis przy pierwszym wystąpieniu:

> **Zarządca Pamięci Fizycznej (PMM)**

> **Zarządca Pamięci Wirtualnej (VMM)**

Skróty PMM/VMM są powszechnie rozumiane i nie ma potrzeby tworzyć dla nich dodatkowych własnych akronimów.

---

# 8.18 „Ramka”, „strona” i „blok”

Te pojęcia powinny pozostać rozdzielone.

### Ramka pamięci fizycznej

```text
frame
```

jednostka zarządzana przez PMM.

### Strona pamięci wirtualnej

```text
page
```

jednostka mapowania VMM.

### Blok BSP2

```text
4096 B
```

jednostka logiczna filesystemu.

### Sektor urządzenia

aktualnie w obsługiwanym instalatorze:

```text
512 B / 512e
```

Nie należy używać tych słów zamiennie.

---

# 8.19 „Menedżer Okien”

Oficjalna polska nazwa procesu desktopowego:

```text
Menedżer Okien
```

Techniczna nazwa pliku:

```text
menedzer_okien.bur
```

Menedżer Okien działa w Ring 3 i obsługuje m.in.:

- pulpit,
- pasek zadań,
- Menu Start,
- okna,
- focus,
- Z-Order,
- część integracji GUI.

Nie należy utożsamiać Menedżera Okien z całym kompozytorem Ring 0 lub z BursztynUI.

---

# 8.20 „Kompozytor”

Termin:

```text
kompozytor
```

pozostaje standardowym, poprawnym polskim określeniem komponentu składającego powierzchnie graficzne.

Aktualna architektura obejmuje m.in.:

- powierzchnie aplikacji,
- popupy top-level,
- damage starych i nowych obszarów dla wybranych ścieżek,
- końcowy kursor jako overlay,
- pacing pełnej klatki.

Nie należy w słowniku deklarować pełnego „damage compositor”, ponieważ bazowa ścieżka nadal nie jest kompletnym tile/damage rendererem.

---

# 8.21 Warstwa graficzna — poprawna terminologia

Aktualny system posiada **trzy backendy ekranu**:

```text
UEFI GOP
VESA VBE
Bochs VBE
```

Stary opis mówiący o „dwóch sterownikach” jest nieaktualny.

Role:

### UEFI GOP

Framebuffer dostarczany w ścieżce UEFI.

### VESA VBE

Framebuffer w środowiskach legacy/BIOS.

### Bochs VBE

Backend używany głównie w QEMU i środowiskach wirtualnych; obecnie najważniejszy dla runtime przełączania rozdzielczości.

Nie należy tłumaczyć nazw standardów GOP/VBE na autorskie polskie akronimy.

---

# 8.22 HAL

Skrót:

```text
HAL — Hardware Abstraction Layer
```

może być używany w dokumentacji, ponieważ jest standardowym terminem architektonicznym.

Polskie objaśnienie:

```text
warstwa abstrakcji sprzętu
```

jest zalecane przy pierwszym wystąpieniu.

W Bursztyn OS warstwa grafiki wykorzystuje m.in. abstrakcję `SterownikEkranu` i konkretne backendy.

---

# 8.23 UTF-8

Bursztyn OS używa UTF-8 w warstwie użytkownika i aplikacji.

Obsługiwane są polskie znaki:

```text
ą ć ę ł ń ó ś ź ż
Ą Ć Ę Ł Ń Ó Ś Ź Ż
```

UTF-8 jest wykorzystywany między innymi w:

- Powłoce Bursztynowej,
- aplikacjach Ring 3,
- BursztynUI,
- tekstach GUI,
- kontrolkach edycyjnych.

Nie należy jednak mówić, że system „odrzuca ASCII”.

UTF-8 jest kompatybilny z 7-bitowym ASCII, więc poprawniejsze sformułowanie brzmi:

> **Bursztyn OS nie ogranicza warstwy tekstowej do ASCII i wykorzystuje UTF-8 jako podstawowe kodowanie tekstu użytkownika.**

---

# 8.24 Fonty i renderowanie tekstu

Starsza dokumentacja wskazuje na własny renderer znaków i proporcjonalny font Unicode.

Aktualny system obsługuje:

- UTF-8,
- proporcjonalne szerokości glifów,
- polskie znaki,
- BWS do pomiaru szerokości/wysokości fontu,
- renderowanie tekstu w BursztynUI.

Nie należy jednak utożsamiać:

```text
font 16×16
```

z gwarancją, że każdy glif ma stałą szerokość 16 pikseli.

Przy foncie proporcjonalnym wymiary glifu i metryki tekstu są odrębnymi pojęciami.

---

# 8.25 Klawiatura: PS/2 i USB HID

Starszy słownik opisywał wyłącznie klawiaturę PS/2.

Aktualny Bursztyn OS posiada również:

```text
USB xHCI
USB HID keyboard
USB HID mouse
```

PS/2 pozostaje starszą ścieżką wejścia.

Znormalizowane zdarzenia powinny trafiać do wspólnego Event ABI niezależnie od źródła sprzętowego.

---

# 8.26 Physical key i `TEXT_INPUT`

Nowoczesny Event ABI rozdziela:

```text
fizyczny klawisz
```

od:

```text
TEXT_INPUT Unicode
```

Aktualne zdarzenia:

```text
event 23 — physical key
event 24 — Unicode TEXT_INPUT
```

Dzięki temu:

- skróty korzystają z klawisza fizycznego,
- edytory tekstu korzystają z `TEXT_INPUT`,
- wejście tekstowe nie musi być rekonstruowane w każdej aplikacji z raw scancode.

---

# 8.27 Modyfikatory klawiatury

Event ABI zawiera maskę modyfikatorów, obejmującą m.in.:

```text
SHIFT
CTRL
ALT
SUPER
CAPS
NUM
```

Starszy opis AltGr jako ręcznego „wstrzykiwania bajtów UTF-8” z PS/2 nie opisuje już całego aktualnego modelu.

Pełne:

- layouty klawiatury,
- dead keys,
- composition,
- IME

pozostają przyszłymi elementami warstwy Ring 3/system service.

---

# 8.28 Mysz, koło i precyzyjny wskaźnik

Aktualne Event ABI zawiera:

```text
event 25 — vertical/horizontal wheel
event 26 — precision pointer
```

Obecne pokrycie sprzętowe jest częściowe:

- USB HID wheel działa dla obsługiwanych raportów,
- PS/2 wheel negotiation nie jest jeszcze kompletne,
- arbitrary HID report descriptor parsing pozostaje przyszłością,
- obecne PS/2 i boot-HID używają głównie ograniczonych relatywnych delta.

W słowniku należy więc rozróżniać:

```text
kontrakt Event ABI
```

od:

```text
pełne wsparcie każdego sprzętu HID
```

---

# 8.29 „Focus”, „capture” i „Z-Order”

W kodzie GUI pozostają utrwalone terminy techniczne:

```text
focus
capture
Z-Order
```

Można je objaśniać po polsku:

- **focus** — aktywny odbiorca wejścia klawiatury,
- **capture** — czasowe przejęcie zdarzeń wskaźnika,
- **Z-Order** — kolejność warstw od tyłu do przodu.

Nie należy na siłę zmieniać nazw publicznego API, jeśli są już częścią istniejącego kontraktu BWS/BursztynUI.

---

# 8.30 „Powiadomienie”

Dla krótkiej wiadomości GUI używana jest kontrolka:

```text
Powiadomienie
```

W dokumentacji można doprecyzować:

```text
toast
```

jako powszechnie używany termin branżowy.

Preferowany zapis:

> **Powiadomienie (toast)**

przy pierwszym użyciu.

---

# 8.31 „Motyw”

Aktualny system posiada podsystem motywów.

BWS:

```text
86 — pobranie aktywnego motywu
87 — polecenia motywu
```

Termin:

```text
Motyw
```

jest oficjalnym polskim określeniem theme.

Obsługiwane działania obejmują m.in.:

```text
ustaw
podglad
zatwierdz
przywroc
listuj
```

Mutacje systemowego motywu wymagają zaufanego procesu i odpowiednich praw.

---

# 8.32 „Aktualizacja A/B”

Mechanizm systemowych aktualizacji należy określać jako:

```text
aktualizacje A/B
```

lub:

```text
mechanizm slotów A/B
```

Podstawowe terminy:

```text
slot aktywny
slot nieaktywny
slot oczekujący
próbny rozruch
punkt kontrolny gotowości
rollback / wycofanie
```

Termin `rollback` może pozostać w nawiasie, ponieważ jest jednoznaczny i szeroko stosowany.

---

# 8.33 „Rollback” i „wycofanie”

W polskim tekście użytkowym preferowane jest:

```text
automatyczne wycofanie
```

W dokumentacji technicznej:

```text
rollback
```

jest dopuszczalny.

Zalecany zapis przy pierwszym wystąpieniu:

> **automatyczne wycofanie (rollback)**

---

# 8.34 „Piaskownica”

Polskie określenie:

```text
Piaskownica
```

odpowiada terminowi:

```text
sandbox
```

PZB-5 jest przeznaczone dla najmniej zaufanego kodu.

Należy jednak zachować ostrożność:

> pełny kernel-enforced per-app sandbox filesystemu nie jest jeszcze udokumentowany jako kompletny dla wszystkich ścieżek.

Dlatego słowo „Piaskownica” opisuje model docelowy PZB-5, a nie automatyczną gwarancję kompletnej izolacji każdej funkcji systemu.

---

# 8.35 „Pulpit”

Oficjalny termin warstwy desktopowej:

```text
Pulpit
```

Pulpit jest częścią środowiska obsługiwanego przez Menedżer Okien.

Warstwa pulpitu ma specjalną pozycję Z-Order i nie jest zwykłym oknem aplikacji.

---

# 8.36 „Pasek Zadań” i „Menu Start”

Aktualne nazwy widoczne użytkownikowi:

```text
Pasek Zadań
Menu Start
```

mogą być używane jako nazwy interfejsu Bursztyn OS.

„Menu Start” jest terminem UI projektu i nie oznacza zgodności API z Windows.

---

# 8.37 „Powłoka Bursztynowa”

Oficjalna nazwa systemowego shell/terminala:

```text
Powłoka Bursztynowa
```

W dokumentacji technicznej można dopisać:

```text
shell
```

jako opis kategorii programu.

Przykład:

> **Powłoka Bursztynowa — natywny shell Ring 3.**

---

# 8.38 „Eksplorator plików”

Dla użytkowej aplikacji do zarządzania filesystemem można używać:

```text
Eksplorator plików
```

Nie należy jednak zakładać, że termin oznacza zgodność z interfejsem lub protokołami programu o tej samej nazwie z innych systemów.

---

# 8.39 „Instalator”

Graficzny instalator jest aplikacją Ring 3.

Aktualna ścieżka:

```text
/programy/instalator.cebula/instalator.bur
```

uruchamiana jest w trybie instalacyjnym.

Termin:

```text
Instalator Bursztyn OS
```

jest zalecany w GUI i dokumentacji użytkowej.

---

# 8.40 „Aktualizacje Systemu”

Aplikacja obsługująca mechanizm OTA/A-B nosi nazwę:

```text
Aktualizacje Systemu
```

Krytyczne operacje nie są wykonywane bezpośrednio przez GUI, lecz przez kontrolowane BWS i kod systemowy.

---

# 8.41 Avocado

**Avocado** jest nazwą własnego, planowanego języka programowania dla ekosystemu Bursztyn OS.

Aktualny status:

```text
roadmapa / kierunek rozwoju
```

Nie należy obecnie dokumentować jako zakończonych:

- kompilatora Avocado,
- automatycznej generacji `.bur`,
- automatycznego generowania `opis.aplikacji`,
- kompletnego packagera `.cebula`.

Docelowy pipeline:

```text
Avocado
   ↓
kompilator
   ↓
program.bur
+
opis.aplikacji
   ↓
paczka .cebula
   ↓
Ring 3
```

jest planem, nie ukończonym obecnie toolchainem.

---

# 8.42 Standardowe skróty, których nie należy tłumaczyć

Dla zachowania zgodności z dokumentacją branżową należy pozostawić standardowe nazwy:

```text
x86-64
UEFI
GOP
VESA VBE
Bochs VBE
ACPI
APIC
LAPIC
HPET
RTC
PCI
SATA
AHCI
LBA
USB
xHCI
HID
HDA
E1000
DMA
MMIO
TCP
UDP
IPv4
ICMP
DHCP
DNS
HTTP
HTTPS
TLS
SHA-256
RSA-2048
CRC32
GPT
ESP
FAT32
```

Można dodawać polskie objaśnienia, ale nie należy tworzyć nowych, własnych skrótów dla istniejących standardów.

---

# 8.43 Użycie angielskich terminów w kodzie

Polska tożsamość projektu nie wymaga eliminacji każdego angielskiego słowa z kodu.

Angielska nazwa jest uzasadniona, gdy:

- jest nazwą rejestru sprzętowego,
- pochodzi bezpośrednio ze specyfikacji,
- jest elementem ABI urządzenia,
- opisuje standard branżowy,
- jej tłumaczenie utrudniłoby porównanie z dokumentacją producenta.

Przykłady poprawne:

```text
IA32_LSTAR
RSP
CR3
PTE
MMIO
BAR
framebuffer
hotspot
stride
```

Polskie nazewnictwo powinno zwiększać czytelność projektu, a nie utrudniać debugowanie sprzętu.

---

# 8.44 ASCII w kodzie, UTF-8 w treści

Aktualna praktyczna zasada:

```text
identyfikatory kodu
→ ASCII

teksty użytkownika i dokumentacja
→ UTF-8
```

Przykład:

```cpp
void ustaw_tytul_okna(const char* tekst_utf8);
```

i tekst:

```text
"Ustawienia ekranu"
```

Takie rozdzielenie pozwala uniknąć problemów toolchainu i jednocześnie zachować pełny polski interfejs.

---

# 8.45 Nazwy plików źródłowych

Dla kodu projektu preferowane są nazwy:

```text
snake_case
```

bez diakrytyki.

Przykłady:

```text
menedzer_okien.cpp
zarzadca_pamieci.cpp
dialog_pliku.cpp
selektor_koloru.cpp
bws_nowoczesne.cpp
aktualizacja_systemu.cpp
```

Nazwy plików dokumentacji również powinny być stabilne i przewidywalne:

```text
08_bursztynowy_slownik_i_architektura.md
```

---

# 8.46 Wielkość liter w API

Dla typów/klas BursztynUI preferowane są nazwy typu:

```text
PascalCase
```

np.:

```cpp
DialogPliku
PoleWyszukiwania
WidokMiniatur
```

Dla funkcji niskopoziomowych i ścieżek systemowych w wielu częściach projektu używany jest:

```text
snake_case
```

Nie należy mieszać konwencji w ramach jednego publicznego modułu bez przyczyny.

---

# 8.47 Przykład współczesnego nazewnictwa GUI

Przykład kodu dokumentacyjnego:

```cpp
Okno okno;
Kolumna zawartosc;

Etykieta tytul("Ustawienia ekranu");
PoleWyboru rozdzielczosc;
Przycisk zastosuj("Zastosuj");

zawartosc.dodaj(&tytul);
zawartosc.dodaj(&rozdzielczosc);
zawartosc.dodaj(&zastosuj);
```

Jest to przykład stylu nazewniczego, a nie gwarantowana sygnatura aktualnych konstruktorów/metod.

---

# 8.48 Przykład współczesnego nazewnictwa systemowego

```cpp
struct Proces {
    uint64_t pid;
    uint8_t poziom_zaufania;
    uint64_t uprawnienia;

    // przestrzen_adresowa
    // stos_ring0
    // kolejka_zdarzen
    // zasoby_gui
};
```

To model logiczny.

Nie należy traktować takiego przykładu jako publicznego ABI struktury kernela.

---

# 8.49 Przykład wpisu katalogowego

Starszy dokument przedstawiał:

```cpp
struct teczka_wpis {
    uint32_t id_wezla;
    char nazwa[32];
} __attribute__((packed));
```

Aktualna dokumentacja BSP2 nie powinna utrwalać tego layoutu bez audytu `psf.h`.

Dlatego w słowniku wystarczy model:

```text
wpis katalogu:
nazwa
+
identyfikator węzła
```

Dokładne:

- rozmiary pól,
- alignment,
- packing,
- długość nazwy

należą do specyfikacji BSP2.

---

# 8.50 Przykład `NaglowekBur`

Analogicznie stary przykład:

```cpp
struct NaglowekBur {
    uint8_t magia[4];
    uint64_t punkt_wejscia;
    uint64_t tekst_wirtualny;
};
```

należy traktować jako historyczny przykład koncepcji.

Słownik powinien potwierdzać tylko:

```text
NaglowekBur
→ nagłówek własnego formatu .bur
→ magic BUR
→ informacje potrzebne loaderowi
```

Dokładny binarny layout wymaga osobnej, aktualnej specyfikacji loadera.

---

# 8.51 Polskie komunikaty błędów

Warstwa użytkownika powinna preferować jasne, polskie komunikaty:

```text
Brak dostępu
Nie znaleziono pliku
Nie można zastosować rozdzielczości
Aktualizacja nie została zatwierdzona
Połączenie sieciowe nie powiodło się
```

W logach developerskich można zachować standardowe skróty i nazwy sprzętowe:

```text
[HPET]
[AHCI]
[E1000]
[USB]
[BWS]
[VMM]
```

Log powinien ułatwiać debugowanie, nie być demonstracją lokalizacji językowej.

---

# 8.52 Polskie nazwy a kompatybilność dokumentacji

Przykład zalecanego stylu:

> **Zarządca Pamięci Wirtualnej (VMM)** mapuje rejestry **HPET MMIO** jako strony supervisor z odpowiednimi atrybutami cache.

Nie zaleca się tworzenia sztucznych zamienników:

```text
HPET → „Bursztynowy Zegar Precyzyjny”
MMIO → „Pamięcioport”
```

ponieważ utrudniałoby to pracę z dokumentacją sprzętu.

---

# 8.53 Lokalizacja a bezpieczeństwo

Tekst widoczny użytkownikowi może być lokalizowany.

Natomiast wartości wykorzystywane jako:

- identyfikatory ABI,
- ID motywów,
- prawa `PRAWO_*`,
- nazwy struktur wymienianych z kernelem,
- magic numbers,

powinny pozostawać stabilne i niezależne od języka UI.

Przykład:

```text
wyświetlana nazwa: „Grafit”
wewnętrzny identyfikator: grafit
```

Pozwala to zmienić język interfejsu bez zmiany ABI.

---

# 8.54 Lokalizacja motywów i aplikacji

Nazwy wyświetlane użytkownikowi nie powinny być używane jako jedyny identyfikator obiektu.

Dotyczy to m.in.:

- motywów,
- aplikacji,
- formatów clipboard,
- urządzeń,
- ustawień.

Preferowany model:

```text
stabilne ID techniczne
+
lokalizowana nazwa użytkowa
```

---

# 8.55 Nazwy typów zdarzeń

Event ABI jest publicznym kontraktem, dlatego nazwy techniczne mogą być opisane w dokumentacji zarówno po polsku, jak i przez ustalony symbol kodowy.

Przykład:

```text
TEXT_INPUT — wejście tekstowe Unicode
```

Nie należy zmieniać symbolu ABI tylko dlatego, że interfejs użytkownika jest po polsku.

---

# 8.56 Słownik zalecanych zapisów

| Zamiast | Zalecany zapis |
|---|---|
| BZL | **PZB** |
| Bursztynowy Poziom Zaufania | **Poziom Zaufania Bursztyna** |
| BSP jako 512 B FS | **BSP2 / PSF, blok 4 KiB** |
| EXE Bursztyna | **plik `.bur`** |
| archiwum `.cebula` | **paczka/katalog `.cebula`** |
| GUI library | **BursztynUI** |
| system call | **BWS** lub „wywołanie systemowe BWS” |
| scheduler | **Planista (scheduler)** |
| memory manager | **Zarządca Pamięci (PMM/VMM)** |
| sandbox | **Piaskownica (sandbox)** |
| theme | **Motyw** |
| toast | **Powiadomienie (toast)** |
| rollback | **wycofanie (rollback)** |
| directory w UI | **teczka** lub **katalog**, zależnie od kontekstu |
| directory w kodzie BSP2 | **katalog** zgodnie z istniejącym API |

---

# 8.57 Terminy, których status zmienił się od starego dokumentu

### `BZL`

Status:

```text
historyczny
```

Zastąpione przez:

```text
PZB
```

### `Włókno`

Status:

```text
termin zarezerwowany dla modelu thread
```

Nie przedstawiać przykładowej struktury jako aktualnego ABI bez audytu.

### `RING3_BASE = 0x1000000`

Status:

```text
historyczny/przykładowy detal loadera
```

Nie przedstawiać jako gwarantowanego kontraktu bez aktualnego audytu `.bur`.

### PS/2 jako jedyne wejście

Status:

```text
nieaktualne
```

Aktualnie istnieje również:

```text
USB xHCI + HID
```

### Dwa backendy grafiki

Status:

```text
nieaktualne
```

Aktualnie:

```text
UEFI GOP
VESA VBE
Bochs VBE
```

### ASCII kontra UTF-8

Status:

```text
stare sformułowanie zbyt kategoryczne
```

Aktualnie:

```text
UTF-8 jest podstawowym kodowaniem tekstu użytkownika
i pozostaje zgodne z ASCII dla kodów 0–127.
```

---

# 8.58 Zasady dla przyszłych nazw

Nowa nazwa w Bursztyn OS powinna spełniać kilka warunków:

1. być zrozumiała po polsku;
2. nie kolidować z istniejącym znaczeniem technicznym;
3. nie ukrywać ważnego standardu sprzętowego;
4. być możliwa do zapisania w kodzie bez znaków diakrytycznych;
5. mieć stabilny identyfikator techniczny;
6. nie wymuszać zerwania ABI tylko dla kosmetyki;
7. pasować do obecnego stylu BursztynUI i BWS.

Przykład dobry:

```text
DialogPliku
```

Przykład gorszy:

```text
BursztynowyPodsystemZaawansowanejManipulacjiDokumentami
```

Nazwa powinna być polska, ale również krótka i praktyczna.

---

# 8.59 Zasada dokumentacyjna

W dokumentach projektowych należy jasno odróżniać:

```text
nazwa oficjalna
```

od:

```text
opis działania
```

Przykład:

```text
BWS — nazwa własnego ABI
SYSCALL — instrukcja procesora używana przez BWS
```

oraz:

```text
BursztynUI — nazwa frameworka
GUI — kategoria technologii
```

Nie są to pojęcia konkurencyjne.

---

# 8.60 Podsumowanie

Współczesne nazewnictwo Bursztyn OS powinno być polskie tam, gdzie system tworzy własny koncept lub własne publiczne API, ale nie powinno utrudniać pracy z istniejącymi standardami sprzętowymi.

Aktualne najważniejsze nazwy projektu:

```text
Jądro Bursztyna
BWS
PZB
BSP2 / PSF
BursztynUI
.bur
.cebula
opis.aplikacji
.btp
Powłoka Bursztynowa
Menedżer Okien
Planista
Piaskownica
Motyw
Powiadomienie
Aktualizacje Systemu
Avocado
```

Zewnętrzne standardy pozostają pod oficjalnymi nazwami:

```text
x86-64
UEFI GOP
VESA VBE
Bochs VBE
ACPI
APIC
HPET
SATA AHCI
USB xHCI/HID
Intel E1000
Intel HDA
TCP/IPv4
TLS
GPT
FAT32
```

Taki model pozwala zachować „polską duszę” Bursztyn OS bez tworzenia terminologii, która utrudniałaby rozwój, audyt i porównywanie kodu z dokumentacją sprzętu.

---

## 8.61 Dokumenty powiązane

Słownik powinien pozostawać zsynchronizowany z:

- `01_wstep_i_filozofia.md` — filozofia i tożsamość systemu,
- `02_architektura_systemu.md` — PZB, Ring 0/Ring 3,
- `04_zarzadzanie_sprzetem.md` — nazwy standardów sprzętowych,
- `05_bursztynowy_system_plikow.md` — BSP2/PSF, katalogi i ścieżki,
- `06_wywolania_systemowe.md` — BWS 1–87,
- `07_ekosystem_i_formaty.md` — `.bur`, `.cebula`, `opis.aplikacji`, Avocado,
- `09_tryb_graficzny.md` — UEFI GOP, VESA VBE i Bochs VBE,
- `11_audyt_bws_nowoczesne.md` — Event ABI i nowoczesne rozszerzenia BWS,
- `aktualizacja-systemu.md` — terminologia A/B i rollback.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod i publiczne API,
2. `bws_api.h`,
3. aktualne nazwy klas BursztynUI,
4. aktualny BSP2/PSF,
5. nowsze audyty,
6. starsza dokumentacja historyczna.

---

## 8.62 Najważniejsze zmiany względem starej wersji

Wprowadzono następujące korekty:

- **BZL → PZB**;
- BSP → aktualne **BSP2 / PSF**;
- usunięto założenie, że „teczka” musi zastąpić słowo „katalog” również w istniejącym API;
- `.cebula` opisano jako katalogową paczkę, a nie obowiązkowo archiwum;
- `.bur` zachowano jako własny format Ring 3, ale usunięto niepotwierdzony stabilny layout z dokumentu słownikowego;
- termin „Włókno” oznaczono jako przyszły termin dla modelu thread, bez udawania istniejącego publicznego ABI;
- dodano **BursztynUI** i współczesne nazwy kontrolek;
- dodano motywy, powiadomienia, popupy, A/B i współczesną terminologię aplikacyjną;
- poprawiono listę grafiki do **UEFI GOP + VESA VBE + Bochs VBE**;
- dodano **USB xHCI/HID** obok PS/2;
- dodano rozdzielenie physical key i Unicode `TEXT_INPUT`;
- poprawiono opis ASCII/UTF-8;
- rozdzielono własne nazwy Bursztyn OS od standardów branżowych;
- Avocado opisano zgodnie z aktualnym statusem jako kierunek rozwoju.
