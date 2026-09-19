# 02. Architektura Systemu i Model Bezpieczeństwa PZB

**Stan dokumentu:** 2026-09-18  
**Status:** aktualny opis architektury Ring 0 / Ring 3, PZB i bezpieczeństwa BWS

Bursztyn OS łączy sprzętową izolację procesora x86-64 z autorskim modelem logicznym **PZB — Poziom Zaufania Bursztyna**. PZB nie zastępuje sprzętowych ringów procesora: jądro i sterowniki niskopoziomowe działają w Ring 0, a programy użytkownika `.bur` działają w Ring 3.

---

## 2.1 Warstwy ochrony

Aktualny model bezpieczeństwa można przedstawić jako kilka współpracujących warstw:

```text
opis.aplikacji / polityka aplikacji
                ↓
             PZB 0–5
                ↓
         64-bitowe PRAWO_*
                ↓
   PID / właściciel / generacja
                ↓
 walidacja argumentów i user-copy
                ↓
         VMM / PTE / Ring 3
                ↓
 GDT / TSS / IDT / kernel stack / IST
                ↓
          x86-64 Ring 0 / Ring 3
```

Żadna pojedyncza warstwa nie jest traktowana jako wystarczająca sama w sobie.

---

## 2.2 Ring 0 — tryb jądra

W Ring 0 wykonywane są między innymi:

- Jądro Bursztyna,
- PMM i VMM,
- scheduler,
- GDT/TSS/IDT,
- APIC i HPET,
- BSP2/PSF,
- implementacja BWS,
- SATA AHCI,
- Intel E1000,
- USB xHCI/HID,
- Intel HDA,
- niskopoziomowe elementy grafiki i kompozycji.

Kod Ring 0 ma dostęp do instrukcji uprzywilejowanych, portów I/O, MMIO, tablic stron i pamięci jądra. Z tego powodu błąd w kodzie Ring 0 może naruszyć cały system.

PZB nie chroni jądra przed błędem znajdującym się już wewnątrz samego Ring 0.

---

## 2.3 Ring 3 — przestrzeń użytkownika

Ring 3 jest środowiskiem dla:

- Menedżera Okien,
- Powłoki Bursztynowej,
- aplikacji systemowych,
- aplikacji `.bur`,
- aplikacji instalowanych z paczek `.cebula`,
- przyszłych usług użytkownika.

Proces Ring 3 nie powinien mieć bezpośredniego dostępu do:

- pamięci jądra,
- ramek PMM,
- tablic stron jądra,
- portów I/O,
- urządzeń MMIO,
- rejestrów kontrolnych CPU.

Próba wykonania uprzywilejowanej instrukcji, np. `out`, `cli` lub `hlt`, powoduje wyjątek procesora. Dostęp do niezmapowanej albo niedozwolonej strony prowadzi do `#PF`.

Operacje uprzywilejowane są wykonywane przez kontrolowane BWS.

---

## 2.4 GDT i TSS

W Long Mode głównym mechanizmem ochrony pamięci jest stronicowanie, ale GDT i TSS nadal są potrzebne do prawidłowej obsługi poziomów uprzywilejowania i wyjątków.

Bursztyn OS posiada deskryptory dla:

- kodu Ring 0,
- danych Ring 0,
- kodu Ring 3,
- danych Ring 3,
- TSS.

Należy odróżnić **16-bajtowy deskryptor TSS w GDT** od właściwej struktury TSS.

TSS przechowuje między innymi:

- `RSP0`,
- stosy IST.

Przy przerwaniu lub wyjątku przechodzącym z Ring 3 do Ring 0 procesor może użyć `RSP0`. Dla wybranych wyjątków może zostać użyty dedykowany stos IST.

Bursztyn OS wykorzystuje awaryjny stos IST dla krytycznego `#DF` — Double Fault.

---

## 2.5 Ważna korekta: SYSCALL nie przełącza RSP przez TSS

Starsza dokumentacja zawierała stwierdzenie, że przy `SYSCALL` procesor automatycznie ładuje stos jądra z TSS. W x86-64 jest to niepoprawne.

Instrukcja `SYSCALL`:

- przechodzi do kodu wskazanego przez `IA32_LSTAR`,
- zmienia poziom uprzywilejowania zgodnie z konfiguracją MSR,
- zapisuje adres powrotu w `RCX`,
- zapisuje `RFLAGS` w `R11`,

ale **nie przełącza automatycznie RSP na `TSS.RSP0`**.

Dlatego wejście BWS musi jawnie przełączyć wykonanie na bezpieczny stos Ring 0 przed wejściem do kodu C/C++.

Schemat:

```text
Ring 3
  │
  │ SYSCALL
  ▼
wejście asemblerowe BWS
  │
  ├─ zachowanie stanu użytkownika
  ├─ wybór stosu jądra bieżącego procesu
  └─ przełączenie RSP
  ▼
bezpieczny stos Ring 0
  │
  ▼
bws_obsluga(...)
```

TSS pozostaje ważny dla przerwań, wyjątków i IST.

---

## 2.6 Stosy Ring 0

Aktualny model stosów jądra zakłada oddzielony stos Ring 0 dla kontekstu procesu.

Mechanizmy ochronne obejmują:

- stos Ring 0 o rozmiarze rzędu 64 KiB,
- wydzielony slot przestrzeni wirtualnej,
- niezmapowaną stronę strażnika,
- kontrolę integralności stosu,
- dedykowany IST dla `#DF`.

Guard page ma spowodować `#PF`, zanim przepełnienie stosu uszkodzi sąsiednią pamięć.

Kanarek stosu jest dodatkowym mechanizmem wykrywania korupcji i nie zastępuje guard page ani poprawnej kontroli długości buforów.

---

## 2.7 PZB — Poziomy Zaufania Bursztyna

PZB jest logiczną polityką jądra. Niższa wartość oznacza wyższy poziom zaufania.

| PZB | Nazwa | Przeznaczenie |
|---:|---|---|
| **0** | Jądro | Kod Ring 0 i najbardziej uprzywilejowane mechanizmy systemu |
| **1** | Sterowniki / warstwa niskopoziomowa | Zaufane komponenty sprzętowe i przyszłe wydzielone usługi sterownikowe |
| **2** | Usługi systemowe | Zaufane usługi platformy i komponenty administracyjne |
| **3** | Aplikacje zaufane | Systemowe aplikacje Ring 3 wymagające szerszego zakresu praw |
| **4** | Aplikacje użytkownika | Zwykłe programy `.bur` i aplikacje z paczek `.cebula` |
| **5** | Piaskownica | Kod najmniej zaufany i najbardziej ograniczony |

PZB nie jest dodatkowym ringiem procesora. Proces PZB-2, PZB-3, PZB-4 czy PZB-5 nadal może działać sprzętowo w Ring 3.

---

## 2.8 Maski `PRAWO_*`

Poziom PZB definiuje klasę zaufania, ale szczegółowe możliwości procesu są określane przez 64-bitową maskę praw.

Podstawowe udokumentowane prawa:

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

Przy polu `uint64_t uprawnienia` należy stosować `1ULL << n`, a nie zwykłe `1 << n`.

Przykładowy Notatnik może potrzebować:

```text
PZB-4
PRAWO_GUI
PRAWO_PLIKI_CZYTAJ
PRAWO_PLIKI_ZAPISZ
```

bez praw:

```text
PRAWO_SYSTEM_CONFIG
PRAWO_STEROWNIK
PRAWO_DEBUG
```

---

## 2.9 Manifest `opis.aplikacji`

Manifest paczki `.cebula` opisuje żądane możliwości aplikacji.

Przykład koncepcyjny:

```text
nazwa = "Notatnik"
poziom_zaufania = 4

uprawnienia = [
    "okna",
    "pliki_czytaj",
    "pliki_zapisz"
]
```

Manifest nie powinien być traktowany jako bezwarunkowe źródło uprawnień.

Prawidłowa ścieżka:

```text
opis.aplikacji
      ↓
parser manifestu
      ↓
walidacja polityki
      ↓
PZB + PRAWO_*
      ↓
struktura procesu w Ring 0
```

Aplikacja nie może samodzielnie podnieść własnego PZB ani dopisać sobie nowych praw.

Proces nie powinien również móc przekazać potomkowi wyższego poziomu zaufania lub praw, których sam nie posiada.

---

## 2.10 Struktura procesu

Dokumentacja nie powinna zamrażać wewnętrznego layoutu struktury procesu jako publicznego ABI.

Logicznie proces przechowuje między innymi:

```cpp
struct Proces {
    uint64_t pid;

    uint8_t poziom_zaufania;   // PZB 0..5
    uint64_t uprawnienia;      // PRAWO_*

    void* przestrzen_adresowa;

    // kontekst wykonania
    // stos Ring 3
    // stos Ring 0
    // kolejka zdarzeń
    // zasoby GUI
    // SHM
    // IPC
    // watchery
    // kursory
    // socket
};
```

Rzeczywista struktura kernela może się zmieniać bez zmiany ABI aplikacji.

---

## 2.11 Aktualny zakres BWS

Starsza dokumentacja mówiła o 26 BWS. Ten stan jest nieaktualny.

Nowszy audyt dokumentuje pełną mapę **BWS 1–83**, a kolejne podsystemy używają również:

```text
BWS 84 — OTA status
BWS 85 — OTA command
BWS 86–87 — podsystem motywów
```

Aktualne drzewo projektu używa więc BWS co najmniej do numeru **87**.

Canonical listę należy utrzymywać w kodzie (`bws_api.h`) i synchronizować z dokumentacją.

---

## 2.12 Walidacja wskaźników Ring 3

Jądro nie może ufać adresowi przekazanemu przez aplikację.

Nowsze BWS wykorzystują kontrolowane helpery kopiujące dane pomiędzy Ring 3 i Ring 0.

Walidacja powinna obejmować:

- kanoniczność adresu,
- odrzucanie niedozwolonych obszarów,
- sprawdzanie mapowania PTE,
- wymagane prawo odczytu lub zapisu,
- sprawdzanie całego zakresu,
- ochronę przed overflow `adres + rozmiar`,
- kopiowanie danych do pamięci należącej do jądra.

Schemat:

```text
pointer Ring 3
      ↓
walidacja zakresu
      ↓
walidacja PTE
      ↓
copy_from_user_checked()
      ↓
bufor Ring 0
      ↓
walidacja semantyczna
      ↓
operacja
```

Jądro nie powinno zachowywać surowego pointera Ring 3 i używać go później po zmianie kontekstu.

---

## 2.13 Bezpieczna arytmetyka rozmiarów

Przed alokacją lub mapowaniem należy sprawdzać operacje takie jak:

```text
count * element_size
width * bytes_per_pixel
stride * height
offset + length
size + PAGE_SIZE - 1
```

Nowsze BWS sprawdzają dodawanie, mnożenie oraz zaokrąglanie liczby stron.

Przykład:

```cpp
size_t stride;
if (!checked_mul(width, 4, &stride))
    return BWS_BLAD_PRZEPELNIENIA;

size_t bytes;
if (!checked_mul(stride, height, &bytes))
    return BWS_BLAD_PRZEPELNIENIA;
```

---

## 2.14 Uchwyty generacyjne

Ring 3 nie powinien otrzymywać surowych wskaźników struktur jądra ani adresów fizycznych.

Nowsze podsystemy BWS używają uchwytów z generacją.

Koncepcyjnie:

```text
handle = slot + generation
```

Po zamknięciu obiektu i ponownym wykorzystaniu slotu numer generacji ulega zmianie.

Stary uchwyt zostaje odrzucony.

Mechanizm chroni przed stale-handle i częścią błędów use-after-close.

Generacyjne uchwyty są używane między innymi przez:

- SHM,
- IPC,
- file watch,
- sockety.

---

## 2.15 SHM — BWS 64–67

Natywna pamięć współdzielona obejmuje:

- create,
- map,
- unmap,
- close.

Aktualne ograniczenia:

- strona: 4 KiB,
- maks. 2 MiB na obiekt,
- maks. 8 MiB globalnie,
- zerowanie ramek przed udostępnieniem,
- uchwyty generacyjne,
- kontrola dostępu per PID,
- jawny transfer prawa dostępu,
- brak ujawniania adresów fizycznych i MMIO.

Przy końcu procesu usuwane są jego mapowania i prawa do SHM.

---

## 2.16 IPC — BWS 68–71

BWS IPC implementują ograniczone endpointy i kolejki.

Aktualne limity:

- do 512 B na wiadomość,
- do 16 wiadomości w kolejce endpointu,
- do 32 endpointów,
- generacyjne uchwyty.

Reguły:

- odbiór i zamknięcie są ograniczone do właściciela,
- zły właściciel otrzymuje odmowę,
- stale handle jest odrzucany,
- pełna kolejka nie może nadpisywać pamięci,
- transfer SHM jest jawny i kontrolowany.

---

## 2.17 File watch — BWS 72

BWS 72 udostępnia obserwowanie zmian plików.

Aktualna implementacja:

- tworzy uchwyty watcherów,
- kontroluje właściciela,
- ponownie sprawdza PZB dla istniejących obiektów,
- korzysta z ograniczonej kolejki zdarzeń.

Mechanizm nadal jest częściowy: nie wszystkie wewnętrzne operacje BSP2 mają jeszcze centralne hooki watcherów.

---

## 2.18 Clipboard MIME — BWS 73–76

Nowszy clipboard obsługuje wiele formatów.

Aktualne limity:

- do 8 formatów,
- do 1 MiB na format,
- do 2 MiB łącznie,
- dane są kopiowane przez jądro.

Kernel nie powinien przechowywać pointera do bufora należącego do procesu Ring 3.

---

## 2.19 Kursory — BWS 77–78

BWS 77 udostępnia kursory systemowe.

BWS 78 obsługuje własny obraz kursora procesu.

Dla kursora własnego:

- maksymalny rozmiar: 64×64,
- hotspot jest walidowany,
- obraz jest kopiowany do pamięci jądra,
- stary obraz jest zwalniany dopiero po poprawnym przygotowaniu nowego.

Obowiązuje zasada:

```text
prepare
→ validate
→ publish
→ release old
```

---

## 2.20 Sockety TCP/IPv4 — BWS 79–83

Socket facade obejmuje:

- create,
- connect,
- send,
- receive,
- close.

Warstwa bezpieczeństwa wykorzystuje:

- `PRAWO_SIEC`,
- właściciela PID,
- generacyjny uchwyt,
- walidację buforów Ring 3,
- limity rozmiaru,
- cleanup po zakończeniu procesu.

Aktualne ograniczenie:

> bazowy stos TCP pozwala nadal na jedną systemową sesję jednocześnie.

API socketów nie oznacza więc jeszcze pełnego modelu współbieżnych socketów.

---

## 2.21 GUI i prawa systemowe

Przykładowo BWS 61 dla geometrii własnego okna:

- wymaga `PRAWO_GUI`,
- działa wyłącznie na warstwie bieżącego PID,
- waliduje wymiary,
- przygotowuje nowy stan przed publikacją.

BWS 62–63 udostępniają informacje o monitorze, obszarze roboczym i skali.

Globalny obszar roboczy może publikować wyłącznie zaufana usługa posiadająca `PRAWO_SYSTEM_CONFIG`.

---

## 2.22 Cleanup procesu

Proces nie może pozostawiać trwałych zasobów Ring 0 po zakończeniu.

Nowszy model cleanup usuwa między innymi:

- warstwy GUI,
- własne kursory,
- mapowania SHM,
- prawa SHM,
- endpointy IPC,
- oczekujące wiadomości IPC,
- file watchery,
- posiadany socket.

Każdy nowy typ uchwytu BWS musi zostać dopisany do centralnej ścieżki cleanup procesu.

---

## 2.23 Ochrona ścieżek BSP2

Samo posiadanie:

```text
PRAWO_PLIKI_ZAPISZ
```

nie powinno pozwalać zwykłej aplikacji modyfikować krytycznych obszarów systemu.

Przykładowe chronione przestrzenie:

```text
/jadro
/system
/sterowniki
```

Decyzja może uwzględniać:

- PZB,
- `PRAWO_*`,
- znormalizowaną ścieżkę,
- właściciela,
- rodzaj operacji.

Jest to **logiczna polityka jądra**, a nie sprzętowa ochrona ścieżki — procesor x86-64 nie zna pojęcia katalogu BSP2.

---

## 2.24 Przykład bezpiecznej ścieżki BWS

Poniższy kod jest przykładem dokumentacyjnym, a nie deklaracją dokładnej sygnatury istniejącej funkcji:

```cpp
BwsWynik bws_zapisz_plik(
    Proces* p,
    const char* sciezka_user,
    const void* dane_user,
    size_t rozmiar)
{
    if (!p)
        return BWS_BLAD;

    if (!(p->uprawnienia & PRAWO_PLIKI_ZAPISZ))
        return BWS_BRAK_UPRAWNIEN;

    if (rozmiar > BWS_MAX_ZAPIS)
        return BWS_BLAD_LIMITU;

    char sciezka[KERNEL_MAX_PATH];

    if (!copy_string_from_user_checked(
            p, sciezka_user, sciezka, sizeof(sciezka)))
        return BWS_ZLY_WSKAZNIK;

    if (!normalizuj_sciezke(sciezka))
        return BWS_ZLA_SCIEZKA;

    if (!pzb_moze_zapisac(p, sciezka))
        return BWS_BRAK_UPRAWNIEN;

    return bsp2_zapisz_z_usercopy(
        p, sciezka, dane_user, rozmiar);
}
```

Kluczowa zasada:

> aplikacja przekazuje żądanie, ale ostateczną decyzję i dostęp do zasobu realizuje jądro.

---

## 2.25 Zasada najmniejszych uprawnień

Proces powinien otrzymać tylko prawa potrzebne do działania.

Kalkulator może wymagać jedynie:

```text
PZB-4
PRAWO_GUI
```

Przeglądarka może potrzebować:

```text
PZB-4
PRAWO_GUI
PRAWO_SIEC
```

Prawo zapisu plików powinno być oddzielną możliwością:

```text
PRAWO_PLIKI_ZAPISZ
```

Brak prawa jest stanem domyślnie bezpieczniejszym niż automatyczne nadawanie szerokiej maski.

---

## 2.26 Brak eskalacji

Proces Ring 3 nie powinien móc:

- podnieść własnego PZB,
- dopisać sobie `PRAWO_*`,
- utworzyć procesu o większych prawach bez autoryzacji jądra,
- przejąć uchwytu innego PID,
- użyć starego uchwytu po ponownym wykorzystaniu slotu,
- mapować fizycznej pamięci przez publiczne BWS.

Każda operacja zmiany zaufania musi przechodzić przez zaufaną politykę systemową.

---

## 2.27 Fail-closed

Bursztyn OS preferuje odmowę operacji przy niepewnym stanie.

Przykłady:

- nieznany BWS → błąd,
- zły pointer → odmowa,
- brak `PRAWO_*` → odmowa,
- zły owner → odmowa,
- stale handle → odmowa,
- overflow → odmowa,
- za duży bufor → odmowa,
- niepoprawny podpis aktualizacji → brak aktywacji,
- niepoprawny układ dysku → brak instalacji.

Fail-closed nie oznacza jednak automatycznie braku błędów. Ścieżki odmowy i rollbacku również muszą być testowane.

---

## 2.28 Aktualizacje A/B a bezpieczeństwo

Podsystem aktualizacji korzysta z:

```text
BWS 84 — status OTA
BWS 85 — polecenie OTA
```

Model obejmuje:

- slot A/B,
- SHA-256,
- RSA-2048,
- generacyjny boot state,
- CRC32,
- próbny rozruch,
- health checkpoint,
- rollback.

Aplikacja Ring 3 „Aktualizacje Systemu” nie powinna posiadać bezpośredniego dostępu do surowych struktur stanu rozruchu ani dowolnych sektorów dysku.

Operacje krytyczne są wykonywane po stronie zaufanego kodu systemowego.

---

## 2.29 BWS 86–87

Aktualny kod projektu używa również BWS 86–87 dla podsystemu motywów.

Najnowsze testy projektu wykazywały jeszcze regresję w tej grupie, dlatego dokumentacja nie powinna określać jej jako produkcyjnie stabilnej do czasu nowego pełnego przebiegu testowego bez błędów.

Jednocześnie numeracja ta potwierdza, że stare określenie:

```text
„BWS zawiera 26 wywołań”
```

jest nieaktualne.

---

## 2.30 Ograniczenia modelu bezpieczeństwa

Bursztyn OS posiada realną separację Ring 0 / Ring 3, PZB, maski praw i coraz bardziej rozbudowany model uchwytów, ale pozostaje systemem eksperymentalnym.

Nie należy pisać, że gwarantuje:

> „absolutną stabilność i ochronę przed złośliwym oprogramowaniem”.

Aktualne ograniczenia obejmują m.in.:

- brak Secure Boot,
- brak TPM-backed root of trust,
- brak pełnego szyfrowania danych użytkownika,
- ograniczony sandbox PZB-5,
- brak szerokiego fuzzingu wszystkich parserów i BWS,
- część testów wieloprocesowych i destrukcyjnych nadal wymaga rozbudowy,
- sterowniki Ring 0 powiększają Trusted Computing Base,
- system pozostaje jednordzeniowy i nie posiada jeszcze pełnej złożoności synchronizacji SMP.

Rzetelne sformułowanie:

> **PZB, maski `PRAWO_*`, izolacja Ring 3, checked user-copy, kontrola właściciela oraz generacyjne uchwyty tworzą warstwowy model bezpieczeństwa Bursztyn OS, który jest nadal rozwijany i testowany.**

---

## 2.31 Minimalne testy bezpieczeństwa nowego BWS

Każde nowe BWS powinno posiadać co najmniej testy:

1. poprawnego użycia,
2. NULL,
3. błędnego wskaźnika,
4. zakresu przechodzącego przez niezmapowaną stronę,
5. braku wymaganego prawa,
6. złego właściciela,
7. stale handle,
8. minimalnego rozmiaru,
9. maksymalnego rozmiaru,
10. `MAX + 1`,
11. integer overflow,
12. cleanup po zakończeniu procesu.

Dla obiektów z uchwytami:

```text
create
use
close
reuse slot
old handle must fail
```

Dla zasobów procesu:

```text
baseline
create resources
terminate process
resource usage must return to baseline
```

---

## 2.32 Powiązane dokumenty

- `03_proces_rozruchu.md` — Long Mode i wczesne stronicowanie,
- `04_zarzadzanie_sprzetem.md` — GDT, TSS, IDT, APIC i wyjątki,
- `05_bursztynowy_system_plikow.md` — BSP2/PSF,
- `06_wywolania_systemowe.md` — starszy opis BWS; część numeracji jest historyczna,
- `07_ekosystem_i_formaty.md` — `.bur`, `.cebula`, manifesty,
- `11_audyt_bws_nowoczesne.md` — BWS 61–83, limity i testy,
- `aktualizacja-systemu.md` — BWS 84–85 i bezpieczeństwo A/B.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny kod,
2. `bws_api.h` i publiczne struktury ABI,
3. nowsze audyty,
4. aktualne testy runtime,
5. starsza dokumentacja opisowa.

---

## 2.33 Najważniejsze zmiany względem starego dokumentu

Zaktualizowano:

- liczbę i zakres BWS — nie 26, lecz co najmniej do 87,
- opis TSS i `SYSCALL`,
- model stosów Ring 0,
- PZB i maski 64-bitowe,
- checked user-copy,
- sprawdzanie overflow,
- generacyjne uchwyty,
- własność zasobów per PID,
- SHM,
- IPC,
- file watch,
- clipboard MIME,
- kursory,
- socket facade,
- cleanup procesu,
- OTA BWS 84–85,
- status BWS 86–87,
- zasady fail-closed.

Usunięto niepoprawne twierdzenia:

- że `SYSCALL` automatycznie przełącza RSP przez TSS,
- że ochrona ścieżek BSP2 jest „sprzętowa”,
- że Bursztyn OS gwarantuje absolutną ochronę przed złośliwym oprogramowaniem.
