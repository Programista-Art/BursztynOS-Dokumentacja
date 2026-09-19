# 07. Ekosystem Aplikacji i Formaty Plików

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** aplikacje Ring 3, `.bur`, `.cebula`, `opis.aplikacji`, BursztynUI, formaty zasobów, PZB/BWS oraz planowana integracja z językiem Avocado  
**Status:** aktualny opis ekosystemu aplikacji; zastępuje starszy dokument przedstawiający część planowanych elementów Avocado jako już działające

---

## 7.1 Rola ekosystemu Ring 3

Bursztyn OS posiada własną przestrzeń użytkownika działającą sprzętowo w **Ring 3**.

Aplikacje użytkownika nie są wykonywane wewnątrz jądra. Każdy proces otrzymuje własny kontekst wykonania i komunikuje się z Ring 0 przez **BWS — Bursztynowe Wywołania Systemowe**.

Aktualny stos aplikacyjny można uprościć do:

```text
aplikacja Ring 3
      │
      ├── kod natywny .bur
      ├── BursztynUI
      ├── zasoby
      ├── opis.aplikacji
      │
      ▼
BWS / SYSCALL
      │
      ▼
Jądro Bursztyna
      │
      ├── PZB / PRAWO_*
      ├── BSP2/PSF
      ├── GUI / kompozytor
      ├── sieć
      ├── IPC / SHM
      └── inne usługi systemowe
```

Bursztyn OS nie używa ELF ani PE/COFF jako **natywnego formatu aplikacji użytkownika**. Własnym formatem wykonywalnym pozostaje `.bur`.

---

## 7.2 Główne elementy ekosystemu

Obecny model aplikacji opiera się na trzech podstawowych elementach:

1. **`.bur`** — natywny plik wykonywalny Ring 3;
2. **`.cebula`** — katalogowa paczka aplikacji;
3. **`opis.aplikacji`** — manifest aplikacji i deklarowanych możliwości.

Dodatkowo aplikacja może korzystać z:

- zasobów graficznych,
- ikon,
- plików konfiguracyjnych,
- tapet `.btp`,
- motywów systemowych,
- własnych danych w BSP2,
- BursztynUI,
- BWS,
- SHM,
- IPC,
- clipboardu MIME,
- socketów TCP/IPv4.

---

# 7.3 Format `.bur`

`.bur` jest natywnym formatem wykonywalnym Bursztyn OS dla architektury:

```text
x86-64
```

Program `.bur` jest przeznaczony do wykonania w:

```text
Ring 3
```

i komunikuje się z systemem przez BWS.

Przykładowe pliki:

```text
notatnik.bur
kalkulator.bur
menedzer_okien.bur
shell.bur
```

---

## 7.4 Nagłówek `.bur`

Starsza dokumentacja opisuje własny nagłówek:

```cpp
struct NaglowekBur
```

z sygnaturą:

```text
BUR\0
```

oraz polem punktu wejścia i informacjami o mapowaniu kodu.

Historyczny przykład:

```cpp
struct NaglowekBur {
    uint8_t  magia[4];
    uint64_t punkt_wejscia;
    uint64_t tekst_wirtualny;
};
```

W starszym rozdziale `07_ekosystem_i_formaty.md` opisano także bardziej rozbudowany wariant sekcji kodu i danych.

### Ważne

Aktualne dostępne dokumenty nie stanowią wystarczającej podstawy, aby zamrozić jako oficjalne ABI dokładny layout całego nagłówka `.bur`, jego rozmiar lub liczbę sekcji.

Dlatego nowa dokumentacja powinna rozróżniać:

```text
potwierdzone:
- własny format .bur,
- sygnatura BUR,
- punkt wejścia,
- mapowanie do przestrzeni Ring 3

wymaga aktualnego audytu loadera:
- dokładny sizeof(NaglowekBur),
- wszystkie pola,
- wersjonowanie,
- alignment,
- dokładny layout sekcji,
- zasady relokacji
```

Przed opublikowaniem stabilnej specyfikacji binarnej należy porównać dokument z aktualnym kodem loadera i dodać `static_assert(sizeof(...))`.

---

## 7.5 `RING3_BASE`

Starsze dokumenty używają przykładowej bazy:

```text
RING3_BASE = 0x1000000
```

czyli 16 MiB.

Nie należy jednak traktować tej wartości jako niezmiennego publicznego kontraktu formatu `.bur`, dopóki nie zostanie to potwierdzone przez aktualny loader i publiczne ABI.

Poprawna zasada architektoniczna:

> loader mapuje program wyłącznie do dozwolonej przestrzeni użytkownika i nie pozwala nagłówkowi `.bur` nadpisać jądra, MMIO ani innych procesów.

---

# 7.6 Ładowanie programu `.bur`

Uruchomienie programu odbywa się przez kontrolowaną ścieżkę jądra.

BWS:

```text
BWS 10 — BWS_URUCHOM_PROGRAM
```

jest historycznym publicznym punktem uruchamiania aplikacji.

Koncepcyjny przepływ:

```text
ścieżka programu
      ↓
checked user-copy
      ↓
kontrola PZB / PRAWO_URUCHOM_PROGRAM
      ↓
BSP2 odczytuje .bur
      ↓
walidacja nagłówka
      ↓
utworzenie procesu
      ↓
nowa przestrzeń adresowa
      ↓
mapowanie sekcji Ring 3
      ↓
stosy procesu
      ↓
PZB + PRAWO_*
      ↓
punkt wejścia
      ↓
wykonanie Ring 3
```

Loader nie powinien ufać wartościom z pliku `.bur` tylko dlatego, że plik znajduje się w `/programy`.

---

## 7.7 Walidacja loadera

Plik wykonywalny jest niezaufanym wejściem dla jądra.

Loader powinien sprawdzać co najmniej:

- magic,
- wersję formatu, jeśli istnieje,
- rozmiar nagłówka,
- offsety sekcji,
- rozmiary sekcji,
- overflow `offset + size`,
- alignment,
- kanoniczność adresów,
- granice przestrzeni Ring 3,
- nakładanie sekcji,
- punkt wejścia,
- prawa stron.

Nie wolno mapować sekcji wykonywalnej jako zapisywalnej tylko dla wygody, jeśli aktualny model pamięci pozwala zachować zasadę W^X.

Docelowa polityka:

```text
kod:
read + execute

dane:
read + write

stos:
read + write
NX
```

jeżeli aktualny VMM i loader wspierają tę ochronę.

---

# 7.8 `.cebula` — paczka aplikacji

Starsza wersja dokumentu określała `.cebula` jako „archiwum”.

To nie jest obecnie najprecyzyjniejszy opis.

W aktualnym BSP2 paczka `.cebula` jest przede wszystkim **ustrukturyzowaną teczką/katalogiem aplikacji**.

Przykład:

```text
/programy/notatnik.cebula/
├── opis.aplikacji
├── notatnik.bur
├── ikona...
└── zasoby/
```

`.cebula` grupuje:

- program wykonywalny,
- manifest,
- ikonę,
- zasoby,
- potencjalne pliki konfiguracyjne aplikacji.

Nie należy zakładać, że `.cebula` jest dziś pojedynczym skompresowanym kontenerem binarnym, jeżeli bieżący system przechowuje ją jako katalog BSP2.

---

# 7.9 Lokalizacja aplikacji

Typowa ścieżka aplikacji:

```text
/programy/<nazwa>.cebula/
```

Przykład:

```text
/programy/notatnik.cebula/notatnik.bur
```

Powłoka może uruchomić program np.:

```text
uruchom /programy/notatnik.cebula/notatnik.bur
```

Menedżer Okien oraz inne komponenty systemu mogą używać danych paczki do:

- prezentowania aplikacji,
- ładowania ikon,
- uruchamiania programu,
- odczytu manifestu.

---

# 7.10 Manifest `opis.aplikacji`

`opis.aplikacji` jest tekstowym manifestem aplikacji.

Może zawierać informacje takie jak:

```text
nazwa
autor
wersja
plik_startowy
poziom_zaufania
uprawnienia
```

Przykład dokumentacyjny:

```text
nazwa = "Notatnik"
autor = "Programista Art"
wersja = "0.1"
poziom_zaufania = 4
plik_startowy = "notatnik.bur"

uprawnienia = [
    "okna",
    "pliki_czytaj",
    "pliki_zapisz"
]
```

Dokładna gramatyka parsera manifestu powinna być dokumentowana na podstawie bieżącego kodu parsera.

Powyższy przykład jest modelem logicznym, a nie deklaracją, że każdy pokazany klucz jest obowiązkowy w każdej aktualnej paczce.

---

# 7.11 PZB zamiast BZL

Starsze dokumenty używają nazwy:

```text
BZL — Bursztynowy Poziom Zaufania
```

Aktualną nazwą projektu jest:

```text
PZB — Poziom Zaufania Bursztyna
```

Nowa dokumentacja powinna konsekwentnie używać PZB.

Model obejmuje poziomy:

```text
PZB 0–5
```

ale aplikacje użytkownika nadal działają sprzętowo w Ring 3.

PZB jest polityką logiczną, a nie dodatkowym ringiem procesora.

---

# 7.12 Manifest nie przyznaje sam sobie praw

Stary dokument sugerował, że parser po prostu przepisuje żądane prawa manifestu do struktury procesu.

Bezpieczniejszy i aktualny model powinien być opisywany jako:

```text
manifest
   ↓
żądane możliwości
   ↓
parser
   ↓
polityka jądra
   ↓
dozwolony PZB
+
dozwolone PRAWO_*
   ↓
proces
```

Aplikacja nie może uzyskać np.:

```text
PZB-0
PRAWO_STEROWNIK
PRAWO_SYSTEM_CONFIG
```

wyłącznie przez wpisanie takich wartości do własnego `opis.aplikacji`.

Manifest jest **deklaracją potrzeb**, a nie źródłem autorytetu.

---

# 7.13 Podstawowe `PRAWO_*`

Aktualny model procesu używa 64-bitowej maski praw.

Podstawowe udokumentowane bity obejmują:

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

Przykładowe:

```text
"pliki_czytaj"
→ PRAWO_PLIKI_CZYTAJ

"pliki_zapisz"
→ PRAWO_PLIKI_ZAPISZ

"okna"
→ PRAWO_GUI
```

Dla trzech praw:

```text
PLIKI_CZYTAJ
PLIKI_ZAPISZ
GUI
```

historyczny przykład maski:

```text
0b00001011
```

jest zgodny z powyższym układem bitów.

---

# 7.14 Zasada najmniejszych uprawnień

Typ aplikacji nie powinien automatycznie oznaczać szerokiego zestawu praw.

Przykładowo:

### Kalkulator

```text
PZB-4
PRAWO_GUI
```

### Notatnik

```text
PZB-4
PRAWO_GUI
PRAWO_PLIKI_CZYTAJ
PRAWO_PLIKI_ZAPISZ
```

### Aplikacja sieciowa

```text
PZB-4
PRAWO_GUI
PRAWO_SIEC
```

Prawo zapisu pliku nie powinno wynikać automatycznie z posiadania prawa sieci.

---

# 7.15 BWS jako API aplikacji

Program `.bur` korzysta z usług systemowych przez BWS.

Aktualne publiczne ABI obejmuje:

```text
BWS 1–87
```

a:

```text
BWS_PIERWSZY_WOLNY = 88
```

Współczesne aplikacje mogą korzystać m.in. z:

- plików BSP2,
- event queue,
- warstw GUI,
- BursztynUI,
- monotonicznego czasu HPET,
- geometrii okna,
- informacji o monitorze,
- SHM,
- IPC,
- file watch,
- clipboardu MIME,
- kursorów,
- socketów TCP/IPv4,
- OTA dla zaufanej aplikacji systemowej,
- podsystemu motywów.

---

# 7.16 ABI rejestrów

Aktualne ABI `bws_wywolaj()` nie używa RAX jako numeru BWS.

Obowiązuje:

```text
R8  = numer BWS
R9  = argument 1
R10 = argument 2
R12 = argument 3
R13 = argument 4
RAX = wynik
```

Aplikacja nie musi implementować tego ręcznie. SDK Ring 3 powinno dostarczać wrappery.

---

# 7.17 BursztynUI

BursztynUI jest natywnym frameworkiem GUI dla aplikacji Ring 3.

Obecny inwentarz projektu obejmuje około:

```text
62 publiczne kontrolki, dialogi i elementy infrastruktury
```

Wśród nich są m.in.:

```text
Okno
Panel
Przycisk
PoleTekstowe
Tabela
Drzewo
WidokIkon
Karty
MenuGlowne
MenuKontekstowe
PasekNarzedzi
PasekStanu
OknoDialogowe
DialogPliku
DialogKoloru
DialogKomunikatu
Powiadomienie
SelektorKoloru
TabelaWirtualna
ListaWirtualna
PolecenieUI
ZarzadcaPopupow
```

BursztynUI jest warstwą Ring 3 zbudowaną na mechanizmach BWS. Kontrolka GUI nie jest osobnym syscall'em.

---

# 7.18 Dialogi nie są osobnymi BWS

Elementy takie jak:

```text
DialogPliku
DialogKoloru
DialogKomunikatu
Toast
Tooltip
MenuKontekstowe
```

są częścią BursztynUI.

Kernel udostępnia niższe mechanizmy:

- powierzchnie,
- eventy,
- clipboard,
- IPC,
- SHM,
- kursory,
- pliki.

Pozwala to rozwijać framework GUI bez rozbudowy jądra o syscall dla każdej kontrolki.

---

# 7.19 Event ABI

Aplikacje GUI odbierają zdarzenia z kolejki procesu.

Aktualny model rozdziela m.in.:

```text
physical key
TEXT_INPUT Unicode
mouse wheel
pointer
file watcher
IPC readiness
```

Współczesne aplikacje nie powinny opierać się wyłącznie na legacy:

```text
BWS 18 — globalny stan myszy
BWS 22 — stary model przejęcia myszy
```

Nowy model wykorzystuje:

- per-process event queues,
- focus,
- capture,
- hit-test.

---

# 7.20 SHM i IPC

Aplikacje Ring 3 mogą korzystać z natywnego:

```text
SHM — BWS 64–67
IPC — BWS 68–71
```

SHM zapewnia kontrolowane współdzielenie pamięci bez ujawniania fizycznych adresów Ring 3.

IPC udostępnia ograniczone endpointy i kolejki.

Pozwala to tworzyć przyszłe usługi Ring 3 bez konieczności umieszczania całej logiki w kernelu.

---

# 7.21 Clipboard MIME

BWS 73–76 dostarczają clipboard obsługujący wiele formatów MIME.

Aktualne limity udokumentowanego ABI:

```text
8 formatów
1 MiB / format
2 MiB łącznie
```

Dzięki temu aplikacje mogą wymieniać więcej niż zwykły tekst.

Historyczne BWS 52–54 dla schowka plikowego pozostają osobnym, starszym API.

---

# 7.22 File watch

BWS 72 pozwala aplikacji obserwować zmiany plików.

Mechanizm jest przydatny np. dla:

- Eksploratora plików,
- edytorów,
- widoków katalogów.

Aktualna implementacja pozostaje częściowa i nie należy jej opisywać jako kompletnego odpowiednika inotify/FSEvents.

---

# 7.23 Sieć aplikacji

BWS 79–83 udostępniają owned socket facade TCP/IPv4:

```text
create
connect
send
receive
close
```

Socket jest związany z PID i generacyjnym uchwytem.

Aktualne ograniczenie:

```text
bazowy stos TCP obsługuje jedną systemową sesję jednocześnie
```

Dlatego API nie jest jeszcze pełnym odpowiednikiem dojrzałej implementacji BSD sockets.

---

# 7.24 Format `.btp`

Bursztyn OS posiada własny format tapet:

```text
.btp
```

System posiada:

- aplikację Tapety,
- możliwość zmiany tapety,
- automatyczne ładowanie tapet,
- konwerter JPG/PNG → BTP.

`.btp` jest formatem zasobu graficznego Bursztyn OS, a nie plikiem wykonywalnym ani paczką aplikacji.

Dokładny layout binarny `.btp` powinien być dokumentowany w osobnej specyfikacji formatu.

---

# 7.25 Motywy

Nowszy Bursztyn OS posiada także podsystem motywów.

Publiczne ABI:

```text
BWS 86 — BWS_MOTYW_POBIERZ
BWS 87 — BWS_MOTYW_POLECENIE
```

Obsługiwane są wersje ABI motywu:

```text
v1
v2
```

oraz operacje takie jak:

```text
ustaw
podglad
zatwierdz
przywroc
listuj
```

Zmiana motywu jest osobnym mechanizmem od paczki `.cebula`.

Nie należy bez aktualnego audytu formatu dokumentować rozszerzenia pliku motywu jako publicznego standardu, jeśli nie jest ono potwierdzone przez źródła.

---

# 7.26 Pliki konfiguracji

Bursztyn OS używa zwykłych plików w BSP2 do przechowywania części ustawień.

Przykład:

```text
/ustawienia/ekran.cfg
```

Plik jest zapisywany dopiero po zatwierdzeniu zmiany rozdzielczości.

Konfiguracja aplikacji nie jest częścią binarnego ABI `.bur`.

Powinna być traktowana jako dane aplikacji/systemu przechowywane w filesystemie.

---

# 7.27 Lista obecnych aplikacji

Aktualny ekosystem zawiera m.in.:

- Notatnik,
- Kalkulator,
- Przeglądarkę zdjęć,
- Husarza — eksperymentalną przeglądarkę / klient WWW,
- Eksplorator plików,
- Powłokę Bursztynową,
- Ustawienia ekranu,
- Tapety,
- Aktualizacje Systemu,
- Testy Bursztyn OS,
- Snake,
- Instalator Bursztyn OS.

Nie wszystkie aplikacje mają ten sam poziom dojrzałości.

---

# 7.28 Aplikacje systemowe a zwykłe aplikacje

Nie każda paczka `.cebula` powinna mieć możliwość uzyskania tych samych praw.

Przykład:

```text
Aktualizacje Systemu
```

korzystają z kontrolowanych:

```text
BWS 84–85
```

i wymagają zaufanej polityki.

Zwykła aplikacja PZB-4 nie powinna móc samodzielnie:

- przełączyć slotu A/B,
- zapisać boot state,
- zmienić krytycznej konfiguracji systemowej.

Analogicznie BWS 87 dla mutacji motywu wymaga zaufanego procesu i `PRAWO_SYSTEM_CONFIG`.

---

# 7.29 Instalator jako aplikacja Ring 3

Graficzny instalator Bursztyn OS jest aplikacją Ring 3 uruchamianą w trybie instalacyjnym.

Tryb:

```text
bursztyn.install=1
```

uruchamia:

```text
/programy/instalator.cebula/instalator.bur
```

Interfejs instalatora działa w Ring 3, ale krytyczne operacje dyskowe pozostają po stronie kontrolowanych mechanizmów systemowych.

Jest to przykład docelowej filozofii:

> interfejs i logika użytkownika w Ring 3, operacje uprzywilejowane przez BWS.

---

# 7.30 `.cebula` a instalacja aplikacji

Aktualne źródła potwierdzają strukturę `.cebula` jako katalogowej paczki obecnej w BSP2.

Nie ma natomiast wystarczającej podstawy, aby dokumentować jako gotowy pełny package manager posiadający:

- centralne repozytoria,
- zależności,
- automatyczne aktualizacje wszystkich aplikacji,
- podpis każdej paczki,
- transakcyjne instalowanie aplikacji,
- deinstalację z rollbackiem.

Takie możliwości mogą być rozwijane później.

Obecna dokumentacja nie powinna ich sugerować jako już istniejących.

---

# 7.31 Integralność paczek

Starszy dokument wspominał o:

```text
uprawnienia/
```

i dodatkowych kluczach kryptograficznych wewnątrz `.cebula`.

Dostępne aktualne źródła nie potwierdzają, że jest to obowiązujący, wdrożony element każdej paczki.

Dlatego nie należy obecnie przedstawiać:

```text
uprawnienia/
```

jako wymaganej części standardu `.cebula`.

Podpisy RSA-2048 i SHA-256 są potwierdzone dla **systemu aktualizacji A/B**, ale nie należy automatycznie przenosić tej właściwości na wszystkie paczki aplikacji.

---

# 7.32 Avocado — aktualny status

Największa korekta względem starego dokumentu dotyczy języka **Avocado**.

Starsza wersja pisała o działającym kompilatorze, który:

- generuje `.bur`,
- automatycznie tworzy `opis.aplikacji`,
- pakuje `.cebula`,
- pozwala deklarować uprawnienia bezpośrednio w kodzie.

Aktualne dane projektu nie potwierdzają, że ten pełny pipeline jest już gotowy.

Dlatego Avocado należy obecnie opisywać jako:

> **planowany, własny język aplikacji Bursztyn OS i docelowy element toolchainu developerskiego.**

System i aplikacje są obecnie rozwijane głównie w:

```text
C
C++
assembler
```

---

# 7.33 Docelowy pipeline Avocado

Planowana architektura może wyglądać następująco:

```text
kod Avocado
     ↓
kompilator Avocado
     ↓
walidacja deklarowanych możliwości
     ↓
program.bur
     +
opis.aplikacji
     ↓
aplikacja.cebula
     ↓
BSP2
     ↓
loader Bursztyn OS
     ↓
Ring 3
```

Jest to **roadmapa**, a nie opis obecnego, ukończonego toolchainu.

---

# 7.34 Planowana deklaratywna składnia Avocado

Starsza dokumentacja przedstawiała przykład:

```text
aplikacja "Notatnik" {
    autor: "Programista Art",
    wersja: "0.1",
    poziom_zaufania: 4,
    uprawnienia: [
        "okna",
        "pliki_czytaj",
        "pliki_zapisz"
    ]
}
```

Taki model pozostaje sensownym kierunkiem projektu, ale składni nie należy jeszcze traktować jako zamrożonej specyfikacji języka Avocado bez aktualnej implementacji parsera i kompilatora.

Docelowo compiler może generować manifest na podstawie deklaracji aplikacji, ale jądro nadal musi niezależnie egzekwować politykę bezpieczeństwa.

---

# 7.35 Avocado nie może zastąpić walidacji kernela

Nawet jeżeli przyszły kompilator Avocado wygeneruje poprawny `opis.aplikacji`, jądro nie może założyć:

```text
„wygenerował to kompilator, więc ufamy”
```

Kernel nadal musi kontrolować:

- PZB,
- `PRAWO_*`,
- wskaźniki Ring 3,
- uchwyty,
- właściciela,
- limity,
- przestrzeń adresową.

Bezpieczeństwo nie może zależeć wyłącznie od poprawności toolchainu.

---

# 7.36 Publiczne SDK aplikacji

Docelowe SDK Bursztyn OS powinno ukrywać niskopoziomowe ABI rejestrów za bezpiecznymi wrapperami.

Przykład:

```cpp
bool plik_czytaj(...);
bool gui_utworz_okno(...);
uint64_t czas_monotoniczny_ns();
bool shm_utworz(...);
bool ipc_wyslij(...);
```

zamiast ręcznego wykonywania:

```asm
syscall
```

w każdej aplikacji.

SDK powinno być wersjonowane razem z publicznym ABI BWS.

---

# 7.37 Kompatybilność ABI aplikacji

BWS jest rozwijane addytywnie.

Aktualne drzewo używa:

```text
BWS 1–87
```

Dlatego aplikacja powinna:

- korzystać z publicznych wrapperów,
- nie zależeć od wewnętrznych struktur kernela,
- nie dereferencjonować adresów systemowych,
- nie zakładać layoutu struktury `proces`,
- wersjonować struktury wymieniane z BWS.

Dobrym przykładem wersjonowania jest podsystem motywów, który zachowuje ABI v1 i v2.

---

# 7.38 Wersjonowanie `.bur`

Format `.bur` powinien docelowo mieć jawne wersjonowanie.

Przykładowy przyszły model:

```cpp
struct NaglowekBur {
    uint8_t  magia[4];        // BUR\0
    uint16_t wersja_abi;
    uint16_t rozmiar_naglowka;
    ...
};
```

To pozwoli w przyszłości dodać:

- więcej sekcji,
- flagi ochrony,
- relocation metadata,
- wymagania ABI,
- kompatybilność z różnymi wersjami systemu.

Powyższa struktura jest **propozycją kierunku**, a nie deklaracją aktualnego on-disk ABI.

---

# 7.39 Bezpieczeństwo `.bur`

Loader powinien docelowo zapewniać co najmniej:

```text
niekanoniczny entry point → reject
adres kernel space        → reject
overlap sekcji            → reject
offset poza plikiem       → reject
integer overflow          → reject
rozmiar > limit           → reject
nieznana wersja           → reject/fallback według polityki
```

Format aplikacji jest częścią granicy bezpieczeństwa kernela i powinien być fuzzowany tak samo jak parser filesystemu lub sieci.

---

# 7.40 Bezpieczeństwo `opis.aplikacji`

Parser manifestu również przetwarza niezaufane dane.

Powinien mieć ograniczenia:

- maksymalnej długości pliku,
- długości nazwy,
- liczby praw,
- długości pojedynczego identyfikatora,
- liczby pól,
- zakresu PZB,
- duplikatów,
- nieznanych kluczy.

Nieznane uprawnienie nie powinno automatycznie otrzymywać bitu lub być ignorowane w sposób zmieniający bezpieczeństwo.

Preferowany model:

```text
unknown security field
→ fail-closed
```

lub jawna polityka kompatybilności.

---

# 7.41 Kodowanie znaków

Warstwa aplikacyjna Bursztyn OS używa UTF-8.

Dotyczy to między innymi:

- interfejsu użytkownika,
- tekstu aplikacji,
- nazw widocznych użytkownikowi,
- komunikatów.

Aktualny framework i aplikacje systemowe korzystają z czcionek posiadających polskie znaki.

Wewnętrzne nazwy techniczne i ścieżki są zwykle utrzymywane bez znaków diakrytycznych, np.:

```text
/programy
/ustawienia
/tymczasowe
```

---

# 7.42 Obecny stan formatu `.cebula`

Na obecnym etapie można potwierdzić następujący model:

```text
.cebula
= katalogowa paczka aplikacji

zawiera co najmniej:
- program .bur
- opis.aplikacji

może zawierać:
- ikonę
- zasoby
- konfigurację aplikacji
```

Nie należy jeszcze deklarować jako obowiązkowych:

- kryptograficznego katalogu `uprawnienia/`,
- jednego formatu kompresji,
- centralnego indeksu zależności,
- podpisu każdej paczki,
- repozytorium online.

---

# 7.43 Obecny stan formatu `.bur`

Na podstawie aktualnie dostępnej dokumentacji można potwierdzić:

```text
- własny format wykonywalny Bursztyn OS
- x86-64
- Ring 3
- magic BUR
- własny punkt wejścia
- własne informacje potrzebne loaderowi
```

Nie należy bez bezpośredniego audytu aktualnego loadera deklarować jako zamrożonych:

```text
- dokładnego sizeof nagłówka
- dokładnych offsetów wszystkich pól
- dokładnej liczby sekcji
- stałego RING3_BASE jako wiecznego ABI
- dokładnego sposobu relokacji
```

---

# 7.44 Aktualny stan integracji z systemem

Ekosystem aplikacji jest już realnie używany przez Bursztyn OS.

Przykładowe ścieżki:

```text
/programy/notatnik.cebula/notatnik.bur
/programy/instalator.cebula/instalator.bur
```

Aplikacje Ring 3 korzystają z rozwiniętego BWS obejmującego:

```text
GUI
pliki
sieć
eventy
SHM
IPC
clipboard
kursory
sockety
aktualizacje
motywy
```

To oznacza, że `.bur` i `.cebula` nie są już wyłącznie koncepcją roadmapy.

---

# 7.45 Co jest nadal roadmapą

Nadal przyszłe lub niepotwierdzone jako kompletne są między innymi:

- kompilator Avocado,
- automatyczna generacja `.bur` z Avocado,
- automatyczna generacja manifestów przez Avocado,
- pełny package manager `.cebula`,
- dependency resolver,
- publiczne repozytorium aplikacji,
- obowiązkowe podpisy aplikacji,
- sandbox PZB-5 z pełnym per-app namespace,
- stabilna, zamrożona specyfikacja binarna `.bur`.

---

# 7.46 Zasady projektowe ekosystemu

Aktualna dokumentacja powinna utrzymywać następujące reguły:

1. `.bur` jest plikiem wykonywalnym Ring 3.
2. `.cebula` jest katalogową paczką aplikacji, niekoniecznie archiwum.
3. `opis.aplikacji` deklaruje potrzeby, ale nie nadaje samodzielnie praw.
4. Aktualną nazwą bezpieczeństwa jest PZB.
5. BWS jest jedyną kontrolowaną drogą do uprzywilejowanych operacji.
6. BursztynUI pozostaje w Ring 3.
7. Kontrolka GUI nie wymaga osobnego syscalla.
8. Format `.bur` musi być walidowany jak niezaufane wejście.
9. Manifest musi być walidowany fail-closed.
10. Avocado jest obecnie roadmapą, a nie ukończonym toolchainem.
11. Kod C/C++/ASM pozostaje aktualną bazą aplikacji i systemu.
12. Dokumentacja nie powinna deklarować niepotwierdzonego podpisywania wszystkich `.cebula`.

---

# 7.47 Najważniejsze zmiany względem starego dokumentu

Zaktualizowano:

- **BZL → PZB**;
- `.cebula` z „archiwum” na aktualny model katalogowej paczki;
- oddzielono potwierdzony format `.bur` od niezweryfikowanych szczegółów layoutu;
- usunięto twierdzenie, że `loader.bur` jest modułem odpowiedzialnym za ładowanie, jeśli nie jest to potwierdzone aktualnym kodem;
- nie traktuje się już `RING3_BASE = 0x1000000` jako pewnego, wiecznego ABI;
- manifest opisano jako deklarację praw podlegającą polityce jądra;
- dodano współczesne BWS 1–87;
- dodano BursztynUI;
- dodano Event ABI;
- dodano SHM i IPC;
- dodano clipboard MIME;
- dodano file watch;
- dodano socket facade;
- dodano `.btp`;
- dodano podsystem motywów;
- dodano instalator jako aplikację Ring 3;
- rozdzielono podpisy OTA od niepotwierdzonego podpisywania paczek aplikacji;
- integrację Avocado przeniesiono z „wdrożonej funkcji” do **roadmapy**;
- usunięto stwierdzenie, że kompilator Avocado obecnie eliminuje błędy bezpieczeństwa manifestów.

---

# 7.48 Podsumowanie

Aktualny ekosystem Bursztyn OS można przedstawić następująco:

```text
BSP2
└── /programy
    └── aplikacja.cebula
        ├── opis.aplikacji
        ├── aplikacja.bur
        ├── ikona
        └── zasoby
```

Po uruchomieniu:

```text
.cebula
   ↓
manifest + .bur
   ↓
loader
   ↓
PZB + PRAWO_*
   ↓
prywatna przestrzeń procesu
   ↓
Ring 3
   ↓
BursztynUI / SDK
   ↓
BWS 1–87
   ↓
Jądro
```

Najważniejszym obecnym wyróżnikiem ekosystemu nie jest jeszcze Avocado, lecz fakt, że Bursztyn OS posiada już własne:

- aplikacje Ring 3,
- format `.bur`,
- paczki `.cebula`,
- manifesty,
- BWS,
- PZB,
- BursztynUI,
- filesystem BSP2.

Avocado pozostaje docelowym językiem, który ma w przyszłości spiąć te elementy w jeden toolchain developerski.

---

## 7.49 Dokumenty powiązane

Aktualny stan należy interpretować razem z:

- `02_architektura_systemu.md` — PZB i `PRAWO_*`,
- `05_bursztynowy_system_plikow.md` — BSP2 i `/programy`,
- `06_wywolania_systemowe.md` — aktualne BWS 1–87,
- `08_bursztynowy_slownik_i_architektura.md` — nazewnictwo `.bur` i `.cebula`,
- `11_audyt_bws_nowoczesne.md` — SHM, IPC, clipboard, watchery i sockety,
- `aktualizacja-systemu.md` — BWS 84–85 i aplikacja aktualizacji.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny loader `.bur`,
2. aktualny parser `opis.aplikacji`,
3. `bws_api.h`,
4. aktualne SDK Ring 3,
5. aktualny BSP2,
6. nowsze audyty,
7. starsza dokumentacja opisowa.

Przed publikacją formalnej binarnej specyfikacji `.bur` warto wykonać osobny audyt aktualnego kodu loadera i wygenerować tabelę wszystkich pól `NaglowekBur` bezpośrednio ze źródła.
