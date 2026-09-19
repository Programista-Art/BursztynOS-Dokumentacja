# Bursztyn OS vs macOS 27 Golden Gate — rzetelne porównanie techniczne

**Data opracowania:** 2026-09-18  
**Stan Bursztyn OS:** na podstawie aktualnych plików projektu, nowszych audytów oraz ostatnich wyników testów przekazanych przez autora projektu  
**Stan macOS:** macOS 27 Golden Gate, wydany 14 września 2026 r.  
**Cel dokumentu:** porównanie architektury, możliwości i dojrzałości technicznej. Dokument nie jest rankingiem „który system jest lepszy”.

> **Ważne:** Bursztyn OS jest rozwijanym od zera, eksperymentalnym systemem operacyjnym. macOS jest dojrzałym, komercyjnym systemem desktopowym rozwijanym od dziesięcioleci i ściśle zintegrowanym ze sprzętem Apple. Porównanie ma sens jako zestawienie rozwiązań architektonicznych, ale nie jako porównanie dwóch produktów o podobnej skali, dojrzałości lub bazie użytkowników.

---

## 1. Podsumowanie

| Obszar | Bursztyn OS — stan projektu 2026 | macOS 27 Golden Gate |
|---|---|---|
| **Typ systemu** | Autorski system bare-metal; własne jądro, BWS, BSP2/PSF, PZB i BursztynUI | Komercyjny system Apple oparty na Darwin/XNU |
| **Architektura jądra** | Monolityczne jądro x86-64; sterowniki niskopoziomowe głównie w Ring 0; aplikacje i Menedżer Okien w Ring 3 | Hybrydowe XNU: Mach + komponenty BSD + IOKit; wiele współczesnych sterowników może działać w user space przez DriverKit |
| **Architektura CPU** | x86-64 / Long Mode | Współczesne komputery Mac używają Apple silicon; XNU jako projekt źródłowy zawiera obsługę ARM64 i x86_64 |
| **SMP** | Obecnie jednordzeniowy | Pełne środowisko wielordzeniowe; Apple silicon używa rdzeni wydajnościowych i energooszczędnych |
| **Planista** | Round-Robin; LAPIC kalibrowany HPET, 250 Hz | Wielordzeniowy scheduler XNU; priorytety i QoS, integracja z GCD i współczesną współbieżnością |
| **Pamięć wirtualna** | 4-poziomowe stronicowanie x86-64, PMM/VMM, Ring 0/Ring 3, SHM; brak swapu i kompresji pamięci | Mach VM, kompresja pamięci, swap, rozbudowane zarządzanie presją pamięci |
| **ABI / wywołania systemowe** | Własne BWS przez `SYSCALL`; aktualne testy projektu używają numerów co najmniej do 87, ale dokumentacja ABI wymaga konsolidacji | Mach traps, BSD/POSIX syscalls, Mach IPC i rozbudowane API systemowe |
| **Uprawnienia** | PZB 0–5 + maski `PRAWO_*`; egzekwowanie w jądrze | POSIX DAC/ACL + MAC, App Sandbox, SIP, TCC, entitlements i inne mechanizmy platformowe |
| **System plików** | BSP2/PSF: bloki 4 KiB, wskaźniki bezpośrednie i pośrednie; obecny instalator ogranicza partycję BSP2 do 2 GiB | APFS: copy-on-write, snapshots, clones, space sharing, natywne szyfrowanie |
| **Nośniki** | Obecnie instalacja na SATA AHCI, sektor 512 B/512e; brak działającego backendu NVMe, IDE i USB mass storage | Pełna integracja z pamięcią flash/NVMe w obsługiwanych komputerach Mac |
| **Aktualizacje** | Własny model A/B, rollback, RSA-2048/SHA-256, rekordy stanu z CRC/generacją | Signed System Volume, APFS snapshots, secure boot i mechanizmy aktualizacji Apple |
| **Grafika** | Programowy kompozytor CPU nad framebufferem; BursztynUI; brak akceleracji GPU | Quartz/Core Animation/Metal i sprzętowo akcelerowana kompozycja |
| **GUI** | BursztynUI: ok. 62 publiczne elementy frameworka według aktualnego inwentarza projektu | AppKit, SwiftUI, Core Animation, Core Graphics i duży ekosystem frameworków |
| **Sieć** | E1000, IPv4, ARP, ICMP, UDP, DHCP, DNS, TCP, HTTP/HTTPS/TLS; BWS 79–83 daje socket facade, ale stos nadal ma limit jednej systemowej sesji TCP | Dojrzały stos IPv4/IPv6, BSD sockets, Network framework, TCP/UDP/QUIC, HTTP/2 i HTTP/3 |
| **Audio** | HDA w ograniczonym modelu; brak pełnego publicznego API streamów i miksowania | Core Audio, AVAudioEngine i AudioDriverKit |
| **Sterowniki** | Własne sterowniki m.in. AHCI, E1000, xHCI/HID, HDA; głównie Ring 0 | IOKit oraz DriverKit/System Extensions; wiele klas nowych sterowników może działać w user space |
| **Bezpieczeństwo rozruchu** | Własna walidacja A/B i podpisów aktualizacji; instalator wymaga obecnie wyłączonego Secure Boot | Hardware-rooted secure boot na Apple silicon, SSV, podpisy kodu, Secure Enclave |
| **Dojrzałość sprzętowa** | Głównie QEMU/Bochs i ograniczony zestaw urządzeń; część scenariuszy prawdziwego sprzętu nadal niepotwierdzona | Produkcyjny system dla obsługiwanej rodziny komputerów Mac |
| **Ekosystem aplikacji** | Własny `.bur`, `.cebula`, BursztynUI, rozwijana integracja z Avocado | AppKit/SwiftUI/Metal/POSIX, Xcode, Swift, Objective-C, C/C++ i ogromny ekosystem aplikacji |

---

## 2. Architektura jądra

### Bursztyn OS

Bursztyn OS jest własnym systemem bare-metal dla x86-64. Architektura rozdziela:

- **Ring 0** — jądro i sterowniki niskopoziomowe,
- **Ring 3** — aplikacje `.bur`, Menedżer Okien i inne programy użytkownika.

Jądro ma charakter **monolityczny**: sterowniki takie jak AHCI czy E1000 pracują w tej samej uprzywilejowanej przestrzeni co reszta jądra. Nie należy określać Bursztyna jako mikrojądra ani „mikro-monolitu”.

Projekt ma własne:

- PMM/VMM,
- GDT/TSS,
- IDT/APIC,
- BWS,
- PZB,
- BSP2/PSF,
- podsystem GUI i kompozytor.

Nowsze audyty projektu stwierdzają wprost, że system nadal jest **jednordzeniowy**. HPET służy do kalibracji LAPIC, a okresowy clockevent planisty jest ustawiany na 250 Hz.

### macOS 27

XNU jest oficjalnie opisywany przez Apple jako **jądro hybrydowe**, łączące:

- Mach,
- komponenty BSD,
- C++ API IOKit.

Współczesny macOS korzysta z wielu usług działających poza jądrem. Apple rozwija DriverKit i System Extensions, dzięki którym wiele klas sterowników może działać w przestrzeni użytkownika zamiast jako rozszerzenia jądra.

### Wniosek

Bursztyn OS ma znacznie prostszą architekturę i mniejszy zakres odpowiedzialności. Ułatwia to naukę i analizę całości systemu, ale oznacza też mniejszą izolację sterowników oraz większy wpływ błędu Ring 0 na cały system.

macOS ma architekturę znacznie bardziej złożoną, ale oferuje wieloletnio rozwijane mechanizmy izolacji, współbieżności, kompatybilności i zarządzania zasobami.

---

## 3. Procesor, wielozadaniowość i planowanie

### Bursztyn OS

Aktualny stan projektu:

- x86-64 Long Mode,
- APIC zamiast legacy PIC,
- scheduler Round-Robin,
- LAPIC kalibrowany przez HPET,
- 250 Hz dla okresowego clockeventu,
- PID 0 wykonuje oczekującą pracę kompozytora poza IRQ,
- projekt pozostaje jednordzeniowy.

System nie wykorzystuje jeszcze równolegle wielu rdzeni współczesnego CPU.

### macOS 27

Współczesne komputery Mac używają układów Apple silicon. Są to platformy heterogeniczne, z rdzeniami nastawionymi na wydajność oraz efektywność energetyczną.

macOS stosuje QoS do określania ważności pracy. Scheduler może kierować zadania na odpowiedni typ rdzenia w zależności od obciążenia i priorytetu. GCD i mechanizmy współbieżności pozwalają aplikacjom korzystać z wielu rdzeni.

### Wniosek

Największa różnica jest wyraźna: **Bursztyn OS jest obecnie UP, a macOS jest dojrzałym systemem wielordzeniowym**. Implementacja SMP jest jednym z najważniejszych przyszłych kroków dla Bursztyna.

---

## 4. Pamięć wirtualna

### Bursztyn OS

Projekt ma własny PMM i VMM oraz stronicowanie x86-64. Nowszy audyt BWS dokumentuje m.in.:

- SHM create/map/unmap/close,
- strony 4 KiB,
- generacyjne uchwyty,
- kontrolę dostępu per PID,
- limity rozmiarów,
- walidowane kopiowanie danych Ring 3 ↔ Ring 0.

Projekt nie ma obecnie:

- swapu,
- kompresji stron,
- kompletnego mechanizmu page-out/page-in porównywalnego z Mach VM.

Przy braku pamięci kolejne alokacje muszą zostać odrzucone lub obsłużone przez politykę błędu systemu.

### macOS 27

XNU posiada rozbudowany Mach VM. Aktualne źródła XNU opisują m.in.:

- kompresję pamięci,
- zarządzanie presją pamięci,
- backing store,
- swapfiles,
- odzyskiwanie pamięci.

### Wniosek

Bursztyn ma już prawdziwą izolację przestrzeni adresowych i SHM, ale **zarządzanie presją pamięci jest nadal dużo prostsze** niż w macOS.

---

## 5. BWS a interfejs systemowy macOS

### Bursztyn OS

Programy Ring 3 komunikują się z jądrem przez autorski interfejs **BWS** z użyciem instrukcji `SYSCALL`.

Dokumentacja projektu jest obecnie rozproszona:

- starszy `06_wywolania_systemowe.md` nadal opisuje 26 BWS i jest nieaktualny,
- audyt z 2026-08-30 dokumentuje BWS 1–83,
- dokument aktualizacji dodaje BWS 84–85,
- najnowsza aplikacja testowa wykonuje testy podsystemu motywów BWS 86–87.

Dlatego najbezpieczniejsze sformułowanie dla stanu z 2026-09-18 brzmi:

> **Aktualne drzewo testowe Bursztyn OS używa BWS co najmniej do numeru 87, ale skonsolidowana dokumentacja publicznego ABI wymaga aktualizacji.**

Nowsze BWS obejmują m.in.:

- geometrię okien,
- monitor/work area/UI scale,
- SHM,
- IPC,
- file watch,
- clipboard MIME,
- kursory,
- socket facade TCP/IPv4,
- OTA,
- motywy.

### macOS 27

XNU łączy:

- Mach traps,
- IPC przez Mach ports,
- warstwę BSD/POSIX i klasyczne syscalle,
- liczne API bibliotek i frameworków user space.

### Wniosek

BWS jest zwartym ABI rozwijanym specjalnie pod potrzeby Bursztyn OS. macOS oferuje nieporównanie większy zestaw kontraktów ABI/API, odziedziczony z Darwin, POSIX i wielu frameworków Apple.

---

## 6. Model bezpieczeństwa

### Bursztyn OS

Projekt stosuje dwa poziomy ochrony:

1. **sprzętowy Ring 0 / Ring 3**,  
2. **logiczny PZB 0–5 oraz maski `PRAWO_*`**.

Nowszy audyt BWS dokumentuje:

- kontrolowane kopiowanie struktur Ring 3,
- walidację rozmiarów i zaokrągleń,
- brak zwracania adresów fizycznych,
- uchwyty z generacją,
- kontrolę właściciela dla IPC/socketów,
- cleanup zasobów po śmierci procesu.

To jest realny model bezpieczeństwa, ale nie powinien być opisywany jako „absolutna ochrona” lub „gwarancja bezpieczeństwa”. System nadal jest rozwijany i nie przeszedł skali audytów, fuzzingu i badań bezpieczeństwa porównywalnej z systemem komercyjnym.

### macOS 27

macOS używa wielu warstw bezpieczeństwa, m.in.:

- tradycyjnych mechanizmów UNIX/POSIX,
- Mandatory Access Controls,
- App Sandbox,
- SIP,
- TCC i zgód prywatności,
- Gatekeeper/notarization/XProtect,
- podpisów kodu,
- zabezpieczeń sprzętowych Apple silicon,
- Secure Enclave,
- hardware-rooted secure boot.

### Wniosek

PZB jest interesującą i czytelną koncepcją projektową, ale macOS ma znacznie szerszy, wielowarstwowy model bezpieczeństwa obejmujący sprzęt, boot chain, integralność systemu, aplikacje i prywatność użytkownika.

---

## 7. System plików i pamięć masowa

### Bursztyn OS

Najnowszy audyt instalatora określa BSP2/PSF jako jeden system plików:

- bloki 4096 B,
- 64-bitowe adresy bloków,
- wskaźniki bezpośrednie,
- pojedyncze bloki pośrednie,
- podwójne bloki pośrednie.

W aktualnym wydaniu instalator:

- tworzy ESP FAT32 512 MiB,
- tworzy BSP2 do maksymalnie 2 GiB,
- używa GPT,
- wymaga SATA AHCI,
- obsługuje sektor logiczny 512 B / 512e,
- odrzuca 4Kn,
- nie ma działającego backendu blokowego NVMe,
- nie ma sterownika IDE,
- nie ma USB mass storage jako nośnika docelowego.

To istotna różnica względem starszych plików projektu opisujących wcześniejsze warianty BSP. W porównaniach należy bazować na nowszym audycie instalatora.

### macOS 27

macOS używa APFS. Udokumentowane cechy APFS obejmują:

- copy-on-write,
- snapshots,
- clones,
- space sharing,
- native encryption,
- crash protection,
- atomic safe-save.

Nie należy przypisywać APFS funkcji „wear leveling” jako cechy systemu plików — zarządzanie zużyciem komórek flash jest przede wszystkim zadaniem urządzenia i warstwy FTL.

### Wniosek

BSP2 jest rzeczywistym autorskim systemem plików na etapie rozwoju, ale APFS reprezentuje znacznie dojrzalszy model przeznaczony dla współczesnych pamięci flash, z snapshotami, CoW i zaawansowanym szyfrowaniem.

---

## 8. Rozruch i aktualizacje

### Bursztyn OS

Projekt implementuje własny model aktualizacji A/B:

- Slot A i Slot B,
- nieaktywny slot jako cel aktualizacji,
- health checkpoint,
- rollback po nieudanym rozruchu,
- RSA-2048 i SHA-256 dla pakietów aktualizacji,
- dwa rekordy stanu rozruchu,
- licznik generacji,
- CRC32,
- synchronizację z `grubenv`.

Obecna implementacja przechowuje autorytatywne rekordy stanu na LBA 2046 i 2047, bezpośrednio przed ESP zaczynającą się od LBA 2048. Dokumentacja projektu sama wskazuje, że przed stabilnym wydaniem publicznym warto rozważyć przeniesienie tych danych do dedykowanej małej partycji GPT.

To mechanizm realny i testowany w QEMU, ale nie powinien być opisywany jako odpowiednik poziomu ochrony Secure Boot/SSV Apple.

### macOS 27

Apple stosuje m.in.:

- secure boot,
- Signed System Volume,
- podpisy kodu,
- APFS snapshots,
- łańcuch zaufania zakorzeniony w sprzęcie Apple silicon.

SSV używa kryptograficznego drzewa skrótów SHA-256, a na Apple silicon bootloader weryfikuje seal przed przekazaniem sterowania do kernela.

### Wniosek

Bursztynowy model A/B jest jednym z bardziej zaawansowanych elementów projektu i sensownie rozwiązuje rollback aktualizacji. macOS idzie dalej, ponieważ integralność systemu jest częścią szerszego sprzętowo-programowego łańcucha zaufania.

---

## 9. Grafika, kompozytor i interfejs

### Bursztyn OS

Bursztyn renderuje GUI programowo na CPU do 32-bitowego framebufferu.

Projekt posiada:

- Menedżer Okien w Ring 3,
- własny kompozytor,
- własne powierzchnie aplikacji i popupów,
- BursztynUI,
- skalowanie UI 100%, 125%, 150%, 175%, 200%,
- backend Bochs VBE pozwalający w QEMU przełączać tryby m.in. 1280×720, 1920×1080, 2560×1440 i 3840×2160, jeśli dostępny VRAM jest wystarczający.

Według aktualnego inwentarza projektu BursztynUI ma około **62 publicznych elementów frameworka**, licząc kontrolki, dialogi oraz elementy infrastruktury takie jak `PowierzchniaPopup` i `ZarzadcaPopupow`.

Trzeba zachować ostrożność przy opisywaniu optymalizacji kompozytora. Starszy audyt schedulera mówi o jednym globalnym `dirty` i pełnym składaniu powierzchni, natomiast nowsze prace nad popupami wprowadzają dokładniejsze damage regions. Dopóki dokumentacja kompozytora nie zostanie ujednolicona, nie należy twierdzić, że cały system ma już dojrzały silnik Dirty Rects porównywalny z kompozytorem desktopowym macOS.

### macOS 27

macOS wykorzystuje dojrzały system grafiki oparty m.in. o:

- Quartz / Core Graphics,
- Core Animation,
- Metal,
- sprzętową akcelerację GPU,
- CAMetalLayer,
- synchronizację renderowania z odświeżaniem wyświetlacza.

Core Animation przekazuje znaczną część pracy kompozycyjnej do GPU i innych wyspecjalizowanych mechanizmów sprzętowych.

### Wniosek

BursztynUI jest istotnym osiągnięciem projektu, ponieważ nie korzysta z Qt, GTK ani WinAPI. Jednocześnie brak sterownika GPU i sprzętowej akceleracji jest jednym z największych ograniczeń w porównaniu z macOS.

---

## 10. Zmiana rozdzielczości i wysokie DPI

### Bursztyn OS

Audyt `Ustawienia ekranu` dokumentuje:

- runtime mode switch dla Bochs VBE,
- dynamiczną listę trybów,
- 15-sekundowy rollback,
- zapis `/ustawienia/ekran.cfg`,
- work area przez BWS62,
- skalowanie UI 100–200%,
- osobną pulę PMM/VMM dla dużych buforów grafiki.

W bieżącym rozwoju pojawił się jednak realny problem popupów po zmianie z 1280×720 na 1920×1080. Ostatnia poprawka powiązała stan popupu z **generacją framebufferu** i odrzuca powierzchnie należące do starej generacji. Poprawka nadal wymaga końcowej ręcznej akceptacji wielokrotnego otwierania `DialogPliku` po zatwierdzeniu trybu 1920×1080.

Dlatego nie należy obecnie pisać, że obsługa wysokich rozdzielczości jest „w pełni stabilna” we wszystkich scenariuszach GUI.

### macOS 27

macOS od wielu lat obsługuje wysokie DPI/Retina, wielomonitorowość, sprzętową kompozycję oraz szeroką gamę konfiguracji ekranów obsługiwanych przez sprzęt Apple.

---

## 11. Sieć

### Bursztyn OS

Aktualnie udokumentowane elementy obejmują:

- E1000,
- Ethernet,
- ARP cache,
- IPv4,
- ICMP,
- UDP,
- DHCP,
- DNS,
- routing przez bramę,
- TCP,
- HTTP,
- HTTPS/TLS,
- BWS 79–83 jako owned TCP/IPv4 socket facade.

Kluczowe ograniczenie z nowszego audytu:

> mimo socket facade 79–83 warstwa TCP nadal pozwala tylko na jedną systemową sesję jednocześnie i nie posiada pełnego modelu współbieżnych socketów ani eventu readiness.

W dostarczonej dokumentacji nie ma pełnego stosu IPv6.

### macOS 27

Apple udostępnia m.in.:

- Network framework,
- BSD sockets,
- TCP,
- UDP,
- QUIC,
- TLS,
- URLSession,
- HTTP/2 i HTTP/3,
- Bonjour.

### Wniosek

Bursztyn posiada już własny działający stos sieciowy, co jest dużym krokiem dla systemu budowanego od zera. Nadal jest to jednak model eksperymentalny o znacznie mniejszej współbieżności i liczbie protokołów niż macOS.

---

## 12. Audio

### Bursztyn OS

Audyt BWS dokumentuje działającą warstwę HDA, ale nadal w ograniczonym modelu:

- prywatny tor wyjściowy,
- polling,
- stałe 48 kHz,
- S16,
- stereo,
- BWS 27 jako test tone,
- brak pełnego publicznego API strumieni,
- brak negocjacji formatów i kompletnego miksera.

### macOS 27

macOS oferuje Core Audio, AVAudioEngine i AudioDriverKit z obsługą strumieni, urządzeń, sterowników user-space, efektów, miksowania i zaawansowanych scenariuszy audio.

---

## 13. Sterowniki i sprzęt

### Bursztyn OS

Projekt ma własne sterowniki m.in.:

- AHCI SATA,
- Intel E1000,
- xHCI,
- HID keyboard/mouse,
- HDA,
- APIC/HPET,
- framebuffer HAL.

Zakres sprzętowy jest jednak wąski. Audyt instalatora jawnie wskazuje brak obsługi jako dysku instalacyjnego:

- NVMe,
- IDE,
- USB mass storage,
- 4Kn.

Secure Boot dla instalacji musi być obecnie wyłączony.

### macOS 27

macOS działa na kontrolowanym zbiorze platform Apple, ale w jego obrębie integracja sprzętowo-programowa jest bardzo głęboka. DriverKit pozwala implementować wiele klas sterowników w user space, m.in. USB, HID, networking, PCI, serial i audio.

### Wniosek

Bursztyn daje programiście pełną kontrolę nad własnym HAL-em, ale kompatybilność sprzętowa jest obecnie jednym z największych ograniczeń praktycznych.

---

## 14. Typografia

### Bursztyn OS

BursztynUI korzysta z własnej ścieżki UTF-8 i wbudowanej czcionki Unicode. Projekt kładzie szczególny nacisk na poprawne wyświetlanie języka polskiego.

Jest to rozwiązanie o znacznie mniejszym zakresie niż współczesny silnik typograficzny komercyjnego systemu desktopowego.

### macOS 27

Apple używa m.in. Core Text i Core Graphics do układu i renderowania tekstu. Core Text zapewnia m.in.:

- mapowanie znak → glif,
- kerning,
- ligatury,
- fallback fontów,
- metryki fontów,
- zaawansowany layout Unicode.

Nie należy opisywać FreeType jako systemowego silnika typografii macOS.

---

## 15. Bezpieczeństwo danych na dysku

### Bursztyn OS

Bursztyn ma:

- integralność aktualizacji przez SHA-256/RSA,
- CRC i generacje rekordów boot state,
- model PZB,
- kontrolę dostępu do plików przez BWS.

W dostarczonej dokumentacji nie ma odpowiednika pełnego szyfrowania danych użytkownika na poziomie FileVault/APFS z hardware-rooted key management.

### macOS 27

FileVault używa szyfrowania dysku, a na komputerach z Apple silicon zarządzanie kluczami jest sprzętowo powiązane z Secure Enclave. APFS wspiera również własne mechanizmy szyfrowania.

### Wniosek

To jeden z obszarów, w których macOS ma znacznie bardziej dojrzały i sprzętowo zakotwiczony model ochrony danych.

---

## 16. Ekosystem aplikacji

### Bursztyn OS

Bursztyn ma własny model:

- pliki wykonywalne `.bur`,
- paczki `.cebula`,
- manifesty uprawnień,
- Ring 3,
- BursztynUI,
- rozwijaną integrację z językiem Avocado.

Brak kompletnego środowiska zgodności POSIX/libc oznacza, że popularnego oprogramowania nie można po prostu przekompilować bez dodatkowej warstwy portującej.

### macOS 27

macOS posiada wieloletni ekosystem:

- POSIX/Darwin,
- AppKit,
- SwiftUI,
- Metal,
- Core Foundation,
- Foundation,
- Xcode,
- Swift,
- Objective-C,
- C/C++,
- tysiące bibliotek i aplikacji firm trzecich.

### Wniosek

Bursztyn stawia na własny ABI i tożsamość techniczną; macOS korzysta z ogromnego, dojrzałego ekosystemu kompatybilności i narzędzi.

---

## 17. Rzeczywiste atuty projektowe Bursztyn OS

Nie są to przewagi „nad macOS” w sensie produktu końcowego, lecz cechy wartościowe dla projektu.

### 17.1. Pełna kontrola nad stosem

Autor projektu kontroluje:

- boot,
- kernel,
- pamięć,
- sterowniki,
- ABI,
- system plików,
- GUI,
- formaty aplikacji,
- model uprawnień.

To ma dużą wartość edukacyjną i badawczą.

### 17.2. Mały zakres kodu

W porównaniu z macOS kod projektu jest nieporównanie mniejszy, dzięki czemu pojedyncza osoba może rozumieć większy procent całości.

Nie oznacza to automatycznie większego bezpieczeństwa ani „pełnej audytowalności”, ale ułatwia naukę i analizę.

### 17.3. Własne pomysły architektoniczne

Szczególnie charakterystyczne są:

- PZB,
- `.bur`,
- `.cebula`,
- BWS,
- A/B update flow,
- BursztynUI,
- polskie nazewnictwo systemowe.

### 17.4. Bounded design

W nowszych audytach widać świadome stosowanie limitów:

- bounded IPC queues,
- bounded SHM,
- generation handles,
- kontrolowane rozmiary struktur i buforów.

To dobry kierunek dla bezpiecznego kodu systemowego.

---

## 18. Główne ograniczenia Bursztyn OS względem macOS

Najważniejsze ograniczenia obecnego Bursztyna to:

1. brak SMP,
2. brak akceleracji GPU,
3. brak swapu i kompresji pamięci,
4. ograniczona liczba sterowników sprzętowych,
5. brak NVMe jako backendu blokowego,
6. ograniczony stos TCP i brak pełnej współbieżności socketów,
7. ograniczone audio API,
8. brak pełnego szyfrowania danych porównywalnego z FileVault,
9. brak Secure Boot w obecnym instalatorze,
10. znacznie mniejszy ekosystem aplikacji,
11. brak długoletniego testowania na szerokiej bazie sprzętowej,
12. część GUI nadal przechodzi aktywną stabilizację, zwłaszcza popupy i scenariusze zmiany rozdzielczości.

---

## 19. Aktualny status jakości i testów Bursztyn OS

Projekt ma rozbudowane testy hostowe i QEMU. Poszczególne audyty raportują tysiące pomyślnych asercji, a np. audyt instalatora raportuje pełne PASS dla swojej konkretnej suity.

Jednocześnie najnowszy przekazany log aplikacji **Testy Bursztyn OS** zawierał nadal obowiązkowe FAIL-e w obszarach:

- BWS 86/87 — motywy,
- BursztynUI popup,
- UTF-8,
- toolbar/tooltip allocations,
- SelektorKoloru,
- picker lifecycle.

Trwają prace nad naprawą tych regresji.

Dlatego aktualnego stanu projektu nie należy opisywać jako:

- „w pełni stabilny”,
- „bezbłędny”,
- „produkcyjnie bezpieczny”,
- „zweryfikowany na każdym sprzęcie”.

Rzetelne określenie:

> **Bursztyn OS jest zaawansowanym eksperymentalnym systemem operacyjnym z działającym środowiskiem Ring 3, GUI, systemem plików, siecią, instalatorem i aktualizacjami A/B, ale nadal znajduje się w aktywnej fazie stabilizacji i rozszerzania zgodności sprzętowej.**

---

## 20. Ocena końcowa

Bursztyn OS i macOS 27 nie są systemami na tym samym etapie rozwoju.

**Bursztyn OS** jest wartościowy jako:

- autorska implementacja systemu od zera,
- platforma edukacyjna i badawcza,
- środowisko do eksperymentów z ABI, GUI, schedulerem, bezpieczeństwem i systemem plików,
- projekt pokazujący działanie komputera od bootloadera po aplikację Ring 3.

**macOS 27** jest:

- wielordzeniowym systemem produkcyjnym,
- głęboko zintegrowanym ze sprzętem Apple,
- wyposażonym w zaawansowaną pamięć wirtualną, GPU, audio, networking i bezpieczeństwo,
- wspieranym przez ogromny ekosystem narzędzi i aplikacji.

Najbardziej uczciwy wniosek brzmi:

> Bursztyn OS nie jest dziś funkcjonalnym odpowiednikiem macOS i nie należy przedstawiać go jako konkurenta o tej samej dojrzałości. Jest natomiast rzeczywistym, własnym systemem operacyjnym o coraz szerszym zakresie funkcji, którego wartość leży przede wszystkim w niezależnej architekturze, pełnej kontroli nad kodem i stopniowym budowaniu kolejnych warstw od bare-metal do własnego środowiska graficznego.

---

# 21. Źródła

## 21.1. Bursztyn OS — pliki projektu

Najważniejsze dokumenty wykorzystane przy opracowaniu:

- `01_wstep_i_filozofia.md`
- `02_architektura_systemu.md`
- `03_proces_rozruchu.md`
- `04_zarzadzanie_sprzetem.md`
- `04_audyt_gui_schedulera.md`
- `05_hpet_i_pacing.md`
- `06_wywolania_systemowe.md` — **częściowo nieaktualny względem nowszych audytów BWS**
- `09_tryb_graficzny.md`
- `10_siec.md`
- `11_audyt_bws_nowoczesne.md`
- `instalator_audyt.md`
- `ustawienia_ekranu_audyt.md`
- `aktualizacja-systemu.md`
- ostatni przekazany wynik aplikacji `Testy Bursztyn OS`

Przy rozbieżnościach priorytet nadano nowszym audytom i aktualnym wynikom testów.

## 21.2. Oficjalne źródła Apple / XNU

1. Apple — macOS 27 Golden Gate  
   https://www.apple.com/pl/os/macos/

2. Apple Support — Apple security releases  
   https://support.apple.com/pl-pl/100100

3. Apple Open Source — XNU  
   https://github.com/apple-oss-distributions/xnu

4. Apple Developer — DriverKit  
   https://developer.apple.com/documentation/driverkit

5. Apple Developer — System Extensions  
   https://developer.apple.com/documentation/systemextensions

6. Apple Developer — Core Animation  
   https://developer.apple.com/documentation/quartzcore

7. Apple Developer — Core Text  
   https://developer.apple.com/documentation/CoreText

8. Apple Developer — Core Graphics  
   https://developer.apple.com/documentation/coregraphics

9. Apple Developer — Apple File System Guide / APFS  
   https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/Features/Features.html

10. Apple Platform Security — Signed System Volume  
    https://support.apple.com/en-ie/guide/security/secd698747c9/web

11. Apple Platform Security — System Integrity Protection  
    https://support.apple.com/en-ie/guide/security/secb7ea06b49/web

12. Apple Platform Deployment — FileVault  
    https://support.apple.com/pl-pl/guide/deployment/dep82064ec40/web

13. Apple Developer — Network framework  
    https://developer.apple.com/documentation/network

14. Apple Developer — TN3151: Choosing the right networking API  
    https://developer.apple.com/documentation/technotes/tn3151-choosing-the-right-networking-api

15. Apple Developer — Core Audio  
    https://developer.apple.com/documentation/CoreAudio

16. Apple Developer — AudioDriverKit  
    https://developer.apple.com/documentation/audiodriverkit

17. Apple XNU — VM / memory pressure documentation  
    https://github.com/apple-oss-distributions/xnu/blob/main/doc/vm/memorystatus_notify.md

18. Apple Developer — QoS  
    https://developer.apple.com/documentation/dispatch/dispatchqos

---

## 22. Zalecenie dla dokumentacji Bursztyn OS

Warto utworzyć jeden nadrzędny plik:

`docs/STAN_SYSTEMU_2026-09-18.md`

zawierający wyłącznie aktualne:

- maksymalny numer BWS,
- liczbę procesów i włókien,
- model SMP/UP,
- limity pamięci,
- stan BSP2,
- listę sterowników,
- listę protokołów sieciowych,
- listę backendów grafiki,
- status Secure Boot,
- liczbę elementów BursztynUI,
- wynik ostatniej pełnej suity testowej,
- znane regresje,
- status testów QEMU i prawdziwego sprzętu.

Taki dokument powinien być traktowany jako źródło nadrzędne dla przyszłych porównań, opisów i materiałów promocyjnych.
