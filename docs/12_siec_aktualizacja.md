# 10. Sieć w Bursztyn OS

**Stan dokumentu:** aktualizacja 2026-09-18  
**Zakres:** Intel E1000, Ethernet II, ARP, IPv4, ICMP, UDP, DHCPv4, DNS, TCP, HTTP, HTTPS/TLS, BWS sieciowe i aktualne ograniczenia stosu  
**Status:** funkcjonalny natywny stos IPv4 rozwijany w Ring 0; część transportowa pozostaje ograniczona do jednej aktywnej sesji TCP

---

## 10.1 Aktualna architektura sieci

Bursztyn OS posiada własny stos sieciowy dla IPv4 z własnym sterownikiem Intel E1000.

Warstwy można przedstawić następująco:

```text
aplikacja Ring 3
      ↓
BWS / SDK
      ↓
PZB + PRAWO_SIEC
      ↓
DNS / HTTP / HTTPS / socket facade
      ↓
TCP / UDP / ICMP
      ↓
IPv4
      ↓
ARP
      ↓
Ethernet II
      ↓
Intel E1000
      ↓
sieć
```

Aktualnie zaimplementowane są:

```text
Ethernet II
ARP
IPv4
ICMP Echo
UDP
DHCPv4
DNS A
TCP
HTTP/1.0
HTTPS/TLS
```

Stos jest natywną częścią Bursztyn OS.

Nie korzysta z hostowego stosu sieciowego Linuxa, BSD ani Windows.

---

# 10.2 Ważna korekta statusu

Starszy dokument opisywał stos jako:

> „w pełni responsywny” i „pełnoprawny TCP/IP”.

Takie określenie jest obecnie zbyt szerokie.

Aktualny stos działa i obsługuje realną komunikację IPv4, ale świadomie nie implementuje jeszcze pełnego zestawu mechanizmów dojrzałego stosu TCP/IP.

Najważniejsze ograniczenia:

```text
brak IPv6
brak IPv4 fragmentation/reassembly
jedna aktywna sesja TCP w całym systemie
brak pełnego TCP retransmission/RTO
brak pełnego congestion control
brak SACK
brak readiness event dla socketów
E1000 działa obecnie pollingowo
DHCP nie wykonuje jeszcze renew/rebind
ARP cache nie posiada jeszcze czasowego TTL
```

Dokumentacja powinna więc określać stos jako:

> **funkcjonalny natywny stos IPv4 w aktywnym rozwoju.**

---

# 10.3 Intel E1000

Aktualny sterownik sieciowy jest przeznaczony dla urządzenia Intel E1000 zgodnego z testowanym modelem:

```text
PCI Vendor: 8086
Device:     100E
```

czyli typowym:

```text
Intel 82540EM / PRO/1000
```

Sterownik:

- wykrywa urządzenie przez PCI,
- obsługuje funkcje urządzeń multifunction,
- mapuje BAR MMIO,
- włącza PCI Memory Space,
- włącza Bus Mastering,
- przygotowuje ringi DMA RX/TX,
- alokuje deskryptory i bufory przez PMM,
- używa HHDM do kontrolowanego dostępu CPU do ramek DMA.

---

# 10.4 E1000 pracuje obecnie pollingowo

Istotna korekta:

> aktualny sterownik E1000 **nie używa jeszcze MSI/MSI-X ani IRQ do odbioru pakietów**.

Polling jest obecnie celowym rozwiązaniem.

Kod regularnie wykonuje:

```text
e1000_obsluz_odbior()
```

w oczekujących operacjach sieciowych.

Sterownik maskuje przerwania karty i korzysta z pollingu RX/TX.

To upraszcza aktualną architekturę, ale ma konsekwencje:

- większe zużycie CPU podczas oczekiwania,
- synchroniczne pętle w części operacji,
- słabszą skalowalność,
- potencjalne zatrzymania GUI przy długich operacjach Ring 0.

Docelowo należy rozważyć:

```text
IRQ/MSI-X
→ krótki ISR
→ kolejka pakietów
→ worker / network task
```

---

# 10.5 Bufory DMA

Ringi i bufory RX/TX są zasobami DMA.

Nie powinny być traktowane jak zwykłe wskaźniki aplikacji.

Wymagane zasady:

```text
PMM ownership
właściwe alignment
fizyczny adres dla urządzenia
kontrolowany HHDM dla CPU
bounded descriptors
pełny cleanup przy błędzie inicjalizacji
```

Sterownik nie przekazuje fizycznego adresu DMA do Ring 3.

---

# 10.6 Ethernet II

Najniższa warstwa protokołu obsługuje ramki Ethernet II.

Aktualnie rozpoznawane są między innymi:

```text
EtherType IPv4 = 0x0800
EtherType ARP  = 0x0806
```

Parser nie powinien dereferencjonować nagłówka przed sprawdzeniem minimalnej długości ramki.

Aktualny kod sieciowy jawnie wykonuje bounds-checking przed użyciem danych typu wire-format.

---

# 10.7 Byte order

Bursztyn OS działa obecnie na:

```text
x86-64 little-endian
```

natomiast protokoły sieciowe używają network byte order.

Aktualne helpery:

```text
bws_htons
bws_ntohs
bws_htonl
bws_ntohl
```

są funkcjami, a nie niebezpiecznymi makrami wielokrotnie rozwijającymi argument.

Zachowane są również legacy nazwy:

```text
HTONS
NTOHS
HTONL
NTOHL
```

dla zgodności istniejącego kodu.

---

# 10.8 ARP

Stos implementuje ARP dla Ethernet/IPv4.

Funkcje obejmują:

- wysłanie ARP Request,
- odbiór ARP Reply,
- zapamiętywanie mapowania IPv4 → MAC,
- używanie cache przy kolejnych wysyłkach.

Aktualna tablica ARP posiada:

```text
16 wpisów
```

i mechanizm zastępowania wpisów.

---

# 10.9 Walidacja ARP

ARP jest wejściem niezaufanym.

Aktualny parser sprawdza między innymi:

- długość ramki,
- typ sprzętu/protokołu,
- rozmiary adresów,
- sensowność adresu IPv4,
- sensowność MAC,
- poprawność relacji Ethernet/ARP.

Cache nie powinien przyjmować dowolnej zawartości odebranej z sieci.

---

# 10.10 Ograniczenie ARP cache

Aktualny ARP cache nie posiada jeszcze pełnej polityki czasowego TTL.

Oznacza to, że obecny mechanizm nie jest jeszcze odpowiednikiem dojrzałego cache sąsiedztwa z:

- aging,
- reachability state,
- dynamicznym odświeżaniem wpisów,
- wykrywaniem konfliktów.

Jest to jeden z punktów przyszłego rozwoju.

---

# 10.11 IPv4

Aktualny stos obsługuje:

```text
IPv4
```

Nagłówek jest walidowany pod kątem:

- minimalnej długości,
- IHL,
- całkowitej długości,
- checksum,
- protokołu,
- adresów,
- granic otrzymanej ramki.

---

# 10.12 Brak fragmentacji IPv4

Aktualna implementacja **nie wykonuje reassembly fragmentów IPv4**.

Pakiety fragmentowane są odrzucane zamiast częściowo interpretowane.

To świadoma polityka fail-closed.

Nie należy pisać, że Bursztyn OS obsługuje pełne IPv4, jeśli przez „pełne” rozumie się również fragmentację/reassembly.

---

# 10.13 Brak IPv6

Aktualny publiczny stos sieciowy jest:

```text
IPv4-only
```

BWS socket facade również używa dokładnie:

```text
4 bajtów IPv4
```

IPv6 pozostaje roadmapą.

---

# 10.14 ICMP Echo

Bursztyn OS posiada własną obsługę ICMP Echo.

BWS:

```text
11 — ping
```

pozwala aplikacji z odpowiednimi prawami uruchomić test ICMP.

Stos potrafi:

```text
Echo Request
→ IPv4
→ ARP/routing
→ E1000

Echo Reply
→ parser
→ walidacja
→ wynik/log
```

Aktualny timeout ICMP wynosi w kodzie:

```text
4000 ms
```

---

# 10.15 Routing IPv4

Stos używa:

- własnego adresu IPv4,
- maski podsieci,
- bramy domyślnej.

Przed wysłaniem pakietu określa, czy cel jest:

```text
w tej samej podsieci
```

czy:

```text
poza lokalną podsiecią
```

Dla hosta zewnętrznego Ethernet destination MAC jest rozwiązywany dla:

```text
bramy domyślnej
```

a nie bezpośrednio dla zdalnego hosta Internetu.

---

# 10.16 UDP

UDP jest wykorzystywany między innymi przez:

- DHCPv4,
- DNS.

Parser sprawdza:

- minimalny rozmiar nagłówka,
- porty,
- długość UDP,
- granice pakietu.

UDP jest aktualnie warstwą wewnętrzną stosu, a nie kompletnym publicznym API datagram sockets dla aplikacji Ring 3.

---

# 10.17 DHCPv4

Aktualny klient DHCP działa bez statycznego adresu IP w kodzie.

Typowy przebieg:

```text
0.0.0.0
  ↓
DHCP DISCOVER
  ↓
DHCP OFFER
  ↓
DHCP REQUEST
  ↓
DHCP ACK
  ↓
konfiguracja IPv4
```

Po sukcesie zapisywane są między innymi:

- adres IPv4,
- maska podsieci,
- brama,
- serwer DNS.

Porty:

```text
serwer DHCP = 67
klient DHCP = 68
```

---

# 10.18 Walidacja DHCP

Parser DHCP nie zakłada, że cały pakiet ma rozmiar lokalnej struktury C.

Korzysta z rzeczywistego rozmiaru UDP i waliduje opcje TLV przed odczytem.

Obsługiwane są między innymi opcje:

```text
Subnet Mask
Router
DNS
Requested IP
Message Type
Server Identifier
Parameter Request List
```

Aktualny timeout etapu DHCP w kodzie wynosi:

```text
8000 ms
```

---

# 10.19 Ograniczenie DHCP

Aktualny klient uzyskuje konfigurację początkową, ale nie posiada jeszcze kompletnej obsługi:

```text
lease renewal
rebind
pełnego lifecycle dzierżawy
```

Dlatego DHCP nie należy jeszcze przedstawiać jako kompletnej implementacji RFC dla długotrwałej pracy systemu.

---

# 10.20 DNS

Aktualny system posiada dwa historyczne poziomy API:

```text
BWS 12 — legacy DNS
BWS 28 — aktualniejsze DNS
```

Nowe aplikacje powinny preferować nowsze wrappery SDK.

Resolver obsługuje obecnie rekordy:

```text
DNS A
```

czyli tłumaczenie nazwy na IPv4.

---

# 10.21 Bezpieczeństwo DNS

Aktualny parser DNS weryfikuje między innymi:

- długość domeny,
- maks. 63 B na label,
- maks. 253 B całej nazwy,
- transaction ID,
- port źródłowy/docelowy,
- oczekiwany serwer,
- granice pakietu,
- compressed name pointers.

Maksymalny klasyczny bufor DNS w aktualnym kodzie:

```text
512 B
```

Timeout zapytania:

```text
4000 ms
```

---

# 10.22 TCP — aktualny model

Bursztyn OS posiada własny transport TCP.

Obsługiwane elementy obejmują:

- SYN,
- SYN-ACK,
- ACK,
- PSH,
- FIN,
- RST,
- numery sekwencyjne,
- numery potwierdzeń,
- checksum z pseudo-headerem,
- handshake,
- odbiór payload,
- kontrolowane zamknięcie.

Aktualne:

```text
TCP window = 8192
MSS wysyłania = 1400
```

---

# 10.23 Jedna sesja TCP

Najważniejsze aktualne ograniczenie:

> **bazowy stos `siec.cpp` utrzymuje jedną aktywną sesję TCP w całym systemie.**

Stan jest jawnie serializowany przez:

```text
tcp_sesja_zajeta
```

Jeśli jedna warstwa posiada aktywny TCP, druga operacja może zostać odrzucona.

Dotyczy to interakcji pomiędzy:

- legacy HTTP/HTTPS,
- TLS,
- nowym socket facade.

Nie należy więc nazywać obecnej implementacji pełnym wielogniazdowym stosem TCP.

---

# 10.24 Ograniczenia niezawodności TCP

Aktualny kod źródłowy wprost zaznacza brak pełnego:

- timerowego retransmission/RTO,
- congestion control,
- SACK,
- pełnej współbieżności sesji.

To oznacza, że handshake i podstawowy transfer działają, ale implementacja nie jest jeszcze produkcyjnym odpowiednikiem stosu TCP z Linux/BSD.

---

# 10.25 Bufor strumienia TCP

Aktualny wewnętrzny bufor sesji socketowej wynosi:

```text
64 KiB
```

Odbiór ma statusy rozróżniające:

```text
> 0 — odebrano dane
  0 — EOF
 -1 — błąd
 -2 — obecnie brak danych, połączenie nadal otwarte
```

Jest to pollingowa semantyka używana także przez TLS.

---

# 10.26 Timeout połączenia TCP

Aktualny kod używa dla zestawiania TCP:

```text
5000 ms
```

jako timeout connect/handshake warstwy transportowej.

Timeouty są oparte na monotonicznym czasie systemowym, a nie na przypadkowej liczbie iteracji pętli.

---

# 10.27 HTTP

Bursztyn OS posiada własnego klienta:

```text
HTTP/1.0
```

Aktualne żądanie ma postać zbliżoną do:

```http
GET /sciezka HTTP/1.0
Host: example.com
Connection: close
User-Agent: BursztynOS/0.1
Accept: */*
```

Domena i ścieżka są walidowane przed zbudowaniem requestu.

Znaki:

```text
\r
\n
```

są odrzucane w polach wejściowych, co ogranicza możliwość wstrzyknięcia dodatkowych nagłówków przez parametry aplikacji.

---

# 10.28 Korekta: HTTP zwraca surową odpowiedź

Starszy dokument mówił, że parser:

> „odcina nagłówki odpowiedzi serwera i pozostawia czyste dane pliku”.

Aktualna funkcja:

```text
kernel_siec_pobierz_http()
```

pobiera **surowe bajty odpowiedzi HTTP/1.0** do bufora.

Warstwa nie powinna więc być dokumentowana jako pełny parser HTTP body.

Jeżeli aplikacja potrzebuje:

- kodu statusu,
- nagłówków,
- `Content-Length`,
- body,

powinna użyć właściwej warstwy parsera HTTP.

---

# 10.29 Timeout HTTP

Aktualny timeout operacji HTTP wynosi:

```text
12000 ms
```

Jeśli bufor odpowiedzi zostanie przepełniony, transfer jest odrzucany jako ucięty.

Kod nie uznaje cichej truncation za sukces.

---

# 10.30 Historyczne BWS 12–13

BWS:

```text
12 — legacy DNS
13 — legacy HTTP do pliku
```

pozostają częścią zamrożonego ABI.

BWS 13 wymaga:

```text
PRAWO_SIEC
+
PRAWO_PLIKI_ZAPISZ
```

oraz sprawdzenia, czy proces może modyfikować wskazaną ścieżkę BSP2.

Pobrana odpowiedź jest zapisywana przez filesystem.

Ze względu na zachowanie ABI nie należy zmieniać numeru lub znaczenia BWS 13, ale nowe aplikacje powinny preferować nowsze API, jeśli nie potrzebują zachowania legacy.

---

# 10.31 Nowsze BWS DNS / HTTP / HTTPS

Późniejszy zakres ABI zawiera:

```text
28 — DNS
29 — HTTP
30 — HTTPS
31 — TLS certificate status
```

BWS 29 i 30 zwracają dane do kontrolowanego bufora Ring 3.

Jądro:

- kopiuje IP/domenę/ścieżkę z user space,
- waliduje zakres docelowy,
- alokuje tymczasowy bufor kernela,
- wykonuje operację sieciową,
- dopiero po sukcesie kopiuje wynik do Ring 3.

Wymagane jest:

```text
PRAWO_SIEC
```

---

# 10.32 Bezpieczeństwo buforów Ring 3

Sieć jest granicą bezpieczeństwa zarówno od strony:

```text
Internet → kernel
```

jak i:

```text
Ring 3 → kernel
```

Aktualne BWS używają checked user-copy.

Przed skutkiem ubocznym walidowane są m.in.:

- wskaźniki,
- długości,
- zakres zapisu,
- domena,
- ścieżka HTTP,
- adres IPv4.

Dane odebrane z sieci również nie są traktowane jako zaufane.

---

# 10.33 HTTPS/TLS

HTTPS jest zaimplementowane w osobnej warstwie:

```text
tls.cpp
```

i korzysta z istniejącego strumienia TCP:

```text
tcp_gniazdo_polacz
tcp_gniazdo_wyslij
tcp_gniazdo_odbierz
tcp_gniazdo_otwarte
tcp_gniazdo_zamknij
```

Warstwa TLS nie implementuje kryptografii od zera.

Aktualny source tree zawiera:

```text
mbedTLS 2.28.8
```

z portem dostosowanym do środowiska bare-metal Bursztyn OS.

---

# 10.34 Port mbedTLS

Bursztyn OS posiada własną warstwę integracyjną dla mbedTLS.

Port dostarcza m.in.:

- pamięć dla biblioteki,
- funkcje platformowe,
- minimalne funkcje libc wymagane przez bibliotekę,
- źródło entropii,
- integrację z własnym transportem TCP.

Kod mbedTLS znajduje się w:

```text
biblioteki/mbedtls/
```

a warstwa integracyjna między innymi w:

```text
mbedtls_port.cpp
tls.cpp
```

---

# 10.35 Entropia TLS

mbedTLS korzysta z hooka:

```text
bursztyn_krypto_wypelnij_losowe(...)
```

Komentarz w warstwie portu jawnie wymaga, aby było to:

```text
CSPRNG / bezpieczne źródło entropii
```

i ostrzega przed używaniem jako jedynego źródła:

- stałego seeda,
- prostego LCG,
- samego RDTSC,
- samego RTC.

Jest to krytyczny element bezpieczeństwa TLS.

---

# 10.36 Weryfikacja certyfikatów

Aktualny HTTPS używa:

```text
MBEDTLS_SSL_VERIFY_REQUIRED
```

Nie ma automatycznego fallbacku do:

```text
VERIFY_NONE
```

Nazwa hosta jest przekazywana do:

```text
mbedtls_ssl_set_hostname()
```

co zapewnia:

- SNI,
- walidację nazwy certyfikatu.

Po handshake kernel sprawdza:

```text
mbedtls_ssl_get_verify_result() == 0
```

dopiero wtedy oznaczając certyfikat jako zaufany.

---

# 10.37 Magazyn CA

TLS posiada mechanizm magazynu zaufanych CA.

Hook:

```cpp
bursztyn_tls_pobierz_magazyn_ca(...)
```

może dostarczyć:

- jeden lub wiele certyfikatów PEM,
- pojedynczy certyfikat DER.

Brak magazynu CA powinien powodować odmowę HTTPS.

W aktualnym source tree znajduje się również jawny minimalny fallback zawierający publiczny:

```text
ISRG Root X1
```

Nie jest to certyfikat konkretnego serwera i nie omija `VERIFY_REQUIRED`.

Docelowo magazyn CA powinien być osobnym, kontrolowanym komponentem systemu i podlegać bezpiecznej aktualizacji.

---

# 10.38 Timeouty TLS

Aktualna warstwa TLS używa deadline opartych o monotoniczny czas.

W kodzie znajdują się:

```text
TLS handshake timeout = 15000 ms
TLS I/O timeout       = 15000 ms
TLS close timeout     = 1000 ms
```

Pętle:

```text
WANT_READ
WANT_WRITE
```

mają ograniczony czas.

Nie są nieskończonym pollingiem.

---

# 10.39 Clean TLS shutdown

Warstwa HTTPS nie uznaje każdego zamknięcia TCP za poprawny koniec bezpiecznego transferu.

Brak poprawnego:

```text
TLS close_notify
```

jest traktowany jako potencjalnie ucięta odpowiedź.

Wynik jest publikowany dopiero po spełnieniu warunków obejmujących:

- poprawny certyfikat,
- zakończony handshake,
- wysłanie całego requestu,
- brak overflow bufora,
- poprawny koniec TLS.

---

# 10.40 BWS 31

BWS:

```text
31 — TLS certificate status
```

pozwala aplikacji odczytać status weryfikacji certyfikatu ostatniej odpowiedniej sesji HTTPS.

Nie powinien być interpretowany jako ogólny systemowy magazyn certyfikatów lub API zarządzania PKI.

---

# 10.41 Nowe sockety BWS 79–83

Nowoczesne rozszerzenie BWS dodało owned socket facade:

```text
79 — socket create
80 — connect
81 — send
82 — recv
83 — close
```

Aktualny typ:

```text
TCP/IPv4
```

Każdy publiczny socket posiada:

- właściciela PID,
- uchwyt z generacją,
- kontrolę `PRAWO_SIEC`,
- walidację buforów Ring 3,
- cleanup przy zakończeniu procesu.

---

# 10.42 Limity socket facade

Aktualny limit pojedynczej operacji send/recv w BWS wynosi:

```text
64 KiB
```

Port:

```text
0
```

jest odrzucany.

Dozwolony zakres:

```text
1..65535
```

Stale handle i uchwyt obcego właściciela są odrzucane.

---

# 10.43 Socket facade a jedna sesja TCP

Ważne:

> BWS 79–83 tworzą nowoczesny **model własności i uchwytów**, ale nie zmieniają jeszcze bazowego silnika TCP w wielosesyjny stos.

Pod spodem nadal istnieje:

```text
jedna globalna sesja TCP
```

Dlatego:

```text
wiele handle w API
≠
wiele równoległych połączeń transportowych
```

Aktualny wrapper serializuje dostęp do bazowego stosu.

---

# 10.44 Brak readiness event

Nowszy IPC posiada event gotowości, ale socket facade nie posiada jeszcze kompletnego mechanizmu:

```text
readable
writable
error
hangup
```

dla wielu socketów.

Brakuje odpowiednika:

```text
select
poll
epoll
kqueue
IOCP
```

Nie należy więc opisywać BWS socketów jako kompletnego nowoczesnego I/O API.

---

# 10.45 Legacy HTTP kontra socket facade

Gdy nowy socket facade rezerwuje bazowy stos TCP, legacy HTTP/HTTPS nie powinno równocześnie próbować przejąć tej samej pojedynczej sesji.

Dispatcher sprawdza stan rezerwacji.

To zapobiega dwóm niezależnym API manipulującym jednocześnie tym samym globalnym stanem transportu.

---

# 10.46 Synchroniczność sieci

DNS, HTTP, HTTPS i TLS mają obecnie istotne ścieżki synchroniczne.

W praktyce kod wykonuje pętle typu:

```text
do deadline:
    e1000_obsluz_odbior()
    pause
```

Podczas długiego TLS handshake może to oznaczać zauważalne zatrzymanie aplikacji/GUI.

Timer nadal obsługuje IRQ/EOI, ale scheduler nie wykonuje arbitralnego context switcha w środku zwykłej ramy `SYSCALL`.

---

# 10.47 Sieć a scheduler

Aktualnego ograniczenia nie należy rozwiązywać przez wymuszenie preempcji w losowym miejscu kodu TLS.

Rama:

```text
SYSCALL
```

nie jest tym samym co zwykła rama ISR procesu Ring 3.

Docelowy model powinien przejść w kierunku:

```text
Ring 3 request
      ↓
network request object
      ↓
WAITING_IO
      ↓
network worker/state machine
      ↓
IRQ/poll completion
      ↓
event
      ↓
wake process
```

Pełny lifecycle async request/cancel/completion nie istnieje jeszcze.

---

# 10.48 PZB i `PRAWO_SIEC`

Dostęp do sieci jest kontrolowany przez:

```text
PRAWO_SIEC
```

Nowoczesne socket BWS sprawdzają to prawo przy operacjach.

Wrażliwe systemowe komponenty mogą dodatkowo łączyć je z innymi prawami.

Przykładowo operacja pobierająca dane i zapisująca je do BSP2 może wymagać:

```text
PRAWO_SIEC
+
PRAWO_PLIKI_ZAPISZ
```

Sam manifest aplikacji nie może skutecznie nadać sobie dowolnych praw bez polityki jądra.

---

# 10.49 Bezpieczeństwo parserów sieciowych

Dane z sieci są niezaufane.

Aktualny `siec.cpp` wprowadza kilka zasad:

```text
length before dereference
bounded packet parsing
validated wire-format fields
checked DNS compression
checked DHCP options
checked TCP header length
checked TCP/IP ports
checked checksum
bounded receive buffers
reject unsupported fragmentation
```

Packed struct nie jest traktowany jako pozwolenie na ślepe rzutowanie dowolnej ramki.

---

# 10.50 Maksymalne rozmiary i bounded buffers

Aktualny kod stosuje jawne limity, między innymi:

```text
Ethernet frame      ≈ 1514 B
IPv4 packet         ≈ 1500 B
DNS packet          = 512 B
DNS name            = 253 B
DNS label           = 63 B
DHCP packet helper  = 548 B
HTTP request        = 4096 B
TCP receive buffer  = 64 KiB
socket BWS I/O      = 64 KiB / call
TLS BIO fragment    = 16 KiB
TLS HTTP request    = 4096 B
TLS domain          = 253 B
TLS HTTP path       = 2047 B
```

Takie limity są częścią strategii bezpieczeństwa bare-metal i powinny być testowane na wartościach:

```text
0
MAX
MAX + 1
```

---

# 10.51 Powłoka Bursztynowa

Powłoka Bursztynowa działa jako aplikacja Ring 3.

Może udostępniać użytkownikowi operacje sieciowe, takie jak:

```text
ping
DNS
HTTP/HTTPS
```

Tekst interfejsu wykorzystuje UTF-8.

Sieć nie jest jednak częścią renderer'a terminala. Powłoka jest klientem BWS, a właściwy stos protokołów działa po stronie systemowej.

---

# 10.52 Aplikacje graficzne i sieć

BursztynUI nie potrzebuje specjalnego „sieciowego GUI syscall”.

Aplikacja graficzna może używać:

```text
BursztynUI
+
BWS socket/network
```

niezależnie.

Przykładem może być Husarz, eksperymentalna aplikacja WWW.

Ograniczenie jednego TCP i synchronicznych ścieżek oznacza jednak, że pełna nowoczesna przeglądarka wielopołączeniowa nie jest jeszcze możliwa na obecnym modelu transportowym.

---

# 10.53 Aktualizacje systemowe i sieć

Podsystem aktualizacji A/B korzysta z praw sieciowych i własnego kontrolowanego API.

BWS OTA:

```text
84 — status
85 — polecenie
```

nie zastępują ogólnych socketów.

Aktualizacja wymaga własnej polityki PZB i praw, ponieważ pobranie pakietu kończy się operacjami krytycznymi dla systemu.

---

# 10.54 IPv6 — roadmapa

IPv6 nie jest obecnie zaimplementowane.

Przyszłe wsparcie powinno być projektowane jako osobna warstwa, a nie przez rozciąganie struktur 4-bajtowego IPv4.

Potencjalny zakres:

```text
EtherType IPv6
IPv6 base header
ICMPv6
Neighbor Discovery
SLAAC / DHCPv6
DNS AAAA
TCP/UDP over IPv6
socket address ABI v2
```

Zmiana będzie wymagała wersjonowania publicznego ABI socketów.

---

# 10.55 Przyszły wielosesyjny TCP

Najważniejszym krokiem rozwojowym sieci jest odejście od jednego globalnego stanu:

```text
tcp_nasz_seq
tcp_nasz_ack
tcp_nasz_port
tcp_cel_port
tcp_cel_ip
tcp_sesja_zajeta
```

w kierunku obiektów:

```text
TcpConnection
{
    state
    local_ip
    remote_ip
    local_port
    remote_port
    snd_una
    snd_nxt
    rcv_nxt
    timers
    rx_queue
    tx_queue
}
```

Każdy BWS socket handle powinien docelowo wskazywać niezależny obiekt transportu.

---

# 10.56 Przyszły worker sieciowy

Docelowo warto rozdzielić:

```text
sterownik RX/TX
transport
API aplikacji
```

poprzez kolejki.

Przykład:

```text
E1000 IRQ
   ↓
RX queue
   ↓
network worker
   ↓
Ethernet/ARP/IP/TCP
   ↓
socket receive queue
   ↓
readiness event
   ↓
Ring 3
```

Pozwoli to uniknąć długich pollingowych syscalli.

---

# 10.57 Testy E1000

Minimalny zestaw testów powinien obejmować:

```text
PCI device present
PCI device absent
bad BAR
MMIO mapping failure
DMA ring allocation failure
RX buffer allocation failure
TX buffer allocation failure
link unavailable
oversized frame
descriptor ownership
cleanup after partial init
```

Niepowodzenie inicjalizacji nie może pozostawiać częściowo zaalokowanych ramek PMM.

---

# 10.58 Testy ARP / IPv4

Należy testować:

```text
valid ARP request/reply
invalid MAC
wrong EtherType
short packet
bad IPv4 checksum
bad IHL
total length < header
total length > received frame
fragmented IPv4 → reject
unsupported IP protocol
same-subnet route
gateway route
ARP timeout
```

---

# 10.59 Testy DHCP / DNS

### DHCP

```text
DISCOVER/OFFER/REQUEST/ACK
NAK
short option
bad TLV length
missing cookie
timeout
missing router
missing DNS
```

### DNS

```text
valid A response
wrong transaction ID
wrong source port
wrong server
compressed names
compression loop
label > 63
name > 253
truncated packet
zero answers
timeout
```

---

# 10.60 Testy TCP

Minimalne przypadki:

```text
SYN → SYN-ACK → ACK
RST during connect
connect timeout
valid payload
out-of-order sequence
wrong source IP
wrong source port
bad TCP checksum
short TCP header
FIN
buffer overflow
second simultaneous connection → controlled refusal
```

Przed uznaniem transportu za produkcyjny należy dodać testy:

```text
retransmission
packet loss
duplicate segment
reordering
zero window
RTO
congestion behavior
long transfers
```

---

# 10.61 Testy HTTP

Minimalne przypadki:

```text
valid HTTP/1.0 response
response larger than buffer
connection closes normally
RST
timeout
CRLF injection attempt in host/path
path without leading /
maximum valid host
maximum valid path
```

Aplikacja parsera HTTP powinna dodatkowo testować:

```text
status line
headers
Content-Length
body separation
malformed headers
```

ponieważ transport kernela zwraca obecnie surową odpowiedź.

---

# 10.62 Testy TLS

Minimalna macierz:

```text
valid trusted certificate
wrong hostname
untrusted CA
expired certificate
missing CA store
handshake timeout
WANT_READ/WANT_WRITE
buffer overflow
server without close_notify
RST
truncated TLS record
entropy failure
```

Szczególnie ważne:

```text
VERIFY_REQUIRED
```

nie może zostać osłabione w ścieżce błędu.

---

# 10.63 Testy BWS 79–83

Nowoczesny socket facade powinien być testowany dla:

```text
create
connect
send 0 B
send 64 KiB
send 64 KiB + 1 → reject
recv
EOF
no data
close
stale handle
wrong owner
missing PRAWO_SIEC
process death cleanup
```

Wymagane są również testy współzawodnictwa z legacy HTTP/HTTPS o jedną bazową sesję TCP.

---

# 10.64 Aktualny status bezpieczeństwa

Aktualna warstwa sieciowa posiada istotnie lepszy model bezpieczeństwa niż pierwsza wersja dokumentu:

- parsery są ograniczone rozmiarami,
- fragmentowane IPv4 jest odrzucane,
- DNS weryfikuje transaction context,
- DHCP sprawdza opcje,
- TCP waliduje transport,
- BWS sprawdzają user pointers,
- sockety mają owner PID i generacyjne uchwyty,
- HTTPS używa wymaganej weryfikacji X.509.

Nie oznacza to „absolutnego bezpieczeństwa”.

Stos nadal jest kodem Ring 0 przetwarzającym niezaufane dane z sieci i wymaga:

```text
fuzzingu
fault injection
testów granicznych
testów packet loss/reordering
audytu parserów
```

---

# 10.65 Najważniejsze korekty względem starego dokumentu

Stary dokument został zaktualizowany w następujących obszarach:

1. usunięto stwierdzenia o „pełnej niezależności sieciowej” i pełnoprawnym dojrzałym TCP;
2. dodano aktualny model E1000 z DMA i **pollingiem bez IRQ/MSI-X**;
3. dodano jawne ograniczenia IPv4;
4. dodano brak IPv6;
5. dodano brak fragmentacji/reassembly;
6. dodano brak DHCP renew/rebind;
7. dodano brak TTL ARP cache;
8. doprecyzowano parser DNS;
9. opisano pojedynczą globalną sesję TCP;
10. dodano brak pełnego retransmission/congestion control/SACK;
11. poprawiono HTTP — aktualny kernel zwraca surową odpowiedź, nie gwarantuje usunięcia nagłówków;
12. dodano BWS 28–31;
13. dodano HTTPS/TLS;
14. dodano mbedTLS 2.28.8;
15. dodano `VERIFY_REQUIRED`, SNI i walidację X.509;
16. dodano magazyn CA i ISRG Root X1;
17. dodano BWS socket 79–83;
18. dodano 64 KiB limit socket I/O;
19. dodano owner PID i generation handles;
20. opisano wpływ synchronicznych syscalli sieciowych/TLS na GUI;
21. dodano roadmapę async I/O, wielosesyjnego TCP i IPv6.

---

# 10.66 Podsumowanie

Aktualny stos sieciowy Bursztyn OS wygląda następująco:

```text
Intel E1000 (polling, DMA)
        ↓
Ethernet II
        ↓
ARP
        ↓
IPv4
        ├── ICMP Echo
        ├── UDP
        │   ├── DHCPv4
        │   └── DNS A
        └── TCP
            ├── HTTP/1.0
            ├── HTTPS / mbedTLS
            └── BWS socket facade
```

Najważniejsze mocne strony obecnej implementacji:

```text
własny sterownik E1000
własny stos IPv4
DHCP
DNS
ICMP
TCP
HTTP
HTTPS z weryfikacją certyfikatu
PZB / PRAWO_SIEC
owned socket handles
checked user-copy
bounded parsery
```

Najważniejsze dalsze prace:

```text
wiele równoległych TCP connections
pełne retransmission/RTO
congestion control
readiness events
async network I/O
E1000 IRQ/MSI-X
DHCP renew/rebind
ARP aging
IPv4 reassembly
IPv6
network fuzzing
```

---

## 10.67 Dokumenty i źródła powiązane

Najważniejsze źródła prawdy dla bieżącego stosu:

```text
siec.cpp
siec.h
e1000.cpp
e1000.h
tls.cpp
sterowniki/siec/sockets.h
bws_nowoczesne.cpp
bws_nowoczesne.h
bws_api.h
syscalls.cpp
biblioteki/mbedtls/
```

Dokumenty powiązane:

- `02_architektura_systemu.md` — PZB i Ring 0/Ring 3,
- `04_zarzadzanie_sprzetem.md` — PCI i E1000,
- `04_audyt_gui_schedulera.md` — wpływ synchronicznego TLS na scheduler,
- `05_hpet_i_pacing.md` — monotoniczne deadline,
- `06_wywolania_systemowe.md` — BWS 11–13, 28–31 i 79–83,
- `11_audyt_bws_nowoczesne.md` — socket facade i ograniczenie jednej sesji TCP.

Przy rozbieżnościach pierwszeństwo powinny mieć:

1. aktualny `siec.cpp` i `siec.h`,
2. aktualny `tls.cpp`,
3. aktualny `e1000.cpp`,
4. `bws_api.h` i `bws_nowoczesne.cpp`,
5. aktualne testy runtime,
6. najnowsze audyty,
7. starsza dokumentacja opisowa.
