# Bursztyn OS vs Windows 11 — rzetelne porównanie techniczne

**Data opracowania:** 2026-09-18  
**Stan Bursztyn OS:** na podstawie aktualnych plików projektu, nowszych audytów oraz ostatnich wyników testów przekazanych przez autora projektu  
**Punkt odniesienia po stronie Windows:** przede wszystkim **Windows 11 25H2** jako szeroko wdrażana wersja; dodatkowo uwzględniono **Windows 11 26H1**, który jest specjalizowanym wydaniem dla wybranych nowych platform sprzętowych  
**Cel:** porównanie architektury, funkcji i dojrzałości technicznej bez marketingowego zawyżania możliwości którejkolwiek strony.

> **Ważne:** Bursztyn OS jest rozwijanym od zera, eksperymentalnym systemem operacyjnym. Windows 11 jest dojrzałym, komercyjnym systemem desktopowym rozwijanym przez Microsoft i partnerów sprzętowych przez dziesięciolecia. Porównanie ma sens jako zestawienie rozwiązań architektonicznych i kierunków rozwoju, ale nie jako zestawienie dwóch produktów o porównywalnej skali, kompatybilności sprzętowej lub ekosystemie.

---

# 1. Aktualny punkt odniesienia Windows 11

Na dzień 2026-09-18 Microsoft utrzymuje kilka wspieranych wydań Windows 11.

Najważniejsze:

- **Windows 11 25H2** — ogólnodostępne wydanie przeznaczone do szerokiego wdrożenia,
- **Windows 11 26H1** — wydanie sprzętowo zoptymalizowane dla wybranych nowych platform i urządzeń, niebędące zwykłą aktualizacją funkcjonalną dla istniejących komputerów.

Microsoft wprost zaznacza, że 26H1:
- nie jest standardowym corocznym feature update,
- nie jest oferowane jako zwykły upgrade przez Windows Update na istniejących urządzeniach,
- jest przeznaczone dla wybranych nowych platform sprzętowych,
- nie zastępuje 25H2 jako zalecanej wersji do szerokiego wdrożenia.

Dlatego w tym dokumencie **Windows 11 25H2** jest głównym punktem odniesienia dla typowego współczesnego Windows 11.

Na 14 września 2026 r. Microsoft udostępnił dla 25H2 aktualizację KB5129195, podnoszącą build do **26200.9457**.

---

# 2. Zestawienie skrócone

| Obszar | Bursztyn OS — aktualny stan projektu | Windows 11 |
|---|---|---|
| **Typ systemu** | Własny system bare-metal rozwijany od zera | Komercyjny system Microsoft z rodziny Windows NT |
| **Jądro** | Monolityczne x86-64, własny kernel Ring 0 | Architektura Windows NT: kernel + Executive + HAL + sterowniki kernel-mode/user-mode |
| **CPU** | x86-64 / Long Mode | x64 i Arm64; Windows 11 on Arm uruchamia również wiele aplikacji x86/x64 przez emulację |
| **SMP** | Obecnie jednordzeniowy | Pełne wielordzeniowe i wieloprocesorowe środowisko |
| **Scheduler** | Round-Robin, LAPIC kalibrowany HPET, 250 Hz | Preemptive priority scheduler; wiele poziomów priorytetu, wielordzeniowość, processor groups |
| **Pamięć** | Własne PMM/VMM, Ring 0/Ring 3, SHM; brak swapu | Zaawansowany VM, pagefile, memory-mapped files, copy-on-write, paged/nonpaged pools |
| **Syscalle / ABI** | Własne BWS przez `SYSCALL`; aktualne testy projektu używają co najmniej numerów do 87 | Aplikacje używają głównie dokumentowanych API Win32/Windows App SDK; raw NT syscalls są szczegółem implementacyjnym |
| **Bezpieczeństwo** | Ring 0/Ring 3, PZB 0–5, maski `PRAWO_*`, generacyjne uchwyty | ACL, access tokens, integrity levels, AppContainer, Smart App Control, VBS, HVCI, Credential Guard, Secure Boot, TPM 2.0 |
| **System plików** | BSP2/PSF, bloki 4 KiB, własny format | NTFS jako domyślny filesystem Windows; ReFS w wybranych scenariuszach |
| **Nośniki** | Aktualnie SATA AHCI; brak NVMe w pierwszym wydaniu instalatora | NVMe, SATA, USB storage i szeroki Windows storage stack |
| **Aktualizacje** | Własny A/B, rollback, RSA-2048/SHA-256, redundantny boot state | Windows Update, cumulative updates, servicing stack, checkpoint cumulative updates |
| **Grafika** | Programowy kompozytor CPU, framebuffer, BursztynUI | WDDM + DirectX + DWM + GPU hardware acceleration |
| **GUI** | Własny BursztynUI, około 62 publiczne elementy frameworka według bieżącego inwentarza projektu | Win32, WinUI 3, Windows App SDK, WPF, UWP, DirectComposition, Direct2D/DirectWrite |
| **Sieć** | E1000, IPv4, ARP, ICMP, UDP, DHCP, DNS, TCP, HTTP/HTTPS/TLS; ograniczona współbieżność TCP | IPv4/IPv6, Winsock, TCP/UDP/ICMP, rozbudowany stack sieciowy |
| **Audio** | HDA w ograniczonym modelu | Windows Audio Engine, WASAPI, XAudio2, MIDI i szeroki model sterowników audio |
| **Sterowniki** | Własny wąski zestaw: AHCI, E1000, xHCI/HID, HDA, grafika | WDM/WDF, KMDF, UMDF i ogromny ekosystem sterowników producentów |
| **Wirtualizacja** | Brak odpowiednika Hyper-V | Hyper-V na obsługiwanych edycjach, VBS, Windows Sandbox w obsługiwanych konfiguracjach |
| **Szyfrowanie dysku** | Brak odpowiednika BitLocker dla pełnego woluminu użytkownika | BitLocker + TPM |
| **Secure Boot** | Obecny instalator wymaga wyłączenia Secure Boot | Windows 11 wymaga platformy Secure Boot capable i TPM 2.0 |
| **Aplikacje** | `.bur`, `.cebula`, BursztynUI, rozwijane Avocado | Win32, .NET, WinUI, WPF, UWP, MSIX, Microsoft Store, tradycyjne EXE/MSI |
| **Dojrzałość** | Zaawansowany projekt eksperymentalny w aktywnej stabilizacji | Produkcyjny desktopowy system dla ogromnej bazy sprzętu i aplikacji |

---

# 3. Architektura jądra

## Bursztyn OS

Bursztyn OS jest własnym systemem bare-metal dla x86-64.

Architektura rozdziela:
- **Ring 0** — kernel i niskopoziomowe sterowniki,
- **Ring 3** — programy `.bur`, Menedżer Okien i pozostałe aplikacje użytkownika.

Jądro ma charakter **monolityczny**.

W Ring 0 działają obecnie między innymi:
- PMM/VMM,
- scheduler,
- APIC/HPET,
- BWS,
- BSP2/PSF,
- AHCI,
- E1000,
- xHCI/HID,
- HDA,
- niskopoziomowe elementy grafiki i kompozycji.

Bursztyn OS nie wykorzystuje jądra Linux, Windows NT ani żadnego gotowego kernela.

## Windows 11

Windows 11 wywodzi się z architektury Windows NT.

Microsoft dokumentuje warstwę **Executive**, której komponenty zapewniają m.in.:
- Object Management,
- Memory Management,
- Process and Thread Management,
- I/O Management,
- Configuration Management.

Komponenty Executive są częścią `ntoskrnl.exe`.

Poza nimi istnieją m.in.:
- kernel,
- HAL,
- kernel-mode drivers,
- user-mode services,
- subsystem grafiki,
- frameworki sterowników.

Windows nie jest prostym monolitem w znaczeniu „wszystko działa w jednym module”. Jest systemem warstwowym, ale wiele kluczowych sterowników nadal może działać w kernel mode.

## Wniosek

Bursztyn OS jest obecnie znacznie prostszy. To ułatwia śledzenie wykonania, naukę i pełną kontrolę nad własnym kodem.

Windows oferuje natomiast znacznie szerszą izolację funkcjonalną, wiele frameworków sterowników i dekady kompatybilności.

---

# 4. Architektury CPU

## Bursztyn OS

Aktualny system obsługuje:
- x86-64,
- Long Mode.

Projekt nie ma obecnie portów Arm64, RISC-V ani innych architektur CPU.

## Windows 11

Windows 11 działa współcześnie na:
- **x64**,
- **Arm64**.

Windows on Arm może wykonywać:
- aplikacje natywne Arm64,
- wiele aplikacji x86,
- wiele aplikacji x64 przez emulację.

Microsoft zaznacza jednak, że sterowniki kernel-mode muszą być skompilowane natywnie dla architektury systemu.

## Wniosek

Bursztyn jest obecnie systemem jednoarchitekturowym. Windows 11 ma dojrzałe środowisko x64 oraz Arm64 z mechanizmami kompatybilności aplikacji.

---

# 5. SMP i wykorzystanie wielu rdzeni

## Bursztyn OS

Aktualne audyty projektu mówią wprost, że projekt pozostaje **jednordzeniowy**.

Scheduler pracuje obecnie na jednym logicznym CPU.

## Windows 11

Windows 11 jest systemem w pełni wielordzeniowym.

Microsoft dokumentuje:
- scheduling wielowątkowy,
- priorytety threadów,
- processor affinity,
- processor groups.

Od Windows 11 procesy mogą domyślnie korzystać z procesorów obejmujących wiele processor groups również na maszynach z więcej niż 64 logicznymi CPU.

## Wniosek

To jedna z największych różnic architektonicznych.

**Bursztyn OS: UP**  
**Windows 11: SMP / duża skalowalność CPU**

---

# 6. Scheduler

## Bursztyn OS

Aktualna architektura:
- Round-Robin,
- LAPIC,
- HPET jako źródło czasu i kalibracji,
- periodic timer 250 Hz,
- compositor nie renderuje bezpośrednio w IRQ,
- PID 0 wykonuje pending compositor work poza obsługą przerwania.

Audyt GUI wskazuje też ograniczenie: długotrwały synchroniczny kod sieci/TLS wykonywany w Ring 0 może okresowo pogarszać responsywność GUI.

## Windows 11

Windows scheduler działa na poziomie threadów.

Microsoft dokumentuje:
- poziomy priorytetów 0–31,
- preemptive scheduling,
- time slices,
- round-robin pomiędzy gotowymi threadami o tym samym najwyższym priorytecie,
- wywłaszczenie niższego priorytetu przez wyższy.

## Wniosek

Scheduler Bursztyna jest celowo mały i przewidywalny. Windows musi obsługiwać duże systemy SMP, multimedia, sterowniki, usługi i wiele klas priorytetu.

---

# 7. Pamięć wirtualna

## Bursztyn OS

Bursztyn posiada własny:
- PMM,
- VMM,
- 4-poziomowy paging x86-64,
- Ring 0/Ring 3,
- prywatne przestrzenie procesów,
- SHM.

Nowszy audyt BWS dokumentuje dla SHM:
- strony 4 KiB,
- generacyjne handles,
- kontrolę dostępu per PID,
- zeroed frames,
- brak ekspozycji fizycznego adresu do Ring 3,
- 2 MiB na obiekt,
- 8 MiB globalnie.

Bursztyn nie ma obecnie:
- swapu,
- pagefile,
- memory compression,
- rozwiniętego systemu reclaim porównywalnego z Windows.

## Windows 11

Windows Memory Manager zapewnia:
- prywatne virtual address spaces procesów,
- memory-mapped files,
- shared memory,
- copy-on-write,
- paging do pliku wymiany,
- paged pool,
- nonpaged pool.

## Wniosek

Bursztyn ma prawdziwe fundamenty VM i izolacji procesów. Windows oferuje jednak znacznie bardziej rozwinięty model zarządzania pamięcią pod presją.

---

# 8. Ring 0 / Ring 3 i separacja user/kernel

## Bursztyn OS

Bursztyn stosuje:
- Ring 0 dla jądra,
- Ring 3 dla aplikacji,
- własne `copy_from_user` / `copy_to_user`,
- sprawdzanie wskaźników użytkownika,
- sprawdzanie PTE,
- PZB i praw.

## Windows 11

Windows również rozdziela:
- user mode,
- kernel mode.

Każdy proces user-mode ma własną przestrzeń adresową. Kernel-mode ma dostęp do przestrzeni systemowej i — w odpowiednim kontekście — do user address space.

## Wniosek

Zasada izolacji sprzętowej jest podobna na bardzo wysokim poziomie, ale Windows ma dużo bardziej rozwinięty system obiektów, tokenów, ACL i I/O Managera.

---

# 9. BWS kontra Windows API

## Bursztyn OS

Programy Ring 3 używają własnego ABI:

**BWS — Bursztynowe Wywołania Systemowe**

Mechanizm bazuje na instrukcji `SYSCALL`.

Aktualna dokumentacja jest częściowo rozproszona:
- starszy dokument opisuje 26 wywołań,
- nowszy audyt opisuje BWS 61–83,
- system aktualizacji używa późniejszych numerów,
- najnowsza aplikacja testowa sprawdza BWS 86–87.

Najbezpieczniejszy aktualny opis:

> **Aktualne drzewo projektu używa BWS co najmniej do numeru 87, ale centralna dokumentacja ABI wymaga synchronizacji z kodem.**

## Windows 11

Typowa aplikacja Windows nie odwołuje się do numeru raw syscalla jako publicznego kontraktu programistycznego.

Aplikacje korzystają głównie z:
- Win32,
- COM,
- Windows App SDK,
- WinRT,
- .NET,
- DirectX,
- Winsock,
- bibliotek systemowych.

## Wniosek

BWS jest małe i bardzo bezpośrednie. Windows ma znacznie większą liczbę warstw API oraz szeroką kompatybilność historyczną.

---

# 10. PZB kontra model bezpieczeństwa Windows

## Bursztyn OS

Bursztyn posiada własny model:

**Poziomy Zaufania Bursztyna 0–5**

oraz maski `PRAWO_*`.

Nowsze BWS dodatkowo używają:
- właściciela zasobu per PID,
- generacji uchwytów,
- bounded queues,
- stale-handle rejection.

## Windows 11

Windows używa wielu mechanizmów równolegle:
- user accounts,
- security access tokens,
- ACL,
- integrity levels,
- privileges,
- AppContainer,
- code integrity,
- Smart App Control,
- App Control for Business,
- VBS,
- HVCI / Memory Integrity,
- Credential Guard,
- TPM,
- Secure Boot.

## Wniosek

PZB jest ciekawym i jednolitym modelem projektowym. Windows ma dużo bardziej warstwowy i wieloletnio rozwijany system bezpieczeństwa.

---

# 11. VBS i HVCI

Windows 11 ma **Virtualization-Based Security**.

Microsoft opisuje VBS jako środowisko izolowane przez hypervisor, które może chronić krytyczne funkcje bezpieczeństwa nawet przy założeniu, że normalne jądro systemu może zostać zaatakowane.

**Memory Integrity / HVCI**:
- uruchamia kernel-mode code integrity w środowisku VBS,
- ogranicza możliwość modyfikowania kodu kernela,
- wzmacnia politykę wykonywalnej pamięci.

Bursztyn OS nie ma obecnie odpowiednika:
- VBS,
- isolated secure kernel environment,
- hypervisor-enforced code integrity.

---

# 12. TPM 2.0 i Secure Boot

## Windows 11

Minimalne wymagania Windows 11 obejmują m.in.:
- kompatybilny 64-bit CPU / SoC,
- minimum 2 rdzenie,
- 4 GB RAM,
- 64 GB storage,
- UEFI,
- Secure Boot capable,
- TPM 2.0,
- DirectX 12 compatible GPU,
- WDDM 2.0.

## Bursztyn OS

Aktualny instalator Bursztyn OS jawnie wymaga:

> **Secure Boot musi być wyłączony.**

`BOOTX64.EFI` nie posiada obecnie łańcucha podpisów wymaganego przez typowe wdrożenie Secure Boot.

Projekt nie wykorzystuje też TPM 2.0 jako obowiązkowego systemowego root of trust.

---

# 13. Szyfrowanie danych

## Bursztyn OS

Projekt ma kryptografię w obszarze:
- podpisów aktualizacji,
- SHA-256,
- RSA-2048,
- TLS.

Nie ma jednak w aktualnej dokumentacji odpowiednika pełnego systemowego szyfrowania woluminu użytkownika.

## Windows 11

Windows posiada **BitLocker**.

BitLocker:
- szyfruje całe woluminy,
- może wykorzystywać TPM,
- chroni dane offline po kradzieży lub wyjęciu dysku.

---

# 14. System plików

## Bursztyn OS

Aktualny audyt instalatora opisuje BSP2/PSF jako filesystem z:
- blokami 4096 B,
- 64-bitowym adresowaniem bloków,
- wskaźnikami bezpośrednimi,
- pojedynczymi blokami pośrednimi,
- podwójnymi blokami pośrednimi.

Aktualny instalator:
- tworzy ESP FAT32 512 MiB,
- tworzy BSP2 do maks. 2 GiB w tym wydaniu,
- zostawia pozostałą przestrzeń nieprzydzieloną.

## Windows 11

**NTFS** jest domyślnym filesystemem współczesnego Windows.

Microsoft dokumentuje m.in.:
- ACL,
- security descriptors,
- encryption support,
- disk quotas,
- metadata,
- duże woluminy,
- change journal.

---

# 15. Storage i sterowniki dysków

## Bursztyn OS

Aktualny stan instalatora:
- GPT,
- UEFI,
- SATA AHCI,
- ATA LBA48,
- 512 B / 512e,
- brak 4Kn,
- brak NVMe,
- brak IDE,
- brak USB mass storage jako docelowego nośnika instalacji.

## Windows 11

Windows obsługuje szeroki zestaw technologii storage, w tym współczesne urządzenia NVMe, SATA i USB storage.

## Wniosek

Najbardziej naturalnym kolejnym dużym krokiem storage dla Bursztyna jest **NVMe**.

---

# 16. Model sterowników

## Bursztyn OS

Sterowniki są głównie własnym kodem kernel-mode.

Projekt ma m.in.:
- AHCI,
- E1000,
- xHCI,
- USB HID keyboard/mouse,
- HDA,
- framebuffer/HAL,
- APIC,
- HPET.

## Windows 11

Windows posiada rozbudowany Windows Driver Model oraz Windows Driver Frameworks:
- **KMDF**,
- **UMDF**,
- WDM,
- wyspecjalizowane miniport models.

UMDF pozwala część sterowników wykonywać poza kernelem, choć nie wszystkie funkcje sprzętowe są dostępne w user mode.

---

# 17. Plug and Play

## Bursztyn OS

Aktualny projekt wykrywa i inicjalizuje wybrane urządzenia podczas startu.

Nie ma jeszcze kompletnego modelu Plug and Play porównywalnego z Windows:
- dynamic install/remove,
- ogromnej bazy INF,
- device classes,
- automatycznego wyboru sterowników,
- pełnej koordynacji power-state.

## Windows 11

Plug and Play jest fundamentalną częścią Windows Driver Model i współpracuje z instalacją sterowników, enumeracją sprzętu oraz power management.

---

# 18. Grafika Bursztyn OS

Bursztyn ma własną ścieżkę graficzną:
- framebuffer,
- backbuffer,
- compositor,
- warstwy aplikacji,
- popup surfaces,
- Menedżer Okien Ring 3,
- BursztynUI.

Renderowanie jest obecnie głównie **programowe na CPU**.

Brak:
- sprzętowego sterownika GPU,
- odpowiednika WDDM,
- GPU scheduler,
- video memory manager,
- Direct3D/Vulkan/OpenGL.

---

# 19. Grafika Windows 11

Windows używa **WDDM — Windows Display Driver Model**.

WDDM ma komponenty user-mode i kernel-mode.

Microsoft dokumentuje m.in.:
- Direct3D runtime,
- user-mode graphics driver,
- kernel-mode display driver,
- `dxgkrnl`,
- video memory manager,
- GPU/NPU scheduler,
- display modes,
- GPU virtualization,
- power management.

Desktop jest składany przez **Desktop Window Manager**.

DWM composition engine działa w osobnym procesie systemowym i kompozytuje drzewa wizualne wielu aplikacji.

---

# 20. BursztynUI kontra Windows GUI

## BursztynUI

Według bieżącego inwentarza projektu BursztynUI ma około **62 publicznych elementów frameworka**, jeśli liczyć kontrolki, dialogi, layout containers, system poleceń i popup infrastructure.

Przykłady:
- Przycisk,
- PoleTekstowe,
- ListaRozwijana,
- Tabela,
- Drzewo,
- Karty,
- Siatka,
- PanelDokowany,
- Wykres,
- ListaWirtualna,
- TabelaWirtualna,
- OknoDialogowe,
- DialogPliku,
- DialogKoloru,
- DialogKomunikatu,
- Powiadomienie,
- PolecenieUI,
- ZarzadcaPopupow.

## Windows 11

Windows nie ma jednego publicznego frameworka „WindowsUI”.

Ekosystem obejmuje m.in.:
- klasyczne Win32,
- WinUI 3,
- Windows App SDK,
- WPF,
- UWP,
- Direct2D,
- DirectWrite,
- DirectComposition,
- DirectX,
- .NET.

---

# 21. Popupy i wysokie rozdzielczości

## Bursztyn OS

Projekt aktywnie stabilizuje:
- dialogi,
- popupy,
- toast notifications,
- zmianę rozdzielczości,
- generacje framebufferu.

W ostatnim etapie stan popupu został powiązany z `generacja_ekranu`.

Stare popupy nie powinny być publikowane do framebufferu nowej generacji.

Ta poprawka została zbudowana, ale końcowa ręczna akceptacja wielokrotnego `DialogPliku` po przejściu na 1920×1080 pozostawała otwarta w ostatnim raporcie.

## Windows 11

WDDM/DWM są wieloletnio rozwijanym produkcyjnym stackiem obsługującym:
- hardware composition,
- wiele ekranów,
- wysokie DPI,
- GPU memory management,
- occlusion,
- recovery sterownika GPU.

---

# 22. Rozdzielczość i skalowanie

## Bursztyn OS

Audyt ekranu dokumentuje m.in.:
- 800×600,
- 1024×768,
- 1280×720,
- 1280×800,
- 1360×768,
- 1600×900,
- 1920×1080,
- 2560×1440,
- 3840×2160

dla testowego backendu Bochs VBE przy odpowiednim VRAM.

Skale BursztynUI:
- 100%,
- 125%,
- 150%,
- 175%,
- 200%.

## Windows 11

Windows posiada wieloletni model:
- DPI awareness,
- per-monitor scaling,
- wiele ekranów,
- sprzętowe display pipelines.

---

# 23. Sieć

## Bursztyn OS

Aktualny stack obejmuje:
- E1000,
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

BWS 79–83 dostarcza owned TCP/IPv4 socket facade:
- create,
- connect,
- send,
- receive,
- close.

Aktualny audyt zaznacza jednak ważne ograniczenie:

> underlying TCP stack pozwala nadal tylko na jedną systemową sesję TCP jednocześnie.

Nie ma jeszcze:
- pełnej współbieżności socketów,
- kompletnego readiness event model,
- pełnego IPv6.

## Windows 11

Winsock obsługuje natywnie:
- IPv4,
- IPv6,
- TCP,
- UDP,
- ICMP i inne protokoły IP.

---

# 24. Audio

## Bursztyn OS

Aktualny audyt BWS opisuje HDA jako nadal ograniczone:
- fixed 48 kHz,
- S16,
- stereo,
- polling,
- BWS 27 test tone,
- brak kompletnego publicznego stream API,
- brak negocjacji formatów,
- brak pełnego miksera.

## Windows 11

Windows Audio Architecture obejmuje m.in.:
- WASAPI,
- XAudio2,
- MIDI,
- Audio Engine,
- `audiodg.exe`,
- `audioeng.dll`.

Audio Engine miksuje i przetwarza streamy.

---

# 25. System plików i ACL

## Bursztyn OS

Dostęp do plików jest kontrolowany przede wszystkim przez:
- BWS,
- PZB,
- `PRAWO_PLIKI_*`.

## Windows 11

NTFS oferuje rozbudowane security descriptors i ACL przypisywane do plików i katalogów.

---

# 26. Aktualizacje Bursztyn OS

Bursztyn ma własny system A/B.

Kluczowe elementy:
- Slot A,
- Slot B,
- confirmed slot,
- pending slot,
- próbny boot,
- health checkpoint,
- automatyczny rollback,
- SHA-256,
- RSA-2048,
- generacyjny boot state,
- CRC32,
- synchronizacja z `grubenv`.

Autorytatywne rekordy są przechowywane w:
- LBA 2046,
- LBA 2047.

Projekt dokumentuje również przyszłą możliwość przeniesienia tego stanu do dedykowanej partycji GPT.

---

# 27. Aktualizacje Windows 11

Windows używa innego modelu.

Microsoft stosuje:
- Windows Update,
- monthly cumulative updates,
- servicing stack,
- checkpoint cumulative updates,
- feature updates,
- out-of-band updates w wyjątkowych przypadkach.

Od 2021 servicing stack update jest włączany w cumulative update.

Od Windows 11 24H2 Microsoft może używać checkpoint cumulative updates.

---

# 28. Czy A/B Bursztyna jest „lepsze” od Windows Update?

Nie ma podstaw, aby tak twierdzić.

A/B ma zalety:
- prosty rollback,
- łatwiejsza atomowość całego obrazu systemu,
- dwa jawne sloty.

Windows servicing ma inne cele:
- aktualizowanie ogromnej liczby komponentów,
- kompatybilność aplikacji,
- migracje ustawień,
- sterowniki,
- edycje enterprise,
- naprawy komponentów.

To dwa różne modele rozwiązujące problemy o innej skali.

---

# 29. Pakiety aplikacji

## Bursztyn OS

Projekt ma:
- `.bur` — własny plik wykonywalny Ring 3,
- `.cebula` — paczka aplikacji,
- manifest/uprawnienia,
- BursztynUI,
- rozwijaną integrację z Avocado.

## Windows 11

Windows obsługuje wiele modeli:
- klasyczne EXE,
- MSI,
- unpackaged Win32,
- MSIX,
- Store packages,
- WinUI 3,
- .NET apps.

Microsoft opisuje MSIX jako współczesny format pakowania aplikacji Windows z package identity oraz kontrolowanym install/uninstall.

---

# 30. Kompatybilność aplikacji

## Bursztyn OS

Brak zgodności z Win32/POSIX oznacza, że programy Windows i Linux nie uruchamiają się bezpośrednio.

To świadoma cena własnego ABI.

## Windows 11

Windows utrzymuje ogromny ekosystem kompatybilności:
- Win32,
- x64,
- x86,
- Arm64,
- emulację x86/x64 na Windows on Arm,
- .NET,
- COM,
- ogromną bazę starszego oprogramowania.

---

# 31. Wirtualizacja

## Bursztyn OS

Bursztyn jest obecnie głównie **gościem** w QEMU podczas rozwoju.

Nie ma własnego odpowiednika Hyper-V.

## Windows 11

Na obsługiwanych edycjach i konfiguracjach dostępny jest Hyper-V.

Microsoft opisuje Hyper-V jako architekturę hypervisor-based z:
- root partition,
- child partitions,
- VMBus,
- synthetic devices,
- hardware-assisted virtualization.

VBS także wykorzystuje Windows hypervisor.

---

# 32. Windows Sandbox

Windows ma również Windows Sandbox w obsługiwanych wydaniach.

Sandbox używa technologii wirtualizacyjnych do uruchamiania izolowanego środowiska Windows.

Bursztyn nie posiada aktualnie bezpośredniego odpowiednika izolowanej VM/sandbox o tej skali.

PZB 5 jest bardziej modelem bezpieczeństwa procesu niż pełną maszyną wirtualną.

---

# 33. Zarządzanie energią

## Bursztyn OS

Projekt wykorzystuje ACPI między innymi do:
- odczytu tabel,
- HPET,
- shutdown ACPI S5.

Pełny laptopowy power management pozostaje ograniczony.

## Windows 11

Windows ma rozbudowany system power management.

Microsoft dokumentuje m.in.:
- Modern Standby,
- device power management,
- ACPI cooperation,
- network power management,
- low-power idle.

---

# 34. Obsługa sprzętu

## Bursztyn OS

Aktualnie projekt koncentruje się na ograniczonym zestawie:
- QEMU/Bochs,
- AHCI,
- E1000,
- xHCI,
- USB HID,
- HDA,
- wybrane framebuffer paths.

## Windows 11

Windows wspiera ogromny ekosystem OEM:
- laptopy,
- desktopy,
- x64,
- Arm64,
- wiele rodzin GPU,
- Wi-Fi,
- Bluetooth,
- NVMe,
- USB,
- kamery,
- urządzenia biometryczne i inne klasy sprzętu.

---

# 35. Testowanie

## Bursztyn OS

Projekt posiada:
- host tests,
- QEMU tests,
- aplikację `Testy Bursztyn OS`,
- testy BWS,
- testy GUI,
- testy pamięci,
- testy instalatora,
- testy aktualizacji,
- testy ekranu.

Przykładowo:
- audyt instalatora raportował 4491/4491 PASS dla swojej suity,
- audyt ustawień ekranu raportował ponad 31 tys. asercji PASS dla konkretnego zakresu.

Nie oznacza to jednak automatycznie globalnego „0 błędów”.

Ostatni przekazany pełny log aplikacji testowej miał nadal FAIL-e m.in. w:
- BWS 86/87,
- popupach BursztynUI,
- UTF-8,
- SelektorKoloru,
- picker lifecycle,
- wybranych pomiarach alokacji.

## Windows 11

Windows jest testowany przez:
- Microsoft,
- producentów sprzętu,
- Windows Hardware Lab Kit,
- Windows Insider Program,
- ogromną bazę produkcyjnych instalacji.

Skala testów nie jest porównywalna.

---

# 36. Minimalne wymagania

## Bursztyn OS

Nie należy jeszcze publikować uniwersalnych minimalnych wymagań bez formalnej matrycy kompatybilności.

Obecny instalator wymaga co najmniej:
- x86-64,
- UEFI dla opisywanej ścieżki instalacji,
- wyłączonego Secure Boot,
- SATA AHCI,
- kompatybilnego framebuffer path,
- obsługiwanego input stack.

## Windows 11

Microsoft podaje m.in.:
- 1 GHz lub szybszy zgodny 64-bit CPU/SoC,
- 2 lub więcej rdzeni,
- 4 GB RAM,
- 64 GB storage,
- UEFI,
- Secure Boot capable,
- TPM 2.0,
- GPU DirectX 12 / WDDM 2.0,
- ekran 720p.

---

# 37. Rzeczywiste zalety projektowe Bursztyn OS

Nie należy przedstawiać ich jako przewagi produktu nad Windows, lecz jako wartości projektu.

## 37.1. Pełna kontrola nad stosem

Autor kontroluje:
- boot,
- kernel,
- scheduler,
- pamięć,
- syscall ABI,
- filesystem,
- networking,
- GUI,
- format aplikacji,
- aktualizacje.

## 37.2. Brak zależności od cudzego kernela

Bursztyn OS nie jest:
- dystrybucją Linux,
- nakładką na Windows,
- forkiem BSD.

To własna implementacja.

## 37.3. Spójny własny model

Mechanizmy:
- BWS,
- PZB,
- BSP2,
- `.bur`,
- `.cebula`,
- BursztynUI

powstają jako jedna spójna platforma.

## 37.4. Bounded design

Nowsze komponenty używają:
- jawnych limitów,
- generation handles,
- bounded queues,
- walidacji rozmiarów,
- cleanup owner resources.

---

# 38. Główne przewagi dojrzałości Windows 11

Windows ma ogromną przewagę w:
1. SMP,
2. sterownikach,
3. GPU,
4. wirtualizacji,
5. power management,
6. security hardware integration,
7. TPM/Secure Boot,
8. szyfrowaniu dysku,
9. memory management,
10. networking,
11. audio,
12. accessibility,
13. hardware compatibility,
14. application compatibility,
15. development tools,
16. enterprise management,
17. testowaniu,
18. obsłudze wielu języków i input methods.

---

# 39. Najważniejsze przyszłe kroki dla Bursztyna

Patrząc na różnice z Windows, największą wartość techniczną dałyby:
1. **SMP**
2. **NVMe**
3. **pełny model wielowątkowy**
4. **async I/O**
5. **wiele jednoczesnych socketów TCP**
6. **IPv6**
7. **pełny audio stream/mixer API**
8. **VirtIO-GPU jako etap pośredni do sprzętowego GPU**
9. **Secure Boot**
10. **TPM-backed security**
11. **szyfrowanie woluminu/danych użytkownika**
12. **suspend/resume i power management**
13. **USB mass storage**
14. **lepszy PnP**
15. **fuzzing BWS i parserów**
16. **centralna dokumentacja ABI**
17. **testy na fizycznym sprzęcie**

---

# 40. Czego nie należy pisać w publicznym porównaniu

Bez benchmarków i audytu nie należy twierdzić:

> „Bursztyn OS jest szybszy od Windows 11.”

Nie należy twierdzić:

> „Bursztyn OS jest bezpieczniejszy, bo ma PZB.”

Nie należy pisać:

> „Windows ma ciężkie jądro, a Bursztyn ma lekkie.”

bez porównywalnych pomiarów.

Nie należy pisać:

> „Bursztyn ma lepsze aktualizacje.”

Można natomiast napisać:

> „Bursztyn wybrał prosty model A/B z automatycznym rollbackiem, podczas gdy Windows używa component servicing i cumulative updates.”

Nie należy pisać:

> „Bursztyn wspiera 4K tak samo jak Windows.”

Samo ustawienie framebufferu 3840×2160 nie oznacza pełnego, akcelerowanego 4K desktop stacku.

---

# 41. Rzetelny opis Bursztyn OS na tle Windows 11

> **Bursztyn OS jest autorskim systemem operacyjnym x86-64 rozwijanym od zera. Ma własne jądro, Ring 3, ABI BWS, model uprawnień PZB, system plików BSP2, stos sieciowy, framework BursztynUI, instalator i system aktualizacji A/B. Projekt osiągnął już etap kompletnego środowiska eksperymentalnego, ale pod względem wielordzeniowości, sterowników, GPU, bezpieczeństwa sprzętowego, pamięci wirtualnej, power management i kompatybilności aplikacji pozostaje daleko za dojrzałością Windows 11.**

---

# 42. Ocena końcowa

Bursztyn OS i Windows 11 mają pewne wspólne idee na bardzo ogólnym poziomie:
- kernel mode / user mode,
- procesy,
- pamięć wirtualna,
- syscalle,
- sterowniki,
- system plików,
- GUI,
- networking,
- security permissions.

Jednak są projektami o całkowicie innej skali.

## Windows 11

Windows 11 jest:
- produkcyjnym systemem komercyjnym,
- systemem x64 i Arm64,
- wielordzeniowym,
- sprzętowo akcelerowanym,
- wyposażonym w ogromny ekosystem sterowników i aplikacji,
- zintegrowanym z Secure Boot, TPM i zaawansowanym modelem bezpieczeństwa,
- projektowanym do działania na dużej liczbie urządzeń OEM.

## Bursztyn OS

Bursztyn jest:
- rzeczywistym własnym systemem bare-metal,
- obecnie x86-64,
- jednordzeniowym,
- programowo renderującym GUI,
- eksperymentalnym,
- rozwijanym w szybkim tempie,
- bardzo wartościowym edukacyjnie i badawczo.

Najbardziej uczciwy wniosek:

> **Bursztyn OS nie jest obecnie funkcjonalnym odpowiednikiem Windows 11 ani konkurentem o podobnej dojrzałości. Jest jednak realnym, niezależnym systemem operacyjnym, który implementuje coraz większą część pełnego stosu — od bootu i pamięci, przez Ring 3 i własne ABI, po filesystem, networking, GUI, instalator i aktualizacje. Jego największą wartością jest samodzielne projektowanie tych warstw oraz możliwość eksperymentowania z własną architekturą zamiast budowania kolejnej nakładki na istniejący system.**

---

# 43. Źródła — Bursztyn OS

Do opracowania wykorzystano przede wszystkim aktualniejsze dokumenty projektu:

- `01_wstep_i_filozofia.md`
- `02_architektura_systemu.md`
- `03_proces_rozruchu.md`
- `04_zarzadzanie_sprzetem.md`
- `04_audyt_gui_schedulera.md`
- `05_hpet_i_pacing.md`
- `05_bursztynowy_system_plikow.md`
- `06_wywolania_systemowe.md` — częściowo historyczny względem nowszego ABI
- `07_ekosystem_i_formaty.md`
- `08_bursztynowy_slownik_i_architektura.md`
- `09_tryb_graficzny.md`
- `10_siec.md`
- `11_audyt_bws_nowoczesne.md`
- `instalator_audyt.md`
- `ustawienia_ekranu_audyt.md`
- `aktualizacja-systemu.md`
- ostatnie wyniki `Testy Bursztyn OS`
- ostatni raport naprawy popupów i generacji framebufferu przekazany w toku rozwoju.

Przy rozbieżnościach priorytet należy nadawać:
1. aktualnemu kodowi,
2. nowszym audytom,
3. aktualnym testom runtime,
4. dopiero potem starszym rozdziałom dokumentacji.

---

# 44. Źródła — Microsoft / Windows 11

## Aktualne wydania
- Windows 11 release information  
  https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information
- Windows 11 version 26H1 for IT pros  
  https://learn.microsoft.com/en-us/windows/whats-new/windows-11-version-26h1
- Windows 11 version 25H2 release health  
  https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-25h2
- Windows 11 KB5129195 — 14 September 2026  
  https://support.microsoft.com/en-us/servicing/os/windows-11/2026/09/kb5129195-windows-11-24h2-25h2-security-update

## Wymagania systemowe
- Windows 11 requirements  
  https://learn.microsoft.com/en-us/windows/whats-new/windows-11-requirements

## Architektura i kernel
- Windows Kernel-Mode Executive Support Library  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/windows-kernel-mode-executive-support-library
- Virtual Address Spaces  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/virtual-address-spaces
- Windows kernel-mode memory manager  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/windows-kernel-mode-memory-manager
- Scheduling Priorities  
  https://learn.microsoft.com/en-us/windows/win32/procthread/scheduling-priorities
- Processor Groups  
  https://learn.microsoft.com/en-us/windows/win32/procthread/processor-groups

## Windows on Arm
- Windows on Arm overview  
  https://learn.microsoft.com/en-us/windows/arm/overview
- Add Arm support  
  https://learn.microsoft.com/en-us/windows/arm/add-arm-support

## Sterowniki
- Windows Driver Frameworks  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/
- Choose a Driver Model  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/choosing-a-driver-model

## Grafika
- WDDM Architecture  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture
- WDDM overview  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-display-driver-model-design-guide
- DirectComposition architecture  
  https://learn.microsoft.com/en-us/windows/win32/directcomp/architecture-and-components
- Windows Graphics Architecture  
  https://learn.microsoft.com/en-us/windows/win32/learnwin32/overview-of-the-windows-graphics-architecture

## Bezpieczeństwo
- Memory Integrity / HVCI  
  https://learn.microsoft.com/en-us/windows/security/hardware-security/enable-virtualization-based-protection-of-code-integrity
- VBS  
  https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-vbs
- Credential Guard  
  https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/
- Smart App Control / Application control  
  https://learn.microsoft.com/en-us/windows/security/book/application-security-application-and-driver-control
- TPM recommendations  
  https://learn.microsoft.com/en-us/windows/security/hardware-security/tpm/tpm-recommendations
- BitLocker  
  https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/

## Filesystem
- NTFS overview  
  https://learn.microsoft.com/en-us/windows-server/storage/file-server/ntfs-overview
- Filesystem functionality comparison  
  https://learn.microsoft.com/en-us/windows/win32/fileio/filesystem-functionality-comparison

## Sieć
- Network protocol support in Windows / Winsock  
  https://learn.microsoft.com/en-us/windows/win32/winsock/network-protocol-support-in-windows

## Audio
- Windows Audio Architecture  
  https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/windows-audio-architecture

## Aplikacje i pakiety
- Packaging overview  
  https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/packaging/
- MSIX overview  
  https://learn.microsoft.com/en-us/windows/msix/overview

## Aktualizacje
- Servicing stack updates  
  https://learn.microsoft.com/en-us/windows/deployment/update/servicing-stack-updates
- Windows client update release cycle  
  https://learn.microsoft.com/en-us/windows/deployment/update/release-cycle

## Wirtualizacja
- Hyper-V Architecture  
  https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/architecture
- Windows Sandbox architecture  
  https://learn.microsoft.com/en-us/windows/security/threat-protection/windows-sandbox/windows-sandbox-architecture

## Power management
- About Power Management  
  https://learn.microsoft.com/en-us/windows/win32/power/about-power-management

---

# 45. Zalecenie dla dokumentacji Bursztyn OS

Aby przyszłe porównania były jeszcze bardziej jednoznaczne, warto stworzyć jeden nadrzędny plik:

`docs/STAN_SYSTEMU_CURRENT.md`

Powinien zawierać:
- datę,
- commit,
- wersję systemu,
- obsługiwaną architekturę CPU,
- UP/SMP,
- maksymalny numer BWS,
- liczbę procesów i włókien,
- limity pamięci,
- stan swapu,
- listę sterowników,
- stan NVMe,
- stan USB storage,
- stan audio,
- stan IPv6,
- liczbę równoczesnych TCP sockets,
- stan Secure Boot,
- stan TPM,
- stan szyfrowania,
- listę backendów grafiki,
- stan GPU acceleration,
- liczbę elementów BursztynUI,
- ostatni pełny wynik PASS/FAIL,
- listę znanych regresji,
- status QEMU,
- status testów na prawdziwym sprzęcie.

Taki dokument powinien być nadrzędnym źródłem faktów dla dokumentacji, postów, stron projektu i przyszłych porównań.
