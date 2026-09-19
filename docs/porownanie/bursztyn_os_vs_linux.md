# Bursztyn OS vs Linux - rzetelne porównanie techniczne

**Data opracowania:** 2026-09-18  
**Stan Bursztyn OS:** na podstawie aktualnych plików projektu, nowszych audytów oraz ostatnich wyników testów.  
**Punkt odniesienia po stronie Linux:** Linux **7.2.6 stable** (kernel.org, 2026-09-14); mainline: 7.3-rc3  
**Cel dokumentu:** porównanie architektury, funkcji i dojrzałości technicznej bez marketingowego zawyżania możliwości którejkolwiek strony.

> **Ważne rozróżnienie:** Linux sam w sobie jest **jądrem**, a nie kompletnym desktopowym systemem operacyjnym. Typowy system „Linux” używany przez użytkownika to dystrybucja składająca się z jądra Linux oraz wielu komponentów user space: libc, init/service managera, pakietów, serwera grafiki lub kompozytora, środowiska graficznego, sterowników user space i aplikacji.  
>
> Bursztyn OS jest natomiast rozwijany jako **cały własny system**: jądro, ABI/BWS, system plików, GUI, format aplikacji, model uprawnień, instalator i aplikacje systemowe. Dlatego poniższe porównanie rozdziela to, co należy do samego kernela Linux, od tego, co zwykle zapewnia współczesna dystrybucja GNU/Linux.

---

## 1. Podsumowanie

| Obszar | Bursztyn OS - stan projektu 2026 | Linux 7.2.6 + typowy współczesny stack dystrybucji |
|---|---|---|
| **Zakres projektu** | Pełny autorski system operacyjny od bare-metal do GUI | Linux jest jądrem; kompletne środowisko zapewnia dystrybucja i projekty user space |
| **Architektura jądra** | Monolityczne jądro x86-64, własne sterowniki Ring 0 | Monolityczne, modularne jądro z ładowalnymi modułami |
| **Architektury CPU** | x86-64 | Bardzo szeroki zestaw architektur; m.in. x86, arm/arm64, RISC-V, LoongArch, s390 i inne |
| **SMP** | Obecnie jednordzeniowy | Pełne SMP, NUMA, CPU hotplug i skalowanie na wiele CPU |
| **Planista** | Round-Robin, LAPIC kalibrowany HPET, 250 Hz | Rozbudowane klasy planowania; fair scheduling oparty współcześnie o EEVDF, plus RT/deadline i inne mechanizmy |
| **Pamięć wirtualna** | PMM/VMM, x86-64 paging, SHM; brak swapu i kompresji stron | Rozbudowany VM, reclaim, swap, zswap, NUMA, huge pages i wiele polityk pamięci |
| **ABI / syscalle** | Własne BWS przez `SYSCALL`; aktualne testy projektu używają numerów co najmniej do 87 | Bardzo stabilne userspace ABI; setki syscalli i inne interfejsy jak ioctl, procfs, sysfs, netlink, eBPF |
| **Bezpieczeństwo** | Ring 0/Ring 3, PZB 0–5, maski `PRAWO_*`, generacyjne uchwyty | UID/GID, capabilities, LSM, SELinux/AppArmor/Smack/TOMOYO, Landlock, seccomp, namespaces, cgroups, lockdown/IMA/fs-verity zależnie od konfiguracji |
| **System plików** | BSP2/PSF, 4 KiB bloki, własny format | Wiele systemów: ext4, XFS, Btrfs, F2FS, NFS, OverlayFS i inne |
| **Nośniki** | Aktualnie SATA AHCI; brak działającego NVMe/USB mass storage jako dysku instalacyjnego | Bardzo szeroka warstwa blokowa: NVMe, SATA, USB, SCSI, virtio, RAID, device-mapper itd. |
| **Aktualizacje** | Własny model A/B, rollback, RSA-2048/SHA-256, dwa rekordy boot state | Brak jednego modelu w kernelu; zależy od dystrybucji: klasyczne pakiety, transactional/atomic, obrazy, snapshots itp. |
| **Grafika** | Programowy kompozytor CPU i framebuffer; BursztynUI | DRM/KMS w kernelu + Mesa/Vulkan/OpenGL + Wayland/X11 + compositor; sprzętowa akceleracja GPU |
| **GUI** | Własny BursztynUI, ok. 62 publiczne elementy frameworka | Brak jednego GUI „Linux”; środowiska korzystają np. z GTK, Qt, Wayland, KDE, GNOME i innych stosów |
| **Sieć** | E1000, IPv4, ARP, ICMP, UDP, DHCP, DNS, TCP, HTTP/HTTPS/TLS; socket facade z istotnymi ograniczeniami | Dojrzały stos IPv4/IPv6, TCP/UDP, routing, netfilter, namespaces, zaawansowane NIC offloady, eBPF/XDP |
| **Audio** | HDA w ograniczonym modelu | ALSA w kernelu + user-space audio stack zależny od dystrybucji |
| **Kontenery / izolacja** | Własne PZB/IPC/SHM, brak odpowiednika pełnego modelu kontenerów | namespaces + cgroup v2 + seccomp + capabilities + LSM; fundament Dockera/Podmana/LXC |
| **Obserwowalność** | Własne logi, testy i diagnostyka | perf, tracepoints, ftrace, eBPF/BTF, debugfs, sysfs, procfs i rozbudowane narzędzia |
| **Sterowniki** | Wąski, własny zestaw sterowników | Ogromna baza sterowników sprzętowych |
| **Ekosystem aplikacji** | `.bur`, `.cebula`, BursztynUI, rozwijane Avocado | ELF, POSIX, glibc/musl i ogromny ekosystem oprogramowania oraz języków |
| **Dojrzałość** | Zaawansowany projekt eksperymentalny w aktywnej stabilizacji | Produkcyjne jądro używane od urządzeń embedded po serwery, HPC, desktop i chmurę |

---

# 2. Co właściwie porównujemy?

Porównanie „Bursztyn OS vs Linux” jest trudniejsze niż „Bursztyn OS vs macOS”.

macOS jest kompletnym systemem operacyjnym. Linux jest jądrem.

Dlatego należy rozróżnić trzy poziomy:

1. **Bursztyn Kernel vs Linux Kernel** — architektura jądra, pamięć, scheduler, sterowniki, sieć.
2. **Bursztyn OS vs typowa dystrybucja Linux** — instalator, aktualizacje, pakiety, system plików, usługi.
3. **BursztynUI vs desktop Linux** — framework GUI i compositor kontra Wayland/X11, Mesa, GTK/Qt i środowiska graficzne.

W dokumencie określenie „Linux desktop” oznacza typowy współczesny stos oparty na kernelu Linux, a nie funkcję należącą wyłącznie do samego jądra.

---

# 3. Wersja referencyjna Linux

Na dzień 2026-09-18 strona kernel.org podaje:

- **stable: Linux 7.2.6**, wydany 2026-09-14,
- **mainline: Linux 7.3-rc3**, 2026-09-13,
- aktywne gałęzie longterm obejmują również starsze serie, np. 6.18, 6.12, 6.6, 6.1, 5.15 i 5.10.

Nie oznacza to, że wszystkie dystrybucje używają 7.2.6. Dystrybucje często pozostają na wersjach LTS i przenoszą poprawki bezpieczeństwa bez zmiany głównej wersji kernela.

W tym dokumencie Linux 7.2.6 służy jako punkt odniesienia dla aktualnego upstreamu.

---

# 4. Architektura jądra

## Bursztyn OS

Bursztyn OS jest własnym, freestanding systemem dla x86-64.

Architektura rozdziela:

- **Ring 0** — kernel i sterowniki niskopoziomowe,
- **Ring 3** — programy `.bur`, Menedżer Okien i pozostałe aplikacje użytkownika.

Jądro jest **monolityczne**. Sterowniki takie jak:

- AHCI,
- E1000,
- xHCI/HID,
- HDA,

działają po stronie kernela.

Projekt nie jest nakładką na Linux i nie używa jego kernela.

## Linux

Linux również jest zasadniczo **jądrem monolitycznym**, ale z silną modularnością.

Obsługuje:

- kod wbudowany w kernel,
- loadable kernel modules,
- szerokie API sterowników,
- dynamiczne wykrywanie sprzętu.

Linux nie posiada stabilnego binarnego ABI **wewnątrz kernela dla sterowników**, natomiast userspace ABI jest projektowane jako bardzo stabilne.

## Różnica

Bursztyn i Linux należą do podobnej szerokiej rodziny „monolithic kernel”, ale skala jest zupełnie inna.

Linux ma dziesięciolecia warstw abstrakcji, ogromną liczbę sterowników i wsparcie wielu architektur.

Bursztyn ma znacznie mniejszą bazę kodu i dużo węższy zakres sprzętu, co ułatwia zrozumienie całego systemu, ale nie daje dojrzałości ani kompatybilności Linuxa.

---

# 5. Architektury procesora

## Bursztyn OS

Aktualnie:

- x86-64,
- Long Mode,
- APIC,
- HPET,
- własne GDT/TSS/IDT/VMM.

Nie ma obecnie portów ARM64, RISC-V ani innych architektur.

## Linux

Linux jest przenośnym kernelem używanym na wielu architekturach.

Aktualna dokumentacja kernela obejmuje m.in.:

- x86,
- arm,
- arm64,
- RISC-V,
- LoongArch,
- s390,
- powerpc i inne.

Wsparcie konkretnych funkcji zależy od architektury.

## Wniosek

Bursztyn jest celowo skoncentrowany na jednej platformie. Linux jest projektem wieloarchitekturowym.

---

# 6. Scheduler i SMP

## Bursztyn OS

Aktualne audyty opisują:

- scheduler Round-Robin,
- HPET jako źródło czasu,
- kalibrację LAPIC,
- periodic clockevent 250 Hz,
- PID 0 uczestniczący w round-robin,
- wykonywanie pending compositor work poza IRQ.

Projekt pozostaje **jednordzeniowy**.

To jedno z głównych ograniczeń obecnej architektury.

## Linux

Linux obsługuje:

- SMP,
- wiele CPU i rdzeni,
- CPU hotplug,
- NUMA,
- affinity,
- wiele klas planowania.

Dokumentacja współczesnego kernela opisuje **EEVDF (Earliest Eligible Virtual Deadline First)** jako następcę klasycznego modelu CFS dla fair scheduling. Przejście rozpoczęło się od Linux 6.6.

Linux ma również osobne polityki dla zadań czasu rzeczywistego i deadline.

## Wniosek

W tym obszarze różnica jest fundamentalna:

**Bursztyn OS = prosty scheduler UP**  
**Linux = bardzo rozbudowane wieloprocesorowe planowanie produkcyjne**

SMP powinno być jednym z przyszłych kamieni milowych Bursztyn OS.

---

# 7. Pamięć fizyczna i wirtualna

## Bursztyn OS

Projekt ma własny:

- PMM,
- VMM,
- 4-poziomowe stronicowanie x86-64,
- izolację Ring 3,
- prywatne przestrzenie procesów,
- SHM przez BWS 64–67,
- generacyjne uchwyty do zasobów.

Nowszy audyt BWS podaje dla SHM m.in.:

- strony 4 KiB,
- limit 2 MiB na obiekt,
- 8 MiB globalnie,
- zeroed frames,
- dostęp per PID,
- brak eksponowania fizycznych adresów Ring 3.

Bursztyn nie ma obecnie:

- swapu,
- zswap,
- kompresji pamięci,
- NUMA,
- dojrzałego memory reclaim podobnego do Linux.

## Linux

Linux ma rozbudowany memory management:

- page cache,
- reclaim,
- swap,
- zswap,
- anonymous/file-backed VM,
- huge pages,
- NUMA,
- mmap,
- memory cgroups,
- overcommit policies,
- różne mechanizmy reclaim i OOM.

`zswap` może kompresować strony w RAM zanim trafią do właściwego swapu.

## Wniosek

Bursztyn ma poprawne fundamenty nowoczesnej izolacji pamięci, ale Linux ma o kilka poziomów bardziej rozwinięte zarządzanie presją pamięci.

---

# 8. Wywołania systemowe i ABI

## Bursztyn OS

Aplikacje Ring 3 używają autorskiego ABI **BWS**.

BWS wykorzystuje na x86-64 instrukcję:

`SYSCALL`

Aktualna dokumentacja projektu nie jest jeszcze całkowicie skonsolidowana:

- starszy dokument opisuje BWS 1–26,
- nowszy audyt dokumentuje BWS 1–83,
- podsystem aktualizacji używa późniejszych numerów,
- najnowsza aplikacja testowa wykonuje testy BWS 86–87.

Dlatego rzetelny opis aktualnego stanu brzmi:

> Aktualne drzewo projektu używa BWS co najmniej do numeru 87, ale centralny dokument ABI wymaga aktualizacji.

Nowsze BWS obejmują m.in.:

- SHM,
- IPC,
- monitor/work area,
- geometrię okien,
- file watch,
- clipboard MIME,
- kursory,
- socket facade,
- aktualizacje,
- motywy.

## Linux

Linux ma rozbudowane userspace ABI.

Ważna zasada upstreamu:

> nowy syscall staje się częścią API kernela i musi być utrzymywany długoterminowo.

Poza syscallami Linux używa również:

- file descriptors,
- ioctl,
- sysfs,
- procfs,
- netlink,
- eventfd,
- epoll,
- io_uring,
- eBPF,
- specjalizowanych urządzeń i pseudo-filesystemów.

Kernelowa dokumentacja ABI rozróżnia poziomy stabilności, ale syscalle i podstawowe userspace ABI są traktowane jako szczególnie stabilne.

## Wniosek

BWS jest dużo mniejsze i prostsze, co jest zaletą projektową na obecnym etapie.

Linux oferuje znacznie bardziej rozbudowany i historycznie kompatybilny interfejs userspace.

---

# 9. Model bezpieczeństwa

## Bursztyn OS

Bursztyn stosuje:

- sprzętowy Ring 0 / Ring 3,
- PZB 0–5,
- 64-bitowe maski `PRAWO_*`,
- kontrolowane copy_from_user/copy_to_user,
- generacyjne uchwyty,
- właściciela zasobów per PID,
- ograniczone kolejki i bounded allocations,
- cleanup zasobów po śmierci procesu.

PZB jest autorskim modelem polityki uprawnień.

Nie należy jednak twierdzić, że sam PZB daje bezpieczeństwo równoważne do wieloletniego stacku Linux.

## Linux

Linux ma warstwowy model:

- UID/GID,
- DAC,
- POSIX ACL,
- capabilities,
- LSM framework,
- SELinux,
- AppArmor,
- Smack,
- TOMOYO,
- Landlock,
- seccomp-BPF,
- namespaces,
- cgroups,
- moduły podpisywane kryptograficznie,
- IMA/IPE/fs-verity zależnie od konfiguracji.

Landlock umożliwia nawet nieuprzywilejowanym procesom samoograniczenie swoich praw.

Seccomp pozwala ograniczyć zestaw syscalli widocznych dla procesu.

## Wniosek

PZB jest czytelnym i ciekawym fundamentem dla własnego systemu.

Linux oferuje jednak znacznie większy zestaw mechanizmów bezpieczeństwa i możliwość składania ich w warstwy.

---

# 10. Procesy, włókna i izolacja

## Bursztyn OS

Projekt posiada:

- procesy Ring 3,
- event queues,
- SHM,
- IPC endpoints,
- focus/capture dla GUI,
- generacyjne uchwyty.

Dokumentacja używa pojęcia **włókno**, ale aktualny publiczny opis architektury współbieżności nadal wymaga konsolidacji. Nie należy bez weryfikacji kodu przypisywać projektowi rozbudowanego modelu wielowątkowego podobnego do Linux pthreads.

## Linux

Linux posiada dojrzały model tasks/threads oraz:

- `clone`,
- pthreads po stronie libc,
- PID namespaces,
- user namespaces,
- network namespaces,
- mount namespaces,
- IPC namespaces,
- cgroup namespaces.

To właśnie te funkcje, razem z cgroup, seccomp i capabilities, stanowią fundament współczesnych kontenerów.

---

# 11. Kontenery i cgroup

## Bursztyn OS

Bursztyn nie ma odpowiednika pełnego modelu:

- namespaces,
- cgroup v2,
- container runtime.

PZB i bounded IPC/SHM rozwiązują inny problem: kontrolę zaufania i uprawnień procesów.

## Linux

cgroup v2 organizuje procesy hierarchicznie i rozdziela zasoby zgodnie z polityką.

Namespaces izolują widoki m.in.:

- PID,
- mount,
- IPC,
- network,
- user,
- UTS,
- cgroup.

To jeden z najważniejszych obszarów, w których Linux wykracza daleko poza obecny zakres Bursztyna.

---

# 12. System plików

## Bursztyn OS

Aktualny audyt instalatora opisuje BSP2/PSF jako jeden system plików:

- blok 4096 B,
- 64-bitowe adresy bloków,
- wskaźniki bezpośrednie,
- pojedynczy poziom pośredni,
- podwójny poziom pośredni.

Obecny instalator tworzy:

- ESP FAT32 512 MiB,
- BSP2 do maks. 2 GiB w tym wydaniu.

Starsze pliki dokumentacji BSP zawierają informacje historyczne, które nie powinny być używane jako aktualny stan bez porównania z audytem instalatora.

## Linux

Linux VFS umożliwia obsługę bardzo wielu filesystemów.

Przykłady:

- ext4,
- XFS,
- Btrfs,
- F2FS,
- NFS,
- tmpfs,
- OverlayFS,
- FAT/exFAT,
- NTFS i inne.

ext4 typowo używa bloków 4 KiB i ma journaling przez JBD2.

Btrfs oferuje m.in.:

- copy-on-write,
- snapshots,
- checksums,
- subvolumes,
- compression,
- multi-device support.

## Wniosek

BSP2 jest rzeczywistym własnym filesystemem i ważnym elementem tożsamości Bursztyn OS.

Linux ma jednak zupełnie inną skalę funkcji filesystemowych i interoperacyjności.

---

# 13. Warstwa blokowa i urządzenia magazynujące

## Bursztyn OS

Stan aktualnego instalatora:

- SATA AHCI,
- ATA LBA48,
- sektory 512 B / 512e,
- GPT,
- brak obsługi 4Kn,
- brak działającego NVMe jako backendu blokowego,
- brak IDE,
- brak USB mass storage jako dysku instalacyjnego.

Jeżeli nie ma AHCI, instalator przerywa operację bez zapisu.

## Linux

Linux obsługuje szeroką gamę urządzeń:

- SATA/AHCI,
- NVMe,
- USB storage,
- SCSI,
- virtio-blk,
- virtio-scsi,
- device-mapper,
- software RAID,
- multipath i inne.

## Wniosek

Warstwa blokowa jest jednym z miejsc, gdzie Bursztyn powinien rosnąć etapami. Naturalnym następnym dużym krokiem byłby NVMe.

---

# 14. Grafika

## Bursztyn OS

Bursztyn wykorzystuje własny software compositor.

Aktualne elementy:

- framebuffer,
- backbuffer,
- powierzchnie aplikacji,
- popup surfaces,
- własny Menedżer Okien,
- BursztynUI,
- skalowanie UI 100–200%,
- runtime display modes dla Bochs VBE.

Brak obecnie:

- sterownika GPU,
- DRM-like memory managera,
- Vulkan/OpenGL,
- hardware compositing.

Ostatnie prace dotyczyły m.in. generacji framebufferu i bezpiecznego unieważniania popupów po zmianie rozdzielczości.

## Linux

Kernel zapewnia **DRM/KMS**.

DRM obsługuje m.in.:

- GPU memory management,
- command submission,
- DMA,
- fencing,
- vblank,
- framebuffer management.

KMS opisuje pipeline:

`framebuffer -> plane -> CRTC -> encoder/connector`

Poza kernelem typowy desktop korzysta z:

- Mesa,
- Vulkan,
- OpenGL,
- EGL,
- Wayland lub X11,
- kompozytora desktopowego.

Mesa ma sterowniki dla szerokiej gamy GPU Intel, AMD, NVIDIA, Qualcomm, ARM i innych.

## Wniosek

To jeden z największych dystansów technologicznych.

Bursztyn ma już własny compositor i UI, ale Linux posiada dojrzały stack GPU i wieloletnie sterowniki 2D/3D/compute.

---

# 15. Wayland i model desktopu

## Bursztyn OS

Bursztyn ma jeden własny stack GUI kontrolowany przez projekt:

- compositor,
- Menedżer Okien,
- BursztynUI,
- własne popupy/dialogi,
- własny model zdarzeń.

Według aktualnego inwentarza BursztynUI ma około **62 publicznych elementów frameworka**, jeśli liczyć kontrolki, dialogi i elementy infrastruktury.

## Linux

Nie istnieje jeden „Linux GUI”.

Współczesny desktop może używać:

- Wayland,
- X11,
- GNOME,
- KDE Plasma,
- wlroots,
- Mutter,
- KWin,
- GTK,
- Qt,
- innych frameworków i compositorów.

Wayland jest protokołem pomiędzy compositorami i klientami, a nie samym kompletnym desktopem.

## Wniosek

Bursztyn ma większą spójność architektoniczną, bo cały stack jest rozwijany jako jedna platforma.

Linux ma większą elastyczność i dojrzałość, ale również wiele możliwych kombinacji komponentów.

---

# 16. Zmiana rozdzielczości i DPI

## Bursztyn OS

Audyt ekranu dokumentuje m.in.:

- Bochs VBE runtime mode switch,
- 15-sekundowy rollback,
- konfigurację `/ustawienia/ekran.cfg`,
- BWS62 work area,
- UI scale 100%, 125%, 150%, 175%, 200%,
- tryby do 3840×2160 w testowym QEMU przy odpowiednim VRAM.

Trzeba jednak uwzględnić ostatnią regresję:

po zmianie na 1920×1080 `DialogPliku` potrafił się zawieszać.

Wprowadzono poprawkę związaną z **generacją popupów/framebufferu**, ale pełna ręczna akceptacja tego scenariusza nadal była otwarta w ostatnim przekazanym stanie.

Nie należy więc pisać, że cały high-resolution GUI jest już produkcyjnie stabilny.

## Linux

DRM/KMS, Wayland i współczesne compository obsługują dynamiczny modesetting, wiele ekranów, wysokie DPI i nowoczesne GPU.

Szczegóły skalowania zależą od compositora i środowiska graficznego.

---

# 17. Sieć

## Bursztyn OS

Aktualny stack obejmuje:

- Ethernet,
- E1000,
- ARP,
- IPv4,
- ICMP,
- UDP,
- DHCP,
- DNS,
- TCP,
- HTTP,
- HTTPS/TLS.

BWS 79–83 dodaje owned TCP/IPv4 socket facade:

- create,
- connect,
- send,
- receive,
- close.

Kluczowe ograniczenie:

> underlying TCP stack nadal pozwala tylko na jedną systemową sesję jednocześnie.

Nie ma pełnej współbieżności socketów ani kompletnego event readiness modelu.

W aktualnych plikach projektu nie ma kompletnego IPv6.

## Linux

Linux ma dojrzały stack sieciowy obejmujący m.in.:

- IPv4,
- IPv6,
- TCP,
- UDP,
- raw sockets,
- routing,
- bridging,
- VLAN,
- namespaces,
- netfilter,
- IPsec,
- SCTP,
- zaawansowane offloady NIC,
- tun/tap,
- eBPF/XDP.

Dokumentacja kernela obejmuje także nowoczesne ścieżki zero-copy, w tym io_uring zero-copy receive dla odpowiedniego sprzętu.

## Wniosek

Bursztyn ma działającą własną sieć, co jest znaczącym osiągnięciem systemu od zera.

Linux ma jednak jeden z najbardziej rozwiniętych stosów sieciowych na świecie.

---

# 18. Audio

## Bursztyn OS

Aktualny audyt BWS opisuje HDA jako warstwę nadal częściową:

- stałe 48 kHz,
- S16,
- stereo,
- BWS 27 jako test tone,
- brak publicznego owned-stream API,
- brak pełnej negocjacji formatów,
- brak kompletnego systemowego miksera.

## Linux

Kernel Linux zawiera ALSA.

ALSA posiada:

- PCM,
- mixer/control API,
- sequencer,
- MIDI,
- channel mapping,
- timestamping,
- compress offload,
- ASoC,
- obsługę wielu klas urządzeń audio.

Dystrybucje zwykle dokładają osobny user-space audio stack.

---

# 19. Sterowniki

## Bursztyn OS

Własny zestaw obejmuje m.in.:

- APIC/HPET,
- AHCI,
- E1000,
- xHCI,
- HID keyboard/mouse,
- HDA,
- framebuffer backends.

Zaletą jest pełna kontrola nad kodem.

Wadą jest bardzo ograniczony zakres obsługiwanego sprzętu.

## Linux

Jednym z największych atutów Linux jest ogromna baza sterowników:

- GPU,
- sieć,
- NVMe,
- SATA,
- USB,
- Bluetooth,
- Wi-Fi,
- audio,
- kamery,
- HID,
- platform hardware,
- SoC,
- virtualizacja,
- serwery i urządzenia embedded.

To rezultat wieloletniego rozwoju i współpracy wielu producentów.

---

# 20. Moduły i rozszerzalność kernela

## Bursztyn OS

Obecny model jest bardziej statyczny. Większość głównych komponentów jest częścią kernela.

Projekt nie posiada ekosystemu loadable kernel modules porównywalnego z Linux.

## Linux

Linux ma loadable kernel modules.

Kernel może dodatkowo wymagać kryptograficznych podpisów modułów.

Dokumentacja Linux opisuje m.in.:

- podpisy X.509,
- RSA,
- ECDSA,
- ML-DSA,
- SHA-2/SHA-3,
- tryb wymuszający odrzucanie niepodpisanych modułów.

To nie oznacza, że każda dystrybucja konfiguruje wszystkie te zabezpieczenia identycznie.

---

# 21. Aktualizacje

## Bursztyn OS

Bursztyn posiada własny model A/B:

- slot aktywny i nieaktywny,
- health checkpoint,
- rollback,
- RSA-2048,
- SHA-256,
- redundantny boot state,
- CRC32,
- licznik generacji,
- integrację z `grubenv`.

Autorytatywne rekordy stanu znajdują się obecnie w LBA 2046/2047.

To bardzo ciekawy element projektu, ale dokumentacja sama sugeruje, że w przyszłości dedykowana partycja GPT byłaby bardziej standardowym rozwiązaniem.

## Linux

**Linux kernel nie definiuje jednego systemu aktualizacji dystrybucji.**

Model zależy od systemu:

- pakiety aktualizowane in-place,
- aktualizacje obrazu,
- snapshots,
- A/B,
- OSTree-like modele,
- transactional updates.

Dlatego stwierdzenie „Linux ma taki system aktualizacji” byłoby błędne bez wskazania konkretnej dystrybucji.

---

# 22. Secure Boot i integralność

## Bursztyn OS

Aktualny instalator wymaga:

> Secure Boot musi być wyłączony.

`BOOTX64.EFI` nie posiada obecnie pełnego łańcucha podpisu kompatybilnego z firmware Secure Boot.

System ma natomiast własną kryptograficzną walidację pakietów aktualizacyjnych.

## Linux

Wsparcie secure boot jest zależne od:

- firmware,
- bootloadera,
- dystrybucji,
- kluczy,
- konfiguracji kernela.

Kernel posiada mechanizmy takie jak:

- module signature verification,
- lockdown/integrity mechanisms zależnie od konfiguracji,
- IMA/IPE,
- fs-verity,
- dm-verity.

Nie należy przedstawiać secure boot jako jednej identycznej funkcji we wszystkich dystrybucjach.

---

# 23. fs-verity i integralność plików

Linux ma `fs-verity`, które chroni pliki read-only przez Merkle tree.

Aktualnie wspierane są m.in.:

- ext4,
- f2fs,
- btrfs.

Odczyty są weryfikowane względem drzewa Merkle.

Mechanizm może współpracować z:

- IMA,
- IPE,
- podpisami,
- trusted userspace.

Bursztyn nie ma obecnie bezpośredniego odpowiednika tak rozbudowanej warstwy per-file verity.

---

# 24. IPC

## Bursztyn OS

Nowsze BWS 68–71 mają:

- endpoint create,
- send,
- receive,
- close,
- 512-bajtowe messages,
- 16 messages na endpoint,
- 32 endpointy,
- generation handles,
- owner-only receive/close,
- możliwość transferu SHM.

To świadomie bounded design.

## Linux

Linux ma wiele mechanizmów IPC:

- pipes,
- Unix sockets,
- shared memory,
- signals,
- futex,
- eventfd,
- POSIX/System V IPC,
- netlink,
- io_uring,
- memfd,
- pidfd i inne.

## Wniosek

Bursztyn ma spójny mały model IPC, podczas gdy Linux oferuje wieloletnio rozbudowywany zestaw mechanizmów zgodności i wydajności.

---

# 25. Asynchroniczne I/O

## Bursztyn OS

Nowszy audyt BWS jawnie stwierdza:

> pełne async I/O nie zostało sztucznie udawane.

Brakują:

- request submission,
- worker/cancellation lifetime,
- completion records.

To uczciwie oznaczone jako obszar przyszłej pracy.

## Linux

Linux posiada wiele asynchronicznych modeli I/O, w tym współczesne `io_uring`.

io_uring jest integrowane m.in. z:

- plikami,
- socketami,
- buforami,
- mechanizmami zero-copy.

To duża różnica wydajnościowa i architektoniczna.

---

# 26. eBPF

## Bursztyn OS

Bursztyn nie ma odpowiednika eBPF.

## Linux

eBPF tworzy programowalną warstwę wewnątrz kernela z verifierem i wieloma typami programów.

Wykorzystania obejmują m.in.:

- networking,
- tracing,
- observability,
- LSM,
- cgroups.

Jest to jedna z najbardziej zaawansowanych współczesnych funkcji Linux i jednocześnie znaczne źródło złożoności.

---

# 27. Diagnostyka i obserwowalność

## Bursztyn OS

Projekt ma:

- własne logi kernela,
- testy hostowe,
- testy QEMU,
- aplikację `Testy Bursztyn OS`,
- diagnostyczne makra dla GUI/compositora,
- audyty inżynierskie.

Jest to bardzo dobry kierunek dla własnego systemu.

## Linux

Linux ma m.in.:

- tracepoints,
- ftrace,
- perf,
- eBPF,
- BTF,
- kprobes,
- uprobes,
- sysfs,
- procfs,
- debugfs,
- crash tooling,
- sanitizery takie jak KASAN.

Skala diagnostyki Linux jest dużo większa.

---

# 28. Testowanie

## Bursztyn OS

Projekt posiada coraz większy zestaw testów.

Przykłady z aktualnych audytów:

- installer: 4491/4491 PASS w konkretnej suicie,
- ekran: ponad 31 tys. PASS w konkretnej suicie,
- BWS smoke tests,
- testy GUI i kontrolek.

Trzeba jednak rozróżniać wyniki poszczególnych audytów od globalnego stanu systemu.

Ostatni przekazany pełny log `Testy Bursztyn OS` zawierał jeszcze FAIL-e m.in. w:

- BWS 86–87 / motywach,
- popupach BursztynUI,
- UTF-8,
- SelektorKoloru,
- picker lifecycle,
- wybranych testach alokacji toolbar/tooltip.

Dlatego nie należy przedstawiać projektu jako „0 FAIL” globalnie, dopóki nowszy pełny przebieg tego nie potwierdzi.

## Linux

Kernel Linux posiada bardzo rozbudowany proces testowania:

- kernel selftests,
- KUnit,
- xfstests,
- syzkaller używany przez społeczność,
- CI wielu firm,
- testy architektur,
- testy sterowników,
- długotrwałe testy dystrybucji.

Skala testowania jest nieporównywalnie większa.

---

# 29. Stabilność ABI

## Bursztyn OS

BWS jest młodym ABI i nadal szybko rośnie.

Zaletą jest możliwość poprawienia architektury, zanim system stanie się szeroko używany.

Ryzykiem jest szybkie narastanie numeracji i nieaktualnych dokumentów.

Najważniejsza rekomendacja:

utworzyć jeden plik źródłowy:

`docs/BWS_ABI_CURRENT.md`

generowany lub przynajmniej synchronizowany z `bws_api.h`.

## Linux

Userspace ABI jest wyjątkowo stabilne.

Dokumentacja upstream podkreśla, że syscalle powinny być wspierane praktycznie bezterminowo.

Jednocześnie Linux celowo **nie gwarantuje stabilnego wewnętrznego ABI dla modułów/sterowników**.

---

# 30. Ekosystem aplikacji

## Bursztyn OS

Projekt ma:

- `.bur`,
- `.cebula`,
- manifesty,
- BursztynUI,
- własne BWS,
- język Avocado jako kierunek rozwoju.

To buduje całkowicie własny ekosystem.

## Linux

Typowy Linux korzysta z:

- ELF,
- POSIX,
- libc,
- pthreads,
- ogromnego zestawu bibliotek,
- C/C++/Rust/Go/Python/Java/.NET i wielu innych,
- repozytoriów pakietów,
- Flatpak/Snap/AppImage zależnie od środowiska.

Brak zgodności Bursztyna z POSIX oznacza, że istniejącego oprogramowania Linux nie można przenieść automatycznie.

---

# 31. BursztynUI kontra GUI Linux

## BursztynUI

BursztynUI jest jedną spójną biblioteką/frameworkiem kontrolowaną przez projekt.

Aktualny inwentarz obejmuje około 62 elementów, w tym:

- podstawowe kontrolki,
- układy,
- listy i tabele,
- wirtualizowane kontrolki,
- system poleceń,
- menu,
- dialogi,
- `DialogPliku`,
- `DialogKoloru`,
- `DialogKomunikatu`,
- popup infrastructure,
- toast notifications.

To duże osiągnięcie dla projektu budowanego od zera.

## Linux desktop

Linux nie ma jednego odpowiednika BursztynUI.

Możliwe warstwy to np.:

- GTK,
- Qt,
- SDL,
- GLFW,
- Electron,
- Wayland,
- X11,
- wlroots,
- compositor środowiska desktopowego.

Zaletą Linux jest ekosystem i interoperacyjność.

Zaletą BursztynUI jest kontrola całości i jednolita konwencja.

---

# 32. Zarządzanie energią

## Bursztyn OS

Projekt ma ACPI w zakresie potrzebnym m.in. do:

- tabel systemowych,
- shutdown ACPI S5,
- HPET.

Pełne laptopowe power management, suspend/resume i CPU frequency scaling nie jest jeszcze porównywalne z Linux.

## Linux

Linux ma rozbudowany power management:

- suspend-to-idle,
- standby,
- suspend-to-RAM,
- hibernation,
- CPU idle,
- CPU frequency scaling,
- runtime PM urządzeń,
- PM QoS.

To bardzo duży obszar funkcjonalny sam w sobie.

---

# 33. System aktualizacji: różnica filozofii

Bursztyn OS wybrał jeden kontrolowany model:

`A/B + rollback`

Linux pozostawia to dystrybucjom.

Daje to interesującą różnicę:

### Bursztyn

- jeden standard projektu,
- łatwiejszy do formalnego opisania,
- przewidywalna ścieżka rollbacku.

### Linux

- ogromna elastyczność,
- wiele modeli,
- brak jednej wspólnej semantyki dla wszystkich systemów Linux.

Nie jest więc poprawne mówienie, że „Linux ma gorszy/lepszy system aktualizacji” bez wskazania konkretnej dystrybucji.

---

# 34. Gdzie Bursztyn OS jest celowo prostszy

Nie każda różnica jest „brakiem”.

Projekt świadomie upraszcza część mechanizmów:

- jeden wspierany CPU ISA,
- jeden własny ABI,
- własny filesystem,
- ograniczone bounded IPC,
- ograniczone SHM,
- własny framework GUI,
- mała liczba sterowników.

To pomaga w:

- audytowaniu kodu,
- debugowaniu,
- nauce,
- utrzymaniu spójności architektury.

W Linux ta sama funkcjonalność musi działać w ogromnej liczbie konfiguracji.

---

# 35. Gdzie Linux ma przewagę dojrzałości

Największe obszary:

1. SMP i NUMA,
2. ogromna baza sterowników,
3. GPU/DRM/KMS,
4. pamięć wirtualna i reclaim,
5. networking,
6. power management,
7. filesystems,
8. bezpieczeństwo wielowarstwowe,
9. kontenery,
10. observability/eBPF,
11. audio,
12. hardware support,
13. stabilność userspace ABI,
14. skala testów,
15. ekosystem aplikacji.

---

# 36. Gdzie Bursztyn OS ma realną wartość projektową

Nie jako „lepszy od Linux”, lecz jako własny system.

## 36.1. Kontrola całego stosu

Projekt kontroluje:

- boot,
- kernel,
- ABI,
- filesystem,
- uprawnienia,
- format aplikacji,
- GUI,
- instalator,
- aktualizacje.

## 36.2. Czytelna tożsamość

Charakterystyczne mechanizmy:

- BWS,
- PZB,
- BSP2,
- `.bur`,
- `.cebula`,
- BursztynUI,
- Avocado,
- polskie nazewnictwo.

## 36.3. Bounded design

Nowsze BWS stosują:

- limitowane bufory,
- generacyjne uchwyty,
- jawne maksymalne rozmiary,
- kontrolę właściciela.

To dobry kierunek dla kodu systemowego.

## 36.4. Wartość edukacyjna

W Linux ogromna część systemu jest już gotowa.

W Bursztyn OS autor musi sam rozwiązać:

- wejście w Long Mode,
- paging,
- scheduler,
- syscalle,
- filesystem,
- sterowniki,
- networking,
- GUI,
- instalację,
- aktualizacje.

To zupełnie inny rodzaj nauki.

---

# 37. Najważniejsze braki do nadrobienia względem Linux

Jeżeli celem długoterminowym jest przejście od eksperymentalnego systemu do realnie użytkowego desktopu, największy zwrot techniczny dadzą:

1. **SMP**
2. **NVMe**
3. **pełny model wielowątkowy**
4. **async I/O**
5. **wiele jednoczesnych socketów TCP**
6. **IPv6**
7. **rozbudowany audio stream/mixer API**
8. **GPU / VirtIO-GPU jako etap przejściowy**
9. **bardziej formalny VFS lub warstwa filesystem abstraction**
10. **pełny model suspend/resume/power management**
11. **Secure Boot**
12. **szyfrowanie danych**
13. **bardziej rozbudowany sandbox PZB 5**
14. **fuzzing syscalli i parserów**
15. **ciągła synchronizacja dokumentacji ABI z kodem**

---

# 38. Rekomendowane kierunki inspirowane Linux — bez kopiowania Linux

Bursztyn nie musi stawać się klonem Linux, ale może czerpać z jego doświadczeń.

## 38.1. Stabilne ABI user space

Warto przyjąć zasadę podobną do Linux:

> opublikowane BWS nie zmienia semantyki w sposób łamiący istniejące aplikacje.

Nowe funkcje powinny być wersjonowane lub rozszerzane przez:

- `size`,
- `version`,
- `flags`,
- nowe numery BWS.

## 38.2. Brak stabilnego wewnętrznego ABI kernela

Dla wewnętrznych modułów nie trzeba obiecywać stabilności, dopóki projekt szybko się rozwija.

## 38.3. Jasne granice subsystemów

Warto utrzymywać osobne kontrakty:

- scheduler,
- VMM,
- storage,
- network,
- GUI,
- security.

## 38.4. Selftests

Każdy nowy BWS powinien otrzymywać:

- test happy-path,
- test bad pointer,
- test stale handle,
- test permission denied,
- test boundary,
- test cleanup after process death.

## 38.5. Fuzzing

Szczególnie:

- `.cebula`,
- `.bur`,
- BSP2 metadata,
- sieć,
- parsery obrazów,
- dialog file filters,
- UTF-8,
- IPC descriptors.

---

# 39. Rzetelny opis Bursztyn OS na tle Linux

Nie należy pisać:

> „Bursztyn jest szybszy od Linux.”

bez benchmarków.

Nie należy pisać:

> „Bursztyn jest bezpieczniejszy, bo ma PZB.”

bez audytu i modelu zagrożeń.

Nie należy pisać:

> „Linux jest ciężki, a Bursztyn lekki.”

bez zdefiniowania dystrybucji i pomiarów.

Nie należy pisać:

> „Bursztyn ma lepsze aktualizacje od Linux.”

bo Linux nie definiuje jednego systemu aktualizacji.

Można natomiast rzetelnie napisać:

> Bursztyn OS jest własnym, eksperymentalnym systemem bare-metal, który implementuje już wiele funkcji występujących w dojrzałych systemach — procesy Ring 3, własne ABI, filesystem, sieć, GUI, instalator i A/B updates — ale pod względem wielordzeniowości, sterowników, pamięci wirtualnej, GPU, bezpieczeństwa warstwowego i ekosystemu pozostaje daleko za współczesnym Linux.

---

# 40. Aktualny status jakości Bursztyn OS

Projekt posiada dużą liczbę testów hostowych i QEMU.

Jednocześnie trzeba odróżniać:

- PASS konkretnego audytu,
- PASS konkretnego komponentu,
- globalny stan całego systemu.

W ostatnim przekazanym pełnym logu aplikacji testowej pozostawały jeszcze błędy w kilku grupach BursztynUI/BWS.

Trwają prace stabilizacyjne.

Najuczciwszy aktualny opis:

> **Bursztyn OS jest zaawansowanym eksperymentalnym systemem operacyjnym z własnym kernelem, Ring 3, BWS, BSP2, siecią, BursztynUI, instalatorem i aktualizacjami A/B. Nie jest jeszcze systemem produkcyjnym ani funkcjonalnym odpowiednikiem współczesnej dystrybucji Linux.**

---

# 41. Ocena końcowa

Linux i Bursztyn OS mają pewne podobieństwa na poziomie bardzo ogólnym:

- monolityczne jądro,
- separacja kernel/user,
- syscall ABI,
- virtual memory,
- sterowniki,
- filesystem,
- networking.

Jednak skala jest zupełnie inna.

## Linux

Linux 7.2 jest:

- wieloarchitekturowy,
- wielordzeniowy,
- wysoko zoptymalizowany,
- używany w produkcji,
- wyposażony w ogromny zestaw sterowników,
- fundamentem serwerów, chmury, HPC, urządzeń embedded i desktopów.

## Bursztyn OS

Bursztyn jest:

- własnym systemem x86-64,
- aktywnie rozwijanym,
- architektonicznie spójnym,
- znacznie mniejszym,
- eksperymentalnym,
- bardzo wartościowym edukacyjnie,
- coraz bardziej funkcjonalnym.

Najbardziej rzetelny wniosek:

> **Bursztyn OS nie jest konkurentem Linux pod względem dojrzałości, kompatybilności sprzętowej ani ekosystemu. Jest natomiast rzeczywistą, niezależną implementacją systemu operacyjnego, która osiągnęła etap obejmujący własny kernel, userspace, ABI, filesystem, networking, GUI i bezpieczny mechanizm aktualizacji. Wartość projektu leży w samodzielnym zaprojektowaniu tych warstw i możliwości eksperymentowania z własnymi rozwiązaniami, a nie w próbie powielenia kilkudziesięciu lat rozwoju Linux.**

---

# 42. Źródła

## 42.1. Bursztyn OS — dokumentacja projektu

Przy opracowaniu wykorzystano przede wszystkim:

- `01_wstep_i_filozofia.md`
- `02_architektura_systemu.md`
- `03_proces_rozruchu.md`
- `04_zarzadzanie_sprzetem.md`
- `04_audyt_gui_schedulera.md`
- `05_hpet_i_pacing.md`
- `05_bursztynowy_system_plikow.md`
- `06_wywolania_systemowe.md` — historyczny w części numeracji BWS
- `07_ekosystem_i_formaty.md`
- `08_bursztynowy_slownik_i_architektura.md`
- `09_tryb_graficzny.md`
- `10_siec.md`
- `11_audyt_bws_nowoczesne.md`
- `instalator_audyt.md`
- `ustawienia_ekranu_audyt.md`
- `aktualizacja-systemu.md`
- ostatnie przekazane logi `Testy Bursztyn OS`
- ostatnie raporty dotyczące popupów, `DialogPliku` i generacji framebufferu.

Przy rozbieżnościach priorytet nadano nowszym audytom i ostatnim wynikom runtime.

## 42.2. Linux — źródła upstream

### Stan wydania
- kernel.org — The Linux Kernel Archives  
  https://www.kernel.org/

### Scheduler
- Linux kernel documentation — EEVDF Scheduler  
  https://docs.kernel.org/scheduler/sched-eevdf.html
- Linux kernel documentation — CFS Scheduler  
  https://docs.kernel.org/scheduler/sched-design-CFS.html

### ABI i syscalle
- Linux kernel documentation — Adding a New System Call  
  https://docs.kernel.org/process/adding-syscalls.html
- Linux kernel documentation — Linux ABI description  
  https://docs.kernel.org/admin-guide/abi.html
- Linux kernel documentation — The Linux Kernel Driver Interface  
  https://docs.kernel.org/process/stable-api-nonsense.html

### Pamięć
- Linux kernel documentation — zswap  
  https://docs.kernel.org/admin-guide/mm/zswap.html

### cgroup i namespaces
- Linux kernel documentation — Control Group v2  
  https://docs.kernel.org/admin-guide/cgroup-v2.html
- Linux kernel documentation — Namespaces  
  https://docs.kernel.org/admin-guide/namespaces/

### Bezpieczeństwo
- Linux kernel documentation — LSM  
  https://docs.kernel.org/admin-guide/LSM/
- Linux kernel documentation — Landlock  
  https://docs.kernel.org/userspace-api/landlock.html
- Linux kernel documentation — Seccomp BPF  
  https://docs.kernel.org/userspace-api/seccomp_filter.html
- Linux kernel documentation — Kernel module signing  
  https://docs.kernel.org/admin-guide/module-signing.html
- Linux kernel documentation — fs-verity  
  https://docs.kernel.org/filesystems/fsverity.html

### Filesystems
- Linux kernel documentation — ext4  
  https://docs.kernel.org/filesystems/ext4/
- Linux kernel documentation — XFS  
  https://docs.kernel.org/filesystems/xfs/
- Linux kernel documentation — Btrfs  
  https://docs.kernel.org/filesystems/btrfs.html
- Linux kernel documentation — OverlayFS  
  https://docs.kernel.org/filesystems/overlayfs.html

### Sieć
- Linux kernel documentation — Networking  
  https://docs.kernel.org/networking/
- Linux kernel documentation — IPv6  
  https://docs.kernel.org/networking/ipv6.html
- Linux kernel documentation — IP sysctl / TCP  
  https://docs.kernel.org/networking/ip-sysctl.html
- Linux kernel documentation — io_uring zero-copy receive  
  https://docs.kernel.org/networking/iou-zcrx.html

### Grafika
- Linux kernel documentation — DRM GPU guide  
  https://docs.kernel.org/gpu/
- Linux kernel documentation — Kernel Mode Setting  
  https://docs.kernel.org/gpu/drm-kms.html
- Mesa documentation  
  https://docs.mesa3d.org/
- Wayland documentation  
  https://wayland.freedesktop.org/docs/

### Audio
- Linux kernel documentation — Sound / ALSA  
  https://docs.kernel.org/sound/

### Power management
- Linux kernel documentation — Power Management  
  https://docs.kernel.org/admin-guide/pm/

### eBPF
- Linux kernel documentation — BPF  
  https://docs.kernel.org/bpf/

---

# 43. Zalecenie dla dokumentacji Bursztyn OS

Aby kolejne porównania były jeszcze dokładniejsze, warto utrzymywać jeden nadrzędny dokument:

`docs/STAN_SYSTEMU_CURRENT.md`

z automatycznie lub półautomatycznie aktualizowanymi polami:

- commit,
- data,
- maksymalny numer BWS,
- liczba procesów/włókien,
- model UP/SMP,
- obsługiwane architektury,
- limity pamięci,
- stan BSP2,
- lista sterowników,
- obsługiwane urządzenia blokowe,
- protokoły sieciowe,
- liczba jednoczesnych socketów,
- backendy grafiki,
- stan akceleracji GPU,
- wersja BursztynUI,
- liczba publicznych elementów,
- status Secure Boot,
- ostatni pełny wynik testów,
- znane regresje,
- testy QEMU,
- testy na fizycznym sprzęcie.

Taki plik powinien być nadrzędnym źródłem faktów dla dokumentacji, postów i przyszłych porównań.
