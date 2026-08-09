# Terminal ręczny V3S — decyzje projektowe (hardware + firmware)

Dokument referencyjny dla oddzielnej płyty "terminala polowego" (battery-powered field
terminal) opartej o Allwinner V3S + AXP209, odróżnionej od obecnej płyty deweloperskiej
LicheePi Zero (bez PMIC), na której działa `firmware/CanSensorHub-v3s`. Kod aplikacji już
odwołuje się do tego pliku (`core/include/csh/config.h`, `core/include/csh/power_info.h`) —
te odwołania zostały zaktualizowane pod numerację sekcji użytą tutaj (patrz "Numeracja"
niżej).

**Historia tego dokumentu**: to scalenie dwóch wcześniejszych, równolegle prowadzonych
dokumentów:

1. `doc/TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md` (stan na 2026-07-23) — bardziej sformalizowany
   dokument decyzji, 10 sekcji 1:1 odpowiadających punktom brifu użytkownika, z analizą
   przydziału pinów/EINT/I2C i planem realizacji. Zachowany w archiwum jako
   `firmware/LicheePi-support/TERMINAL_RECZNY_DECYZJE_PROJEKTOWE_ARCHIWUM_hardware_2026-07-23_przed_polaczeniem.md`.
2. `firmware/LicheePi-support/TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md` (stan na 2026-07-19) —
   wcześniejszy notatnik pytanie-odpowiedź z fazy koncepcyjnej, obejmujący też tematy
   nieobecne w dokumencie (1): wybór AXP209 vs AXP203, szczegóły przycisku PWRON, RTC
   wbudowany w V3S vs zewnętrzny MCP79410, historia wyboru kontrolera CAN (MCP2515 →
   MCP25625 → CAN FD), migrację kernela 5.2.y → 6.12.95 LTS, schemat aktualizacji A/B
   (RAUC), porównanie panelu 5" vs 7". Zachowany w archiwum jako
   `firmware/LicheePi-support/TERMINAL_RECZNY_DECYZJE_PROJEKTOWE_ARCHIWUM_notatnik_2026-07-19_przed_polaczeniem.md`.

Scalono 2026-07-24. Numeracja sekcji 1-10 (patrz niżej) pochodzi z dokumentu (1) i została
zachowana bez zmian, bo odwołują się do niej po numerze zarówno kod firmware
(`config.h`/`power_info.h`, "point 1" = AXP209), jak i `doc/V3S_PINOUT_ROZPISKA.md` (sekcja
8 = Ethernet). Treści unikalne dla dokumentu (2) dodano jako nowe podsekcje tam, gdzie
tematycznie pasują, oraz jako osobne "Dodatki" na końcu, żeby nie zaburzyć tej numeracji.
Miejsca, w których oba dokumenty się rozjeżdżały, zebrano w sekcji 0 poniżej — do
świadomego uzgodnienia, nie rozstrzygnięte tu jednostronnie.

Stan na 2026-07-23 (bez zmian od poprzedniej wersji): schemat KiCad (`pcb/`) jest we
wczesnym, w dużej mierze pustym stanie (pojedynczy nieopisany symbol V3S, stub AXP209,
szkic CAN bez połączeń) — ten dokument **nie** opisuje stanu schematu, tylko decyzje
koncepcyjne do wdrożenia. Traktować jako specyfikację wejściową do rysowania schematu, nie
jako opis tego, co już narysowane.

## Numeracja

Sekcje 1-10 odpowiadają 1:1 dziesięciu punktom z briefu użytkownika (2026-07-23). Gdziekolwiek
w kodzie firmware pojawia się odwołanie "point 3" do AXP209 — to była numeracja robocza
sprzed powstania tego dokumentu; poprawiona na "point 1" (patrz commit wprowadzający ten
plik). Ta sama numeracja robocza ("3. AXP203 vs AXP209", "4. Zewnętrzny RTC", "5. CAN bus",
"6. Upgrade kernela", ...) jest też używana w notatniku 2026-07-19 cytowanym wyżej — to
inna, chronologiczna numeracja pytań z fazy koncepcyjnej, nieużywana już nigdzie w kodzie;
zachowana tylko wewnątrz cytowanych fragmentów niżej dla czytelności historii decyzji.

---

## 0. Rozbieżności między obydwoma dokumentami — rozstrzygnięte 2026-07-24

Poniższe punkty były miejscami, gdzie notatnik 2026-07-19 i dokument decyzji 2026-07-23
mówiły co innego (albo gdzie 2026-07-23 milczał na temat czegoś, co 2026-07-19 zostawił jako
jawnie otwarte). **Wszystkie trzy zostały rozstrzygnięte przez użytkownika 2026-07-24** —
historia obu wcześniejszych stanowisk zachowana niżej dla kontekstu, decyzja końcowa
wprowadzona też do właściwych sekcji dokumentu (1.2, 1.3, 5.1, 9) i do listy "Decyzje".

### 0.1 VBUS AXP209 / port USB debug — pełne odłączenie czy arbitraż przez PMIC?

- **Notatnik 2026-07-19 (pkt. 1)**: "VBUS zostaje wolne pod USB serwisowe/debug — oba
  źródła (24V i USB) mogą być podłączone jednocześnie, PMIC sam arbitrażuje (ACIN ma
  priorytet)" — czyli VBUS AXP209 **podłączony** do portu USB debug, PMIC robi Intelligent
  Power Select między ACIN/VBUS/baterią.
- **Dokument decyzji 2026-07-23 (pkt. 1.3)**: "VBUS AXP209 nie jest używane w ogóle... Port
  USB opisany w punkcie 9 jest elektrycznie niezależny od AXP209... potrzebuje jedynie
  D+/D-/GND... To rozdzielenie eliminuje pętlę powrotną, która w obecnym szkicu WIP wiąże
  VBUS z `+VUSB`."
- **Decyzja (2026-07-24)**: pośrodku obu stanowisk, nie żadne z nich w całości. Port USB
  debug pozostaje **wyłącznie do celów serwisowych** (D+/D-/GND), **bez** pełnego
  arbitrażu ACIN/VBUS/bateria z notatnika 2026-07-19 (system nadal zasilany wyłącznie z
  ACIN, VBUS nie zasila ani nie ładuje) — ale VBUS **jest** doprowadzone do AXP209
  wyłącznie jako sygnał wykrycia podłączenia kabla (insert/remove IRQ), nie jako ścieżka
  zasilania. Mechanizm: VBUS pin AXP209 podłączony fizycznie, ale power-path pozostaje
  wyłączony (`N_VBUSEN` nieaktywny) — AXP209 nadal widzi obecność napięcia na VBUS i zgłasza
  zdarzenie insert/remove, ale nic z tego napięcia nie trafia do systemu ani do ładowania.
  Pełny opis w sekcjach 1.3 i 9 niżej.

### 0.2 Transceiver CAN — ATA6561 (2026-07-23) vs rodzina "CAN FD-rated" (2026-07-19)

- **Dokument decyzji 2026-07-23 (pkt. 5.1)**: wybiera konkretnie **ATA6561-GAQW-N**.
- **Notatnik 2026-07-19** (końcowa część pkt. 5, po decyzji o osobnym kontrolerze+transceiverze):
  rekomenduje **ATA6563** (ten sam rdzeń co w kombo MCP251863, więc "zero niespodzianek"),
  albo MCP2542FD/MCP2562FD, albo TCAN1042 — dobór **explicite pod kątem transceivera
  wspierającego CAN FD** (bo kontroler MCP2518FD ma pracować w trybie Mixed CAN 2.0B/FD).
- **Decyzja (2026-07-24)**: wymóg na transceiver złagodzony — wystarczy dowolny transceiver
  wspierający klasyczny CAN 2.0B, z opcjonalnym, "najlepiej jak coś" wsparciem dla CAN FD
  (nie twardy wymóg certyfikowanego FD-rated chipu jak ATA6563). **ATA6561 zostaje wybraną
  częścią na razie** — nie ma potrzeby wymiany na ATA6563 ani weryfikacji pełnej zgodności z
  fazą danych CAN FD, dopóki magistrala faktycznie pracuje w trybie klasycznym. Jeśli
  protokół MPSWP/MPCC kiedyś realnie przejdzie na FD, dobór transceivera do tamtej
  przepustowości będzie osobną decyzją w tamtym momencie, nie teraz.

### 0.3 RTC — wbudowany w V3S czy zewnętrzny MCP79410 (czy oba)?

- **Notatnik 2026-07-19 (pkt. 4 i 7)**: zaczyna od założenia "V3S nie ma RTC", projektuje
  pełne podłączenie MCP79410 (I2C, adresy 0x6F/0x57, kwarc 6-9pF, zasilanie VCC z LDO1
  AXP209). Potem (pkt. 7) odkrywa, że **V3S ma jednak wbudowany RTC** (`VCC-RTC`, ball 98,
  zasilany tym samym LDO1) i zostawia to jako **jawnie otwartą decyzję operacyjną**: RTC
  wbudowany (zero dodatkowego BOM, ale ~25-30µA poboru backup → tygodnie/miesiące
  podtrzymania na małym ogniwie) vs MCP79410 (dodatkowy chip, ale ~1µA → lata podtrzymania) —
  zależnie od tego, jak długo terminal może realnie leżeć bez ładowania.
- **Dokument decyzji 2026-07-23 (pkt. 1.2)**: opisuje mechanizm AXP209 BACKUP+LDO1 ogólnie
  ("domena RTC"), **nie wspomina MCP79410 ani jednym słowem**, i nie precyzuje, czy LDO1
  zasila VCC-RTC V3S, VCC MCP79410, czy oba jednocześnie.
- **Decyzja (2026-07-24)**: **wbudowany RTC V3S**, nie MCP79410 — niższy koszt (zero
  dodatkowego chipu w BOM) i mniej routingu (brak dodatkowej magistrali I2C-drop, brak
  dodatkowego kwarcu poza tym, który i tak jest potrzebny dla VCC-RTC). LDO1 AXP209
  (strapowany na 3.3V) podłączony wyłącznie do **VCC-RTC V3S** (ball 98). MCP79410 **nie
  wchodzi do BOM terminala** — zostaje udokumentowany niżej jako rozważona, ale nieużyta
  alternatywa (przydatna gdyby w przyszłości pojawiła się potrzeba wielomiesięcznego
  podtrzymania RTC bez ładowania).

### 0.4 Nazewnictwo przycisku zasilania: PWRON vs PEK (terminologia, nie sprzeczność)

Notatnik 2026-07-19 nazywa pin/funkcję "PWRON" (nazwa fizycznego pinu AXP209 w datasheecie),
dokument 2026-07-23 nazywa ją "PEK" (Power Enable Key, nazwa funkcjonalna używana przez
driver `axp20x-pek`). To ten sam mechanizm opisywany dwoma nazwami z dwóch różnych warstw
(pin krzemu vs nazwa w mainline). Ujednolicone w sekcji 1.4 niżej — nie wymaga decyzji.

---

## 1. Rdzeń zasilania — AXP209

### 1.0 Interfejs komunikacyjny

AXP209 komunikuje się z V3S po **I2C (TWI)**, nie SPI — standardowy `axp20x-i2c` w mainline
(pkt. 1.6), na jednej wspólnej magistrali z TSC2007 (dotyk, pkt. 4.2). Przydział konkretnej
magistrali TWI i pinów, oraz uzasadnienie konsolidacji na jedną magistralę — zob. sekcja
"Przerwania (EINT) i magistrale I2C" niżej, gdzie rozstrzygnięty jest też ewentualny konflikt
z UART0.

### 1.1 Zasilanie z LiPo

Pojemność ogniwa 1S LiPo zależy od obudowy — **do ustalenia po finalizacji mechaniki**, ale
poniżej metoda liczenia, żeby nie zgadywać na ślepo, gdy wymiary będą znane:

| Blok | Typowy pobór |
|---|---|
| V3S rdzeń (1 GHz, obciążenie lekkie/średnie) | ~0.3–0.6 W |
| DDR (SY8088, 1.8 V) | ~0.1–0.2 W |
| Panel 5"/7" TFT + podświetlenie LED (pełna jasność) | ~1.0–2.5 W (dominujący blok) |
| EPHY + magnetyka Ethernet (link aktywny) | ~0.2–0.4 W |
| MCP2518FD + ATA6561 (aktywna transmisja) | ~0.05–0.1 W |
| MAX-M10S (tracking) | ~0.05–0.1 W |
| **Razem, working** | **~1.7–3.9 W** |

Przy sprawności DC-DC ~85% i napięciu ogniwa ~3.7 V nominalnie, prąd z baterii to
ok. 0.55–1.25 A. Jeśli bateria ma pełnić funkcję UPS/pomostu (a nie głównego źródła —
docelowo terminal zasilany z 8-40 V), 2–3 h podtrzymania to **~1500–3500 mAh 1S**. Jeśli ma
być realnym źródłem do pracy przenośnej przez zmianę, potrzeba więcej (5000+ mAh) — kwestia
do ustalenia z użytkownikiem końcowym, nie tylko z obudową (patrz też "Sizing baterii" w
sekcji "Otwarte tematy" niżej — to samo pytanie zadane już w notatniku 2026-07-19). Zostawić
na złączu JST-SH 2-pin (już przyjęte w szkicu: `SM02B-SRSS-TB`) margines prądowy i pamiętać
o zabezpieczeniu ogniwa (PCM/BMS wbudowany w pakiet LiPo — standard dla gotowych pakietów,
nie projektować własnego li-ion protection od zera).

### 1.2 Bateria rezerwowa 3V dla LDO1 (backup RTC) — i wybór, KTÓRY RTC ona zasila

Z datasheetu AXP209 ([X-Powers AXP209 v1.0](https://linux-sunxi.org/images/8/89/AXP209_Datasheet_v1.0en.pdf)):

- Pin **BACKUP** (pin 30) to dedykowane wejście/wyjście dla baterii podtrzymania RTC.
- LDO1 to *zawsze włączony* regulator domeny RTC. Gdy brak głównego zasilania (BAT/ACIN/VBUS),
  AXP209 **automatycznie** przełącza źródło LDO1 na baterię BACKUP.
- AXP209 potrafi też **ładować** baterię BACKUP z głównego zasilania: `REG35H[7]` włącza
  ładowanie, docelowe napięcie 3.0 V domyślnie (konfigurowalne `REG35H[6:5]`), prąd ładowania
  **200 µA**.
- Napięcie LDO1 ustawia się **sprzętowym strapem** pinu `LDO1SET`: do GND → 1.3V, do VINT →
  3.3V. **Strapować do VINT (3.3V)** — 1.3V byłoby poniżej minimalnego VCC MCP79410 (1.8V,
  patrz niżej) i wystarczające dla VCC-RTC V3S tylko marginalnie (V3S wymaga min. 3.0V na
  VCC-RTC, patrz niżej) — 3.3V jest właściwym wyborem niezależnie od tego, która opcja RTC
  zostanie ostatecznie wybrana.

**Decyzja: LIR2032** (Li-ion, doładowywalne, ~3.6 V nominalnie). To poprawny wybór pod kątem
bezpieczeństwa — AXP209 domyślnie *ładuje* BACKUP (`REG35H[7]`), więc zwykłe pierwotne CR2032
(nie do ładowania) byłoby tu niebezpieczne, gdyby ładowanie zostało włączone. Domyślny target
ładowania AXP209 to **3.0 V** — poniżej typowego napięcia ładowania LIR2032 (zwykle ~4.2 V
max), więc ogniwo będzie systematycznie niedoładowane względem swojej pełnej pojemności. To
nie jest problem: pobór RTC to mikroamperowe rzędy wielkości, a niedoładowanie w tę stronę jest
bezpieczne (nie ryzyko przeładowania). Pin BACKUP musi być **realnie podłączony** (nie NC) —
dopilnować przy rysowaniu schematu zasilania. Konkretny model/pojemność ogniwa (np. czy
zwykłe LIR2032 ~40mAh wystarczy, czy potrzeba supercapa o innej charakterystyce) — wciąż
otwarte, patrz "Otwarte tematy".

**Co dokładnie wisi na LDO1 — zdecydowane 2026-07-24, patrz 0.3**: mechanizm BACKUP+LDO1
opisany wyżej jest ustalony i pewny niezależnie od tego, co go konsumuje downstream. Poniżej
obie opcje, które były rozważane, z jasnym wskazaniem, która została wybrana:

- **Opcja A — RTC wbudowany w V3S. WYBRANA (decyzja 2026-07-24): niższy koszt (zero
  dodatkowego chipu w BOM) i mniej routingu (brak dodatkowej magistrali/drop I2C, brak
  osobnego footprintu MSOP-8).** V3S ma wewnętrzny RTC (sekcja 4.12 datasheetu Allwinner,
  rejestr bazowy `0x01c20400`): 30-bitowy licznik kalendarza YY-MM-DD/HH-MM-SS z generatorem
  lat przestępnych, alarm ogólny + tygodniowy, 8×32-bit rejestrów General Purpose z retencją
  gdy `VDD_RTC > 1.0V`. Cytat z datasheetu: *"The unit can be operated by the backup battery
  while the system power is off."*
  - **Zasilanie — uwaga o niespójności w datasheecie Allwinner**: proza sekcji 4.12 datasheetu
    nazywa niezależny pin zasilania "RTC_VIO", ale **czysta tabela pinów w sekcji 3.3**
    ("Table 3-3. Detailed Pin Description") jest jednoznaczna i inna — i to jej ufamy, nie
    niechlujnej prozie:
    - **VCC-RTC** (ball 98): opis "**RTC Power Supply**", typ P (zasilanie). To jest
      prawdziwy, niezależny pin zasilania RTC. Abs. max -0.3…3.6V, zalecane **3.0V min /
      3.3V typ / 3.6V max**.
    - **RTC-VIO** (ball 97): opis "**Internal LDO Output Bypass**", typ P. To NIE jest
      wejście na baterię backup — to wyprowadzenie wewnętrznego LDO (regulowanego rejestrem
      `VDD_RTC_REG`, 0.7-1.4V) wyłącznie pod kondensator odsprzęgający do masy.
  - VCC-RTC pasuje 1:1 do LDO1 AXP209 strapowanego na 3.3V.
  - Kwarc: X32KIN/X32KOUT (ball 96/95, typ AI/AO), zewnętrzny 32.768kHz dla dokładności.
    Driver ma fallback na wewnętrzny oscylator RC (`rc_osc_rate = 32000`) z automatycznym
    przełączaniem (`has_auto_swt`), gdyby kwarca nie było — RTC "działa" nawet bez niego,
    tylko mniej dokładnie.
  - Mainline: `drivers/rtc/rtc-sun6i.c`, `compatible = "allwinner,sun8i-v3-rtc"` —
    zweryfikowane dokładnie na tagu `v6.12` (realna baza projektu, patrz Dodatek "Migracja
    kernela"), pełny, kompletny driver (`sun6i_rtc_probe()`, obsługa alarmu/przerwań).
  - **Pobór w trybie backup**: brak w datasheecie Allwinner tabeli poboru prądu (nie
    publikują takich liczb). Znaleziony realny raport z listy mailingowej linux-sunxi dla
    **A20** (ten sam rdzeń IP RTC co w rodzinie V3/V3s/S3, więc rozsądne przybliżenie, **nie
    zweryfikowana specyfikacja dla samego V3S**): przy pełnym wyłączeniu zasilania pobór z
    VCC-RTC/baterii to **~25-30µA**.
  - Koszt: **zero dodatkowego BOM** poza kwarcem, który i tak byłby potrzebny niezależnie od
    wyboru (patrz opcja B).
- **Opcja B — MCP79410-I/MS. ROZWAŻONA, NIEUŻYTA (patrz decyzja 2026-07-24 wyżej)** —
  udokumentowana tu w całości jako alternatywa na przyszłość, gdyby operacyjnie okazało się
  potrzebne dłuższe podtrzymanie RTC bez ładowania niż daje wbudowany RTC V3S (patrz
  porównanie poniżej). (Microchip, rodzina MCP794xx) — ten sam układ jest już użyty w
  firmware węzłów czujnikowych MPSWP tego projektu (`wsc_mcp79410_write_time`), więc
  reużywamy istniejącą wiedzę/protokół. Sprawdzone w datasheecie Microchip DS20002266J:
  - VCC: 1.8-5.5V (wersja `-I` = Industrial, -40…+85°C — pasuje do terminala polowego).
    Podłączenie: LDO1 → pin **VCC** MCP79410 (nie VBAT!). Pin **VBAT** MCP79410 spiąć do
    masy (sam datasheet MCP79410 tak zaleca, gdy nie używa się jego własnego backupu — bo w
    tej architekturze to LDO1/BACKUP AXP209 pełni funkcję zasilania podtrzymującego, nie
    osobna bateria na VBAT MCP79410).
  - VBAT (backup): 1.3-5.5V, osobny pin, tylko podtrzymanie oscylatora/rejestrów
    (**~0.85-1.2µA**), **bez obsługi I2C** na tym pinie — nieużywany w tej architekturze
    (patrz wyżej).
  - Adresy I2C: `0x6F` (RTCC/SRAM/rejestry czasu), `0x57` (EEPROM 1Kbit) — brak konfliktu z
    AXP209 (`0x34`) ani TSC2007 (`0x48/0x49`).
  - Kwarc 32.768kHz **musi mieć load capacitance 6-9pF** — kwarce 12.5pF datasheet wprost
    odradza. Standardowy layout: krótkie ścieżki, guard-ring. Ten sam typ/koszt komponentu,
    który i tak byłby potrzebny dla wbudowanego RTC V3S (opcja A) — nie jest to różnica
    kosztowa między opcjami, tylko dodatkowy chip + jego własny footprint.
  - Pakiet MSOP-8, nic egzotycznego w dostępności.
  - Mainline: rodzina "z EEPROM" (`mcp7941x`, w odróżnieniu od `mcp7940x` bez EEPROM), w
    pełni obsługiwana przez uniwersalny driver `drivers/rtc/rtc-ds1307.c`: DT compatible
    `"microchip,mcp7941x"`, pełna obsługa alarmu/wybudzania (`mcp794xx_irq`,
    `mcp794xx_rtc_ops`), battery-backed SRAM wystawiane jako **NVMEM**.
  - Jedyny scenariusz, w którym miałby sens inny chip zamiast MCP79410: potrzeba dokładności
    bez kalibracji (np. DS3231 z TCXO ±2ppm vs zwykły kwarc + trim w MCP79410 rzędu
    dziesiątek ppm) — nieistotne dla terminala polowego, który i tak ma okazję
    zsynchronizować czas przy dokowaniu/ładowaniu.
- **Porównanie decydujące** (to samo ogniwo LIR2032 na pinie BACKUP obsługuje obie opcje —
  różni się tylko czas podtrzymania): wbudowany RTC V3S (~25-30µA, tygodnie/dwa miesiące
  podtrzymania na małym ogniwie ~40mAh) vs MCP79410 (~1µA, wiele lat podtrzymania na tym
  samym ogniwie).
- **Zdecydowane 2026-07-24**: wbudowany RTC V3S, zasadą "niższy koszt, mniej routingu" —
  patrz decyzja 0.3. Pytanie operacyjne z notatnika ("jak długo terminal może leżeć bez
  ładowania?") pozostaje więc rozstrzygnięte na "wystarczająco krótko, że tygodnie/miesiące
  podtrzymania na LIR2032 są akceptowalne" — jeśli w praktyce (np. sezonowe magazynowanie)
  okaże się to niewystarczające, MCP79410 (opcja B wyżej) zostaje udokumentowany jako gotowa
  do wdrożenia ścieżka odwrotu, elektrycznie kompatybilna z tym samym LDO1.

**Analiza (nie decyzja): rozważana zamiana LIR2032 → MS621FE-FL11E (2026-07-30)**

Rozważana zamiana ogniwa BACKUP z LIR2032 (20mm, w uchwycie) na **MS621FE-FL11E** (Seiko
Instruments, doładowywalne Li/MnO2, SMD 6.8mm × 2.1mm) — motywacja to najpewniej rozmiar/montaż
(SMD zamiast uchwytu na ogniwo guzikowe), nie chemia czy bezpieczeństwo ładowania.

Dane z datasheetu MS621FE-FL11E (zweryfikowane, nie z pamięci):
- Pojemność nominalna: **5.5 mAh** (vs ~40mAh przyjęte dla LIR2032 — ok. **7.3× mniej**).
- Napięcie nominalne: 3.0V. Zalecane napięcie ładowania: 3.1V (zakres 2.8-3.3V).
- Max prąd ładowania: **0.5mA** przy napięciu ogniwa ~3.1V (rośnie przy głębszym rozładowaniu,
  do 10mA przy 0V); standardowy/nominalny prąd ładowania rzędu **15µA**.
- AXP209 domyślnie ładuje BACKUP prądem **200µA** do target **3.0V** (`REG35H`, patrz wyżej) —
  mieści się w max 0.5mA specyfikacji MS621FE-FL11E, więc **elektrycznie kompatybilne bez zmian
  rejestrów**, ten sam argument bezpieczeństwa co przy LIR2032 (ogniwo doładowywalne, więc
  domyślne ładowanie AXP209 nie jest zagrożeniem).

Obliczenie czasu podtrzymania (przy poborze wbudowanego RTC V3S ~25-30µA przyjętym wyżej —
szacunek zapożyczony z A20, **niezweryfikowany wprost dla V3S**, więc obie liczby niżej mają tę
samą niepewność wejściową):
- LIR2032 (~40mAh): 40000µAh / 27.5µA ≈ 1454h ≈ **~60 dni (~2 miesiące)**.
- MS621FE-FL11E (5.5mAh): 5500µAh / 27.5µA ≈ 200h ≈ **~8 dni (~tydzień)**.

Spadek czasu podtrzymania: **~7.3×**, wprost proporcjonalny do stosunku pojemności (pobór RTC
nie zależy od wybranego ogniwa, tylko od tego, co konsumuje LDO1).

**Wniosek:** MS621FE-FL11E jest elektrycznie bezpiecznym zamiennikiem (kompatybilny z domyślnym
ładowaniem AXP209), ale realny czas przetrwania RTC bez zasilania głównego skraca się z ~2
miesięcy do ~1 tygodnia. Akceptowalne tylko jeśli terminal realnie nie leży bez ładowania dłużej
niż pojedyncze dni — w przeciwnym razie LIR2032 (albo, przy jeszcze dłuższym wymaganiu,
MCP79410 z opcji B wyżej) pozostaje lepszym wyborem. **Status: analiza, nie zmiana decyzji** —
LIR2032 wciąż jest wybranym ogniwem (patrz "Decyzja: LIR2032" wyżej i lista decyzji, poz. 5);
MS621FE-FL11E udokumentowany tu jako rozważana alternatywa pod kątem rozmiaru/footprintu,
kosztem czasu podtrzymania, do rozstrzygnięcia dopiero razem z wyborem obudowy/montażu.

**TODO przy bringupie (kernel/DT)** — dla wybranej opcji A (wbudowany RTC V3S):
- potwierdzić, że `CONFIG_RTC_DRV_SUNXI`/`rtc-sun6i` jest włączone w konfiguracji jądra
  Buildroot (`buildroot-mainline`),
- rozważyć `CONFIG_RTC_HCTOSYS`, żeby zegar systemowy ustawiał się automatycznie z RTC przy
  boocie,
- węzeł DT `compatible = "allwinner,sun8i-v3-rtc"` — potwierdzić obecność/poprawność w DT
  tej płyty.

Gdyby w przyszłości padła decyzja o dodaniu MCP79410 (opcja B wyżej), dodatkowe TODO:
`CONFIG_RTC_DRV_DS1307=y`, węzeł DT `compatible = "microchip,mcp7941x";` pod adresem `0x6f`.

### 1.3 Zasilanie zewnętrzne tylko z ACIN; VBUS wyłącznie do wykrywania kabla USB (decyzja 0.1)

- ACIN to wejście dla zasilacza/ładowarki "systemowej" (tu: buck 24V→5V z sekcji 2). **System
  jest zasilany i ładowany wyłącznie z ACIN** — VBUS AXP209 nie zasila systemu ani nie ładuje
  baterii (`N_VBUSEN` nieaktywny, power-path VBUS wyłączony; `PWR_FLAG`/`+VUSB` z obecnego
  szkicu do usunięcia z sieci ACIN/ładowania przy rysowaniu na czysto). To odrzuca wariant z
  notatnika 2026-07-19, w którym PMIC miał arbitrażować ACIN/VBUS/baterię jak Intelligent
  Power Select w telefonie (patrz historia decyzji 0.1).
- **VBUS pozostaje jednak fizycznie podłączone do AXP209 — wyłącznie jako sygnał wykrywania
  obecności kabla USB** (decyzja 2026-07-24, rozstrzygająca sprzeczność 0.1). AXP209 monitoruje
  napięcie na VBUS niezależnie od stanu `N_VBUSEN` i generuje zdarzenie insert/remove przez
  ten sam mechanizm przerwań co PEK (pkt. 1.4/1.6) — więc firmware/aplikacja może wiedzieć,
  że serwisant podłączył kabel debug, bez żadnego dodatkowego układu detekcji i bez oddawania
  VBUS jakiejkolwiek roli zasilającej. Zero napięcia z VBUS trafia do systemu ani do
  ładowania baterii — to czysto sygnałowe wykorzystanie tego pinu.
- Port USB opisany w punkcie 9 (debug/FEL) niesie więc: D+/D-/GND (dane/OTG device-only) +
  VBUS doprowadzone do AXP209 tylko po sygnał obecności kabla, jak wyżej. To wciąż eliminuje
  pętlę powrotną, która w obecnym szkicu WIP wiązała VBUS z `+VUSB` w sposób zasilający.

### 1.4 Przycisk zasilania (PWRON / PEK)

AXP209 ma sprzętową funkcję na pinie **PWRON** (nazwa pinu w datasheecie), funkcjonalnie
nazywaną **PEK** (Power Enable Key) w mainline — zwykły przycisk NO do GND, debounce robi
układ w krzemie, nie trzeba RC na płytce. Rejestrowo konfigurowalne czasy power-on/power-off
pokrywają cały scenariusz "jak w telefonie":

- **Boot z wyłączonego stanu**: system martwy → przytrzymanie PWRON dłużej niż skonfigurowany
  próg (typowo 1-3s, rejestr) → PMIC sam włącza szyny DCDC → boot.
- **Krótkie/długie przyciśnięcie w działającym systemie**: PWRON generuje przerwania na
  zbocze wciśnięcia/puszczenia. W Linuksie: `axp20x-pek` (`drivers/input/misc/axp20x-pek.c`,
  input driver, część `drivers/mfd/axp20x.c` MFD) → zwykłe zdarzenie `KEY_POWER` w
  `/dev/input/eventX`. **W pełni wspierane mainline** — nie trzeba własnej logiki
  power-button w firmware na poziomie kernela. **Rozróżnienie krótkie vs długie to jednak
  zadanie userspace'u** (mierzysz czas między press/release z evdev) — projekt jest na
  Buildroot bez systemd, więc potrzebny mały fragment kodu w aplikacji (lub osobny daemon)
  czytający ten event: krótkie = wygaszenie ekranu/wybudzenie, długie (np. >2s) =
  zainicjowanie czystego `poweroff`. Prościej: kiosk app łapie `KEY_POWER` bezpośrednio z
  evdev, tak jak już robi to z touchem.
- **Twardy wyłącznik awaryjny**: osobny rejestr PMIC konfiguruje czas wymuszonego odcięcia
  zasilania (typowo 4-10s), niezależny od softu — działa nawet gdy system się zawiesi.

**Zastrzeżenie (wciąż otwarte, patrz "Otwarte tematy")**: prawdziwy "sleep" jak w telefonie
(błyskawiczne wybudzenie, RAM w self-refresh) wymaga wsparcia suspend-to-RAM w mainline'owym
jądrze sun8i-v3s, co bywa niepewne dla tego SoC — nie zweryfikowane w żadnym z dwóch
dokumentów źródłowych. Na start: krótkie przyciśnięcie = wygaszenie LCD/podświetlenia (bez
ryzyka), długie = pełne kontrolowane wyłączenie przez PMIC.

### 1.5 AXP203 vs AXP209 — dlaczego AXP209, nie inny PMIC z tej rodziny

Sprawdzone bezpośrednio w datasheetach i źródłach jądra Linux (nie z pamięci): sprzętowo
AXP203 i AXP209 to niemal bliźniacze układy — ten sam pakiet QFN-48-EP 6×6mm, ta sama
architektura (2× DCDC buck, 5× LDO, 4× GPIO, 12-bit ADC, coulomb counter, ładowanie baterii
backup, IPS z 3 wejściami adapter/USB/bateria), zgodna mapa rejestrów.

**Decydująca różnica**: mainline Linux (`drivers/mfd/axp20x-i2c.c`) wspiera AXP202 i AXP209
(`x-powers,axp202` / `x-powers,axp209`) — **AXP203 nie jest tam wspierany wcale**. Bez tego:
brak gotowego drivera regulatorów, ADC/baterii, i **brak drivera `axp20x-pek` od przycisku
power** opisanego w pkt. 1.4.

**Decyzja: AXP209**, nie AXP203. Ten sam footprint/architektura, zero pracy nad driverami w
kernelu. AXP203 miałby sens tylko przy wyraźnej przewadze kosztowej/dostępności i gotowości
samodzielnie dopisać wsparcie w jądrze.

### 1.6 Mainline Linux — podsumowanie AXP209

| Funkcja | Driver | Status |
|---|---|---|
| I2C MFD core | `drivers/mfd/axp20x-i2c.c` + `drivers/mfd/axp20x-core.c` | mainline, dojrzały |
| Regulatory (DCDC2/3, LDO1-4) | `drivers/regulator/axp20x-regulator.c` | mainline |
| Power button (PWRON/PEK) | `drivers/input/misc/axp20x-pek.c` | mainline |
| ADC (bateria, ACIN, temperatura, GPIO ADC) | `drivers/iio/adc/axp20x_adc.c` | mainline (IIO) |
| power_supply (battery/ac/usb) | `drivers/power/supply/axp20x_*.c` | mainline — dokładnie te
  nody, które `core/include/csh/power_info.h` już czyta |
| GPIO | `drivers/gpio/gpio-axp209.c` (część mfd) | mainline |

Cały AXP209 jest jednym z najlepiej wspieranych PMIC-ów w mainline (rodzina sunxi/A20 go
używa od lat) — brak ryzyka na tej liście.

---

## 2. Zasilanie terminala 8-32(40) V → 5 V (ACIN)

- **Złącze: M12, 5-pin, PWR + CAN łącznie** (decyzja podjęta, pinout wewnętrzny, ostateczny):

  | Pin | Sygnał |
  |---|---|
  | 1 | VCC (8-40V) |
  | 2 | GND |
  | 3 | CAN_H |
  | 4 | CAN_L |
  | 5 | EARTH/PE |

  Jedno złącze niesie zarówno zasilanie, jak i magistralę CAN, zamiast dwóch osobnych złączy.
  Brak osobnego pinu CAN_GND — CAN_H/CAN_L referencjonują tę samą masę (pin 2), co dodatkowo
  wzmacnia zasadność nieizolowanego transceivera CAN z sekcji 5 (wspólna masa jest tu wymuszona
  przez samo złącze, nie tylko przez wybór projektowy).
  **EARTH/PE (pin 5) to osobny sygnał od GND (pin 2)** — nie łączyć ich bezpośrednio na PCB.
  Standardowa praktyka: PE idzie do obudowy/masy ochronnej (chassis), a połączenie z masą
  sygnałową/zasilającą PCB — jeśli w ogóle potrzebne — tylko w jednym punkcie (np. przez
  kondensator/warystor albo bezpośrednio przy złączu), żeby uniknąć pętli masy między obudową
  a elektroniką. Do ustalenia przy layoucie, czy PE w ogóle musi się stykać z GND obwodu, czy
  zostaje wyłącznie na obudowie.
- **Przetwornica buck 24V(8-40V)→5V: TPS54360B** (decyzja podjęta) — TI, wejście 4.5–60 V, do
  3.5 A, sync buck, dojrzała, dużo referencyjnych projektów, dostępna u DigiKey/Mouser/LCSC,
  spory zapas ponad 40 V na transienty bez dodatkowego stopnia ochrony napięciowej.
  (Kontekst z fazy koncepcyjnej: realne szyny przemysłowe 24V bywają specyfikowane szerzej
  niż nominał, typowo 18-32V + transienty wyżej — stąd dobór z zapasem na Vin_max, nie na
  "24V nominalnie".)
- **Ochrona wejścia** (rekomendacja, nie ma tego jeszcze w brief): dioda/ideal-diode lub
  P-MOSFET do ochrony przed odwrotną polaryzacją, TVS (np. SMBJ36A dobrany pod 40 V roboczych
  + margines na load-dump jeśli źródłem może być instalacja pojazdowa), bezpiecznik/PTC
  szeregowo. To samo podejście "TVS + pulseproof" co już zaplanowane dla CAN (pkt. 5) powinno
  objąć też wejście zasilania — ryzyko transientów jest tu większe niż na samej magistrali CAN.
  Rozważyć też filtrację/ferryt na wejściu przetwornicy, skoro 24V dzieli złącze z przewodem
  CAN (to samo złącze M12, pkt. wyżej) — żeby szum przełączania buck nie wracał na magistralę.
- **Pomiar napięcia wejściowego przez AXP209 GPIO ADC**: GPIO0/GPIO1 AXP209 mogą pracować jako
  wejścia ADC 12-bit (`REG85H` ustawia zakres wejściowy, rząd wielkości ok. 0.7–2.8 V — **do
  zweryfikowania dokładnych progów w datasheet przy projektowaniu dzielnika**, nie zakładać
  na pamięć). Dzielnik rezystorowy z 8-40 V trzeba przeskalować w dół do tego okna, z
  zapasem na górny limit 40 V (nie na nominalne 24 V!). W mainline eksponowane przez
  `drivers/iio/adc/axp20x_adc.c` jako kanał IIO — proste odczytanie z userspace
  (`/sys/bus/iio/devices/iio:deviceX/in_voltageY_raw`), bez potrzeby własnego sterownika.

---

## 3. SY8088 (DDR) + PT4101 (podświetlenie LED)

- **SY8088AAC z IPSOUT → +1.8 V DDR**: zgodne z dobrą praktyką V3S (DDR wymaga solidnego,
  osobnego od reszty cyfrowej trackingu zasilania) — Silergy SY8088 to sync buck do 1.2 A/3 A
  (wariant zależny od sufiksu), sprawdzony wybór, używany też w referencyjnych projektach V3S
  (np. FunKey S). Brak potrzeby sterownika — to zwykły fixed-output buck, w DT reprezentowany
  jako `regulator-fixed` bez żadnej kontroli software'owej.
- **PT4101 — driver LED backlight (boost, białe LED)**: EN pin steruje PWM bezpośrednio
  (chip włącza/wyłącza wyjście na poziomie EN, typowe dla tej klasy driverów). Rekomendacje:
  - Wyprowadzić EN z **PWM0 (pin PB4)** — jedyny kanał PWM użyty w tym projekcie (PWM1/PB5
    jest zajęty pod przerwanie AXP209, zob. "Przerwania (EINT) i magistrale I2C" niżej) —
    przez prosty tranzystor bufor jeśli poziomy logiczne wymagają (PT4101 EN zwykle akceptuje
    3.3V CMOS bezpośrednio — do potwierdzenia w konkretnym datasheet wariancie).
  - Częstotliwość PWM: 200 Hz–1 kHz to typowy zakres dla tej rodziny driverów przy
    dimming-przez-EN; poniżej ~200 Hz ryzyko widocznego migotania, zbyt wysoko (>kilka kHz)
    driver może nie nadążać z narastaniem prądu cewki przy krótkich duty cycle. Docelową
    wartość dobrać eksperymentalnie na prototypie.
  - W mainline: żaden dedykowany driver nie jest potrzebny — to klasyczny `pwm-backlight`
    (DT: `compatible = "pwm-backlight"`, `pwms = <&pwm0 ...>`), już standardowa, dojrzała
    ścieżka w jądrze.
  - Dobrać PT4101 pod konkretny string LED panelu (napięcie/prąd LED z datasheet panelu 5"/7"
    — typowe podświetlenia 5-7" to 4-6 LED w serii, Vf łącznie ~16-24V, prąd ~15-20 mA/gałąź;
    PT4101 (boost) musi pokrywać ten zakres wyjściowy z wejścia 3.3V/5V — potwierdzić przy
    wyborze konkretnego panelu, patrz sekcja 4).

---

## 4. Wyświetlacz LCD 5"/7" 800×480, dotyk rezystancyjny I2C, klawisze LRADC

### 4.1 Panel

- **Decyzja: 5"** (panel już posiadany). Rozdzielczość 800×480, pinout złącza 40-pin FPC
  zgodny z panelem już używanym i zweryfikowanym na dev-boardzie: **BL050S061-17 /
  T050SWV012T** (zob. `core/include/csh/config.h`) — gotowy, zweryfikowany punkt startowy,
  routing złącza LCD może iść wprost pod ten model.
- Gdyby w przyszłości pojawiła się potrzeba 7" (inna obudowa, większe cele dotykowe pod
  rękawice robocze): **pinout FPC innego panelu (np. rodziny Innolux AT070TN9x) różni się** od
  BL050S061-17 nawet przy tej samej rozdzielczości 800×480 — to osobna rewizja złącza LCD, nie
  coś do przewidzenia "na zapas" jednym uniwersalnym footprintem. Pełne porównanie
  40-pin/RGB666 (obecny panel) vs 50-pin/RGB888 (rodzina AT070TN9x) — patrz "Dodatek: panel
  7" (40-pin vs 50-pin)" na końcu tego dokumentu; decyzja 5" powyżej czyni to porównanie na
  razie odłożonym, nie blokującym.
- Sterownik obrazu: DE2 (Display Engine 2.0, jeden TCON, wyjście RGB) — **mainline od kernela
  4.13+** (`sun8i-v3s-display-engine`, `sun8i-v3s-tcon`), potwierdzone też praktycznie: obecny
  projekt już renderuje przez `/dev/fb0` na kernelu mainline 6.12.95 LTS na rzeczywistym
  sprzęcie (zob. `firmware/CanSensorHub-v3s/README.md`). Zero ryzyka na tej pozycji. Historia
  migracji na ten kernel (z forka 5.2.y) i wszystko, co się przy tym wydarzyło — patrz
  "Dodatek: migracja kernela 5.2.y → 6.12.95 LTS" na końcu tego dokumentu.

### 4.2 Dotyk rezystancyjny przez I2C

- Rekomendowany kontroler: **TSC2007** (TI) — 4-drutowy rezystancyjny, I2C, 12-bit ADC.
  **Mainline**: `drivers/input/touchscreen/tsc2007.c` + `Documentation/devicetree/bindings/
  input/touchscreen/ti,tsc2007.yaml`, dojrzały, prosty DT node (`compatible =
  "ti,tsc2007"`). Aplikacja `CanSensorHub-v3s` już ma własny raw-evdev reader
  (`app/src/evdev_touch.c`) + kalibrację 4-punktową w runtime (`app/src/touch_calib.h`) — nie
  wymaga zmian po stronie firmware poza device-tree.
- **To inny chip niż na obecnej płycie deweloperskiej LicheePi Zero** — tamta używa **NS2009**
  (też rezystancyjny, I2C), dla którego sterownik musiał zostać ręcznie sportowany do
  współczesnego API kernela, bo mainline nigdy nie scalił samego pliku drivera (patrz
  "Dodatek: migracja kernela", punkt o NS2009). To, że rezystancyjny dotyk po I2C w ogóle
  dobrze działa na tym SoC (potwierdzone na sprzęcie z NS2009 na LicheePi Zero), daje dodatkową
  pewność co do wyboru TSC2007 na nowej płycie terminala — sama kategoria rozwiązania
  (rezystancyjny + I2C + evdev + kalibracja w runtime) jest już zweryfikowana praktycznie,
  zmienia się tylko konkretny chip (TSC2007 ma gotowy driver w mainline, NS2009 nie miał).
- **Wspólna magistrala I2C z AXP209** (TWI1, zob. "Przerwania (EINT) i magistrale I2C" niżej) —
  brak konfliktu adresowego (AXP209 domyślnie 0x34, TSC2007 domyślnie 0x48/0x49), a
  konsolidacja na jedną magistralę uwalnia 2 dodatkowe piny EINT na przyszłość. Kompromis: zawis
  jednego urządzenia na wspólnym I2C teoretycznie może chwilowo zablokować drugie — patrz
  omówienie i mitygacja w sekcji EINT/I2C.
- Przerwanie dotyku (nPENIRQ): przypisane do PB3 — patrz tabela alokacji.
- Alternatywa, gdyby TSC2007 był trudny do zdobycia: ADS7846/XPT2046 (SPI, nie I2C — **niezgodne
  z wymaganiem I2C**, pomijamy) lub STMPE811 (I2C, ale to bardziej "touch controller +
  GPIO expander", overkill tutaj). TSC2007 pozostaje pierwszym wyborem.

### 4.3 Klawisze funkcyjne przez LRADC

- V3S ma **LRADC** (`allwinner,sun4i-a10-lradc-keys`, mainline `drivers/input/keyboard/
  sun4i-lradc-keys.c`), sprawdzone na LicheePi Zero i podobnych płytkach. Zasada: drabinka
  rezystorowa na jednym kanale koduje który klawisz wciśnięty jako poziom napięcia,
  każdy klawisz to sub-node DT (`label`, `linux,code`, `channel`, `voltage`).
- **Ograniczenie do zaakceptowania świadomie**: drabinka rezystorowa na jednym kanale
  wykrywa **jedno naciśnięcie naraz** — nie nadaje się do kombinacji klawiszy (np.
  Shift+F1) ani do wykrywania diagonali jak w D-pad. Dla "klawiszy funkcyjnych" (F1-F5,
  soft-keys) to zwykle akceptowalne (obsługa jednym palcem, jak na urządzeniu z
  ekranem dotykowym jako głównym interfejsem). Jeśli jakikolwiek klawisz musi działać
  jednocześnie z innym (np. modyfikator), przenieść **ten konkretny** klawisz na zwykłe
  GPIO+EINT zamiast LRADC.
- Do potwierdzenia w layout: ile kanałów LRADC V3S faktycznie wyprowadza w tym pakiecie
  (potwierdzony na pewno LRADC0 — drugi kanał, jeśli obecny, podwaja budżet klawiszy per
  drabinka; zweryfikować w `doc/V3S_CDR_STD_V1_0_20150514.pdf` przy przydziale pinów, nie
  zakładać z pamięci).

---

## 5. CAN 2.0B (rozszerzalne do CAN-FD) — MCP2518FD + transceiver

### 5.1 Kontroler i transceiver

- **MCP2518FD** (SPI CAN-FD controller) — **mainline**: `drivers/net/can/spi/mcp251xfd/`
  (`compatible = "microchip,mcp2518fd"`), dojrzały, używany szeroko (Raspberry Pi CAN-FD
  HATy, Jetson, itd.). DT wymaga m.in. `spi-max-frequency`, `interrupts`, `microchip,osc-freq`
  dopasowanego do rzeczywistego kwarcu (typowo 20 MHz albo 40 MHz).
- **Transceiver: ATA6561-GAQW-N** (decyzja potwierdzona 2026-07-24, rozstrzyga sprzeczność
  0.2) — nieizolowany, zgodny ISO 11898-2/-5, ma pin **STBY** (aktywny nisko = tryb
  normalny), zasilany logicznie z VIO (podłączyć do 3.3V V3S, nie do 5V) — poziomy TXD/RXD/
  STBY dopasowują się automatycznie do VIO. Wymóg na transceiver złagodzony przy tej decyzji:
  wystarczy wsparcie klasycznego CAN 2.0B, z ewentualnym częściowym wsparciem CAN FD jako
  bonusem, nie twardym wymogiem — więc **nie ma potrzeby wymiany na dedykowaną rodzinę
  "FD-rated" (ATA6563 i podobne)**, o czym mówiła wcześniejsza analiza koncepcyjna (notatnik
  2026-07-19, patrz historia w 0.2). Jeśli magistrala kiedyś realnie przejdzie na CAN FD,
  dobór transceivera pod tamtą przepustowość będzie osobną decyzją w tamtym momencie.

### 5.2 Izolowany czy nieizolowany transceiver — za/przeciw

| | Nieizolowany (ATA6561, jak zaplanowano) | Izolowany (np. ISO1042, ADM3054 + izolowane 5V) |
|---|---|---|
| Zalety | prosto, tanio, mało miejsca na PCB, niski pobór, transceiver już auto-grade (odporny na transienty magistrali) | odcina masę hosta od masy magistrali CAN — chroni przed prądami wyrównawczymi i skokami potencjału masy przy długich wiązkach/różnych sekcjach pojazdu |
| Wady | brak izolacji galwanicznej — potencjał masy terminala i magistrali CAN musi być wspólny/bliski | dodatkowa izolowana zasilaczyk DC-DC dla strony bus, więcej BOM, większa płytka, dodatkowe opóźnienie propagacji |
| Kiedy wybrać | terminal zasilany z **tego samego** systemu 8-40V co węzły CAN (typowy przypadek: maszyna/pojazd z jedną wspólną instalacją) | terminal i węzły CAN montowane na fizycznie różnych sekcjach z osobnymi masami/długim przewodem (np. ciągnik-przyczepa) |

**Rekomendacja na v1: pozostać przy nieizolowanym ATA6561** — terminal i tak zasila się z tej
samej szyny 8-40 V co magistrala (pkt. 2), więc korzyść z izolacji jest marginalna, a ochrona
PCB (TVS + terminacja cyfrowa, niżej) adresuje realne ryzyko (transienty), nie różnicę
potencjałów masy. Zostawić **świadomie** miejsce/split w warstwie masy pod złączem CAN, żeby
w v2 dało się bez przeprojektowania całej płyty wstawić wariant izolowany, jeśli pojawi się
przypadek użycia z długą wiązką między sekcjami.

### 5.3 STBY i terminacja cyfrowa

- MCP2518FD ma 3 osobne piny w tej roli: **INT** (pin 4, zawsze aktywny jako główne przerwanie
  do hosta, active-low — to on idzie do V3S na **PB2**, patrz tabela alokacji), **INT0/GPIO0/
  XSTBY** (pin 9) i **INT1/GPIO1** (pin 8, oba konfigurowalne jako GPIO przez bity PM0/PM1).
  Bit `XSTBYEN` konfiguruje INT0/GPIO0/XSTBY tak, by *automatycznie* sterował pinem STBY
  zewnętrznego transceivera (wysoki w Sleep, niski poza Sleep) — **to jest właściwy mechanizm**,
  nie bit-bangowanie GPIO w firmware. Zostawia **GPIO1** (pin 8) wolny pod sterowanie
  terminacją (zapis rejestru SPI z firmware).
- Terminacja: 2× fotoMOS (np. CPC1006N) załączane z GPIO1 przez firmware (zapis rejestru SPI)
  — cyfrowo włączana/wyłączana terminacja 120Ω. Przy terminacji dzielonej (split, 2×60Ω) dobrą
  praktyką EMC jest kondensator (np. 4.7 nF) z punktu środkowego do masy — tłumi zakłócenia
  common-mode, warto uwzględnić przy layoucie (ISO 11898-2, rekomendacja ogólna, nie wymóg
  formalny).
- **Zabezpieczenia PCB**: TVS na CANH/CANL (np. para dedykowanych diod TVS do magistrali CAN,
  rated na napięcia robocze busa), ferryt/common-mode choke na wejściu do transceivera, ESD
  na złączu zewnętrznym — zgodnie z założeniem z briefu, żadnych dodatkowych decyzji
  potrzebnych poza wyborem konkretnych referencji przy BOM.

### 5.4 Zasilanie magistrali 5V

- **TPS61240DRVT** (TI, boost, wejście nawet <1V, tu z 3.3V) → 5V dla VCC transceivera.
  Sprawdzony, prosty wybór, brak potrzeby sterownika (fixed regulator z punktu widzenia
  software).

### 5.5 Historia decyzji: dlaczego MCP2518FD, nie MCP2515/MCP25625/kombo

Ta ścieżka decyzyjna przeszła przez kilka etapów w fazie koncepcyjnej (notatnik 2026-07-19),
warto ją zachować jako uzasadnienie, nie tylko wynik:

1. **Punkt wyjścia**: V3S nie ma własnego kontrolera CAN, potrzebny bridge SPI-CAN. W
   momencie pierwszej analizy kernel projektu to był jeszcze fork Lichee-Pi `zero-5.2.y`
   (Linux 5.2, połowa 2019), co mocno ograniczało dostępność nowszych sterowników.
2. **MCP2515** — potwierdzony jako dobry, "kuloodporny" wybór: mainline
   `drivers/net/can/spi/mcp251x.c`, `compatible = "microchip,mcp2515"`, bardzo dojrzały,
   ~15 lat historii. Wymaga osobnego transceivera (np. SN65HVD230, TJA1050/1051).
3. **MCP25625** — lepsza alternatywa dla klasycznego CAN: ten sam rdzeń/rejestry i ten sam
   driver co MCP2515, ale transceiver zintegrowany w jednym IC (mniej BOM). Wsparcie w
   kernelu od wersji 5.2 — dokładnie tyle, ile było w Buildroot wtedy. Czysty drop-in, zero
   ryzyka backportu.
4. **CAN FD jako temat osobny**: rozważono `mcp25xxfd`/`mcp251xfd` (MCP2517FD/MCP2518FD/
   MCP251863, mainline ok. Linux 5.9/5.10) i `tcan4x5x` (TCAN4550 i pochodne, rdzeń Bosch
   M_CAN, mainline od Linux 5.4) — oba nowsze niż `zero-5.2.y`, więc na starym kernelu
   wymagałyby backportu (realna dodatkowa praca/ryzyko). Rekomendacja pierwotna: zostać przy
   MCP2515/MCP25625, odłożyć CAN FD jako świadomy temat na przyszłość — protokół sensorów
   MPSWP/MPCC to i tak czysty CAN 2.0B (29-bit ID), więc brak pilnej potrzeby.
5. **UPDATE po migracji na kernel 6.12.95** (patrz "Dodatek: migracja kernela" niżej,
   wykonana i potwierdzona na sprzęcie 2026-07-19): sprawdzone bezpośrednio na tagu `v6.12`
   przez GitHub API — `tcan4x5x-core.c` kompletny, `drivers/net/can/spi/mcp251xfd/` ma 14
   plików źródłowych (core, chip-fifo, crc16, dump, ethtool, ram, regmap, ring, rx, tef,
   timestamp, tx), dojrzały. **Jedyny powód unikania CAN FD (backport drivera) zniknął.**
   Nowa rekomendacja: MCP251863 (kombo MCP2518FD+ATA6563) albo TCAN4550, uruchomione na
   razie czysto w trybie klasycznego CAN 2.0B, bez przepisywania protokołu MPSWP/MPCC na FD.
6. **Decyzja architektoniczna użytkownika**: osobny kontroler + osobny transceiver, nie kombo
   (MCP251863/TCAN4550) — stąd MCP2518FD (kontroler) + ATA6561 (transceiver, sekcja 5.1) jako
   dwa oddzielne IC.
7. **MCP2518FD konkretnie, nie MCP2517FD**: sprawdzone w kodzie drivera na `v6.12`
   (`mcp251xfd_of_match[]`) i w datasheecie Microchip DS20006027B. Microchip sam rekomenduje
   MCP2518FD nad MCP2517FD dla nowych projektów (AN4808, migration guide) — pin-/funkcjonalnie
   kompatybilne, ale MCP2518FD dokłada: prawdziwy Low Power Mode (`LPMEN`, **max 10µA**,
   nieobecny w MCP2517FD — realny plus dla bateryjnego terminala), kilka poprawek erraty
   (Listen Only Mode przy TX, TX/RX MAB) przypisanych w kodzie drivera specyficznie do
   MCP2517FD, i pakiet "step-cut wettable flanks" (lepszy pod AOI). Bonus: tabela testowanych
   kombinacji SPI w kodzie drivera zawiera wprost `allwinner,sun8i-h3` z MCP2518FD przy 20MHz
   i 40MHz zegara CAN — H3 to bliski krewniak V3S (ta sama rodzina Allwinner), dobry sygnał
   zgodności (nie 1:1 potwierdzenie na V3S).
   - Dane do płytki: VDD 2.7-5.5V (działa wprost na 3.3V), wymagany zewnętrzny kwarc/rezonator
     4/20/40MHz (wewnętrzny oscylator nie wystarcza dla CAN FD — zbyt ścisły timing fazy
     danych), SPI do 20MHz tryby 0,0 i 1,1 z CRC na komendach SPI, pakiet SOIC-14 (łatwy do
     ręcznego lutowania) albo VDFN-14.
   - Tryb: kontroler ma osobno "CAN 2.0B Mode" i "Mixed CAN 2.0B and CAN FD Mode" — dla planu
     "na razie klasyczny CAN 2.0B, gotowość na FD później" sensowniej ustawić **Mixed** już
     teraz (w Linuksie to `ip link ... fd on`/`fd off` na poziomie SocketCAN), zero kosztu,
     nic nie trzeba przełączać w kontrolerze później. Tryb Mixed na kontrolerze nie wymaga,
     żeby transceiver był formalnie "FD-rated" — magistrala i tak pracuje dziś w trybie
     klasycznym; stąd decyzja 0.2 pozostawiająca ATA6561 (patrz 5.1) bez wymogu wymiany.
   - Zastrzeżenie: rodzina MCP251xFD/TCAN4x5x jest młodsza niż "kuloodporny" MCP2515 i ma
     bardziej złożony driver (FIFO/ring, CRC16 po SPI, timestamping) — margines ryzyka
     odrobinę większy, choć niski przy tej dojrzałości drivera. Realna różnica cenowa vs
     MCP25625 nieoceniona jeszcze przy finalizacji BOM.

### 5.6 Mainline — podsumowanie CAN

| Element | Sterownik/wsparcie |
|---|---|
| MCP2518FD | `mcp251xfd`, mainline, dojrzały |
| ATA6561 | brak sterownika potrzebnego (czysto analogowy transceiver) |
| SocketCAN (`can0`) | już zweryfikowane w architekturze firmware (`core/src/socketcan.c`) — patrz `firmware/CanSensorHub-v3s/README.md` |

Zakładam **jeden kanał CAN** zgodnie z opisem w brifie. Jeśli w KiCad pojawiają się dwie
instancje ATA6561 — to poza zakresem tej decyzji koncepcyjnej; jeśli ma to być redundantny/
drugi kanał CAN, to osobna decyzja do podjęcia jawnie, nie domyślna.

---

## 6. Pamięci: microSD (system) + SD/microSD (dane) — bez eMMC

**Zmiana decyzji (2026-07-24)**: **eMMC odrzucone** — trudna dostępność i wysoka cena (ok.
**2-4× droższe** niż odpowiednik gniazdo microSD + karta), bez wyraźnej korzyści dla tego
zastosowania na tyle dużej, żeby to uzasadnić. Architektura logiczna zostaje **bez zmian**
względem pierwotnej decyzji: jedna pamięć **wewnętrzna** na system + aplikację, jedna
**zewnętrzna** na dane — zmienia się tylko fizyczna realizacja pamięci wewnętrznej (karta
zamiast układu lutowanego), nie podział ról.

| Kontroler | Piny | Szerokość | Przeznaczenie |
|---|---|---|---|
| **SDC0** | PF0-PF5 | 4-bit | **microSD "systemowa"** w gnieździe z **pokrywką blokującą** (system + aplikacja) |
| **SDC1** | PG0-PG5 | 4-bit | **SD/microSD wymienna** (logi, dane aplikacji) |

SPI dla CAN (pkt. 5) jest na osobnych pinach (SDC2/port C) i nie wchodzi w grę z SDC0/SDC1 —
zgodnie z pierwotnym przydziałem z briefu.

**Dlaczego gniazdo z pokrywką blokującą na SDC0, a nie zwykłe push-push**: skoro pamięć
systemowa jest teraz kartą, a nie lutowanym układem, traci część naturalnej odporności
mechanicznej eMMC (wibracje, przypadkowe wysunięcie, zabrudzenie styków). Gniazdo z
mechaniczną pokrywką/blokadą (tzw. push-pull z zatrzaskiem lub gniazdo z klapką dociskową)
częściowo to kompensuje — karta nie wypadnie przy wstrząsach ani nie da się jej wysunąć bez
celowego działania. To wciąż nie dorównuje w pełni lutowanemu eMMC pod względem
odporności na wibracje/pył/wilgoć — świadomy kompromis przyjęty w zamian za koszt i
dostępność. Karta "danych" na SDC1 może zostać w zwykłym, prostszym gnieździe (push-push
bez blokady), bo jej przypadkowe wysunięcie nie zatrzymuje bootu systemu (patrz niżej).

Kilka właściwości tego układu wartych odnotowania:

- **Kolejność szukania bootloadera w BROM V3S**: SD/MMC na SDC0 → eMMC na SDC2 → SD na SDC2 →
  SPI NOR → SPI NAND. Karta microSD na SDC0 jest więc **pierwszym urządzeniem sprawdzanym
  przy starcie** — dokładnie tak samo dobre pod initial flash/recovery, jak wcześniej
  planowany eMMC (BROM nie rozróżnia "eMMC vs SD" na SDC0, obu widzi jako urządzenie
  SD/MMC). ([FunKey Project boot ROM docs](https://doc.funkey-project.com/developer_guide/software_reference/boot_process/boot_rom/))
- SDC1 (karta wymienna, dane) **nie jest w ogóle na liście BROM** — karta SD włożona w polu
  nie może przypadkowo stać się wektorem bootowania obcego systemu. Ta właściwość była i
  jest niezależna od tego, czy SDC0 to eMMC czy microSD — utrzymana bez zmian.
- microSD w trybie 4-bit (SDC0) jest wolniejsza niż eMMC w 8-bit, ale przy pojedynczym
  rdzeniu Cortex-A7 @ 1GHz i lekkiej aplikacji LVGL (nie wideo/streaming) to nie jest wąskie
  gardło — ta ocena nie zależy od tego, czy nośnik jest lutowany czy wymienny.
- SDC0 współdzieli piny z **JTAG** i z **jedną z dwóch lokalizacji UART0** — używając SDC0 pod
  microSD systemową, JTAG przestaje być dostępny (akceptowalne — nieużywane w produkcji), a
  UART0 wyprowadzony jest z **drugiej lokalizacji** (PB8/PB9) — patrz sekcja 10 i "Przerwania
  (EINT) i magistrale I2C" niżej. Bez zmian względem wersji z eMMC.
- **Wybór klasy karty (do BOM)**: rekomendacja to karta przemysłowa/wysokiej wytrzymałości
  (np. rodzina "industrial"/pseudo-SLC, wyższa liczba cykli zapisu i szerszy zakres
  temperatur niż konsumencka karta) na SDC0, ze względu na cykle zapisu generowane przez
  aktualizacje A/B (patrz "Dodatek: aktualizacje systemu" niżej) — zwykła karta konsumencka
  ma dużo niższą trwałość zapisu niż eMMC tej samej klasy, więc ten wybór częściowo
  rekompensuje utratę wytrzymałości eMMC. Karta "danych" na SDC1 może być zwykłą kartą
  konsumencką, bo profil zapisu (logi/CSV) jest inny i łatwiej ją wymienić w polu.

**Skutek dla pkt. 9 (USB) — zmieniony względem wersji z eMMC**: przy eMMC initial-flash
wymagał USB **FEL** (`sunxi-fel`), bo układ lutowany nie da się wyjąć i zaprogramować na
zewnątrz. **Z kartą microSD na SDC0 to się upraszcza** — kartę systemową da się wyjąć z
gniazda i zapisać obrazem na zwykłym czytniku kart z poziomu PC (workflow znany np. z
Raspberry Pi), bez potrzeby FEL do pierwszego flashowania. FEL pozostaje **drugorzędną
ścieżką recovery** — przydatną, gdy karta nie jest fizycznie dostępna bez demontażu obudowy,
albo gdy karta jest uszkodzona/skorumpowana w sposób niewykrywalny bez sprzętu diagnostycznego
— ale nie jest już jedyną praktyczną metodą initial-flash, jak było zaplanowane dla eMMC.

**Warstwa softwarowa nad tym storage'em (aktualizacje A/B przez RAUC)** — zaprojektowana
niezależnie w fazie koncepcyjnej, w pełni zgodna z powyższym przydziałem pinów/BROM: patrz
"Dodatek: aktualizacje systemu — schemat A/B (RAUC)" na końcu tego dokumentu, zaktualizowany
pod microSD zamiast eMMC (pojemność/wytrzymałość karty, nie zmiana logiki A/B), w tym wymóg
osobnej, niededuplikowanej partycji danych poza slotami A/B.

---

## 7. GPS — MAX-M10S, dwie anteny (wewnętrzna domyślna + zewnętrzna opcjonalna)

**Zmiana decyzji (2026-07-24)**: moduł zmieniony z MAX-M10C na **MAX-M10S** (ta sama rodzina
u-blox M10, pakiet MAX LCC 18-pin, ta sama architektura interfejsów — zmiana nie wpływa na
resztę tej sekcji poza tym, co niżej). Dodatkowo: **dwie anteny zamiast jednej** — wewnętrzna
(pasywna, używana domyślnie) oraz zewnętrzna (aktywna, przez złącze SMA panelowe + pigtail
U.FL do PCB), z automatycznym przełączaniem między nimi. Poniżej pełne uzasadnienie i
research przeprowadzony w tej sesji (2026-07-24).

- Zasilanie: 3.3V (rodzina MAX-M10 akceptuje typowo 1.71-3.6V VCC — **do potwierdzenia
  dokładnie dla wariantu M10S** w jego integration manual przy BOM, nie zakładać 1:1 z M10C
  bez sprawdzenia). Backup RTC/almanach: pin **V_BCKP** modułu podłączony do tej samej domeny
  co AXP209 BACKUP/LDO1 (pkt. 1.2) — jedna bateria/supercap zapasowa obsługuje RTC hosta *i*
  RTC GPS jednocześnie, mniej BOM, spójne z "backup jak RTC" z briefu.
- Interfejs: UART (rekomendacja: **UART2**, TX/RX na PB0/PB1 — RTS/CTS, fizycznie na PB2/PB3,
  świadomie NC i przeznaczone zamiast tego pod EINT, zob. "Przerwania (EINT) i magistrale I2C"
  niżej). Zero specjalnego sterownika kernela — moduł mówi
  NMEA/UBX po zwykłym `/dev/ttySx`, obsługa w userspace (`gpsd` + `ubxtool`/`gpsctl` do
  konfiguracji binarnego protokołu UBX jeśli potrzebne strojenie, np. wybór constellations).

### 7.1 Dlaczego nie switched SMA — i co zamiast tego (research 2026-07-24)

Pierwotna rekomendacja tego dokumentu (2026-07-23, patrz historia niżej) była złącze SMA z
mechanicznym przełączaniem (styk wewnętrzny fizycznie odcinający antenę wbudowaną po
włożeniu wtyczki zewnętrznej) — prostsze niż aktywny RF switch, bez dodatkowego IC czy
sygnału sterującego. **Sprawdzone w tej sesji**: taki mechanizm przełączający jest cechą
fizyczną korpusu złącza montowanego bezpośrednio na PCB (przez styk stykający się z
laminatem) — nie występuje w praktyce w wersji **pigtail/bulkhead** (złącze SMA na panelu
obudowy połączone krótkim kablem koncentrycznym + U.FL do płytki), bo styk przełączający
musiałby być wyprowadzony osobnym przewodem obok samego kabla RF, czego dostawcy typowo nie
oferują w gotowych zestawach pigtail. To potwierdza to, czego nie udało ci się znaleźć —
**switched SMA w formie pigtail nie jest realną opcją** dla tej architektury obudowy
(złącze na panelu, płytka w środku). Rozwiązanie zamiast tego: przełącznik RF sterowany
elektronicznie, jak zaproponowałeś.

### 7.2 Antenna supervisor MAX-M10S (potwierdza pin LNA_EN)

Sprawdzone bezpośrednio w Integration Manual u-blox MAX-M10S (UBX-20053088): moduł ma
wbudowany **antenna supervisor** z dwoma wariantami konfiguracji:
- **2-pinowy**: `ANT_OFF_N` (sterowanie zasilaniem/biasem anteny aktywnej) + `ANT_SHORT_N`
  (sygnalizacja zwarcia),
- **3-pinowy**: jak wyżej + `ANT_DETECT` (dodatkowo wykrywa obwód otwarty, czyli brak anteny).

**Kluczowe potwierdzenie Twojego pytania**: sygnał `ANT_OFF_N` jest **domyślnie przypisany
właśnie do fizycznego pinu `LNA_EN`** modułu — to dokładnie pin, o którym pytałeś. `ANT_SHORT_N`
i `ANT_DETECT` można przypisać do dowolnych wolnych PIO modułu. Sterowanie napięciem anteny
włącza się konfiguracyjnie przez `CFG-HW-ANT_CFG_VOLTCTRL` (domyślnie `true`), a
`CFG-HW-ANT_CFG_PWRDOWN` automatycznie odcina zasilanie anteny po wykryciu zwarcia (ochrona).
Status supervisora jest też czytelny programowo przez komunikat **UBX-MON-RF**: pole
`antStatus` (`INIT`/`DONTKNOW`/`OK`/`SHORT`/`OPEN`) i `antPower` (`OFF`/`ON`/`DONTKNOW`).

**Ograniczenie ważne dla architektury dwuantenowej**: ten wbudowany supervisor monitoruje
**jedno** wejście `RF_IN` modułu — sam z siebie nie potrafi przełączać między dwiema
fizycznie osobnymi antenami. To wciąż wymaga zewnętrznego przełącznika RF; supervisor
dostarcza tylko diagnostykę/ochronę na torze, który akurat jest wybrany.

### 7.3 Rozwiązanie rekomendowane: dedykowany chip RF switch+LNA+bias (Analog Devices/Maxim MAX2674 / MAX2676)

Znaleziony w tej sesji dokładnie pod ten scenariusz: **"GPS/GNSS LNAs with Antenna Switch and
Bias"** (MAX2674 — wariant wysokiego wzmocnienia, MAX2676 — wariant wysokiej liniowości).
To **w pełni sprzętowe, automatyczne rozwiązanie** — zero logiki firmware/GPIO potrzebne do
samej decyzji przełączania:

- **Mechanizm autodetekcji**: chip stale monitoruje pobór prądu na pinie `ANT` (ten sam pin
  dostarcza bias/zasilanie do ewentualnej anteny aktywnej podłączonej zewnętrznie). Brak
  poboru prądu (nic zewnętrznego podłączone) → chip domyślnie kieruje sygnał przez własny,
  wbudowany LNA zasilany z toru **wewnętrznej anteny pasywnej** — dokładnie "wewnętrzna
  domyślna", o którą pytałeś. Wykryty pobór prądu (aktywna antena zewnętrzna podłączona i
  pobierająca zasilanie z tego samego pinu) → automatyczny bypass wewnętrznego LNA, sygnał
  idzie wprost z toru zewnętrznego (bo aktywna antena ma już własny LNA na pokładzie) — efekt
  uboczny: **poprawiona ochrona przed zwarciem** na porcie zewnętrznym, bo chip i tak monitoruje
  ten prąd.
- Zasilanie: pojedyncze 1.6-3.6V (pasuje wprost do 3.3V płyty).
- Pakiet: bardzo mały WLP (0.86×1.26×0.64mm) — **sprawdzić przy zamawianiu płytek możliwość
  montażu/AOI** (np. u JLCPCB) ze względu na rozmiar, zanim trafi do BOM.
- Pin **SHDN** (shutdown, <10µA w stanie wyłączonym) — opcjonalnie podłączalny do **wolnego
  GPIO V3S** (tego dodatkowego pinu, o którym wspomniałeś) dla programowego wyłączenia całego
  toru RF GPS w celu oszczędności energii, gdy GPS nieużywany. To jedyna rola, jaką miałby
  wolny GPIO w tej architekturze — sama decyzja "która antena" go nie wymaga.
- Skutek dla antenna supervisora modułu (7.2): w tej architekturze staje się drugorzędny —
  MAX2674/76 dostarcza gotowy sygnał do `RF_IN` modułu niezależnie od tego, która antena jest
  aktywna, więc `CFG-HW-ANT_CFG_VOLTCTRL` modułu można zostawić wyłączone (moduł nie musi już
  sam zasilać żadnej anteny — robi to chip przed nim).
- **Dostępność (flaga do sprawdzenia przy BOM)**: część nieco starsza (Maxim, obecnie pod
  marką Analog Devices po przejęciu) — na 2026-07 wciąż widoczna w katalogu ADI i u DigiKey,
  ale ten projekt już raz trafił na problem dostępności/ceny przy podobnie niszowym wyborze
  (eMMC, patrz pkt. 6) — **potwierdzić realny lead-time/cenę przy finalizacji BOM**, nie
  zakładać dostępności tylko na podstawie obecności w katalogu producenta.
- **Walidacja koncepcji w branży**: automatyczne przełączanie antena-wewnętrzna-pasywna /
  antena-zewnętrzna-aktywna jest znaną, ugruntowaną funkcją w innych modułach GPS (np.
  Quectel L80 reklamuje wprost "automatic antenna switching" między anteną wbudowaną a
  zewnętrzną) — to nie jest rozwiązanie eksperymentalne, tylko established wzorzec, tu
  zrealizowany osobnym chipem RF zamiast wbudowaną funkcją modułu (bo MAX-M10S jej nie ma
  wbudowanej — stąd potrzeba osobnego IC).

### 7.4 Rozwiązanie zapasowe: RF switch sterowany z GPIO + odpytywanie UBX-MON-RF

Gdyby MAX2674/MAX2676 okazał się niedostępny/zbyt drogi przy BOM — wariant DIY korzystający
właśnie z tego wolnego GPIO V3S, o którym wspomniałeś:

- **Switch**: kandydat sprawdzony w tej sesji — **pSemi PE4259** (SPDT, zakres 10 MHz-3000
  MHz — pokrywa GPS L1 1575.42 MHz z dużym zapasem, wtrącenie 0.35 dB @ 1000 MHz, izolacja
  30 dB @ 1000 MHz, sterowanie **pojedynczym pinem** logiki CMOS, pakiet SC-70). **Zastrzeżenie
  wymagające weryfikacji przed BOM**: bias zasilający antenę aktywną musi popłynąć przez
  switch do portu zewnętrznego — część switchy RF ma szeregowe kondensatory blokujące DC i
  **nie przepuszczają zasilania stałoprądowego**; nie potwierdzono w tej sesji, czy PE4259
  konkretnie to robi (zakres podany jako "10 MHz-3000 MHz", nie "DC-3000 MHz", co jest sygnałem
  ostrzegawczym, nie potwierdzeniem problemu). Jeśli nie przepuszcza DC, potrzebny osobny
  bias-tee wstrzykujący zasilanie anteny **za** switchem, po stronie zewnętrznej, zamiast
  polegać na biasie modułu płynącym przez cały tor.
- **Logika (firmware)**: domyślnie ścieżka wewnętrzna wybrana przez GPIO, bias antenowy
  modułu (`ANT_OFF_N`/`LNA_EN`, patrz 7.2) wyłączony — antena wewnętrzna pasywna go nie
  potrzebuje, a podanie biasu na tor, który może być zwarty do masy przez element pasywny,
  ryzykowałoby fałszywe zgłoszenie `SHORT`. Firmware (np. przy starcie, ewentualnie okresowo)
  przełącza GPIO na ścieżkę zewnętrzną, włącza bias, odczytuje `antStatus` z UBX-MON-RF —
  `OK` → zostaje na zewnętrznej; `OPEN` → wraca na wewnętrzną, wyłącza bias. To realnie
  automatyczne (nie wymaga ręcznej interwencji użytkownika poza fizycznym podłączeniem
  anteny), ale wymaga własnej pętli firmware (analogicznie do już istniejącej obsługi
  evdev/touch) i weryfikacji DC-pass w wybranym switchu.

**Rekomendacja**: **opcja 7.3 (MAX2674/MAX2676) jako pierwszy wybór** — purpose-built,
sprzętowo automatyczne, brak ryzyka związanego z przepustowością DC switcha i brak potrzeby
pętli firmware. **Opcja 7.4 jako udokumentowany fallback**, korzystający z wolnego GPIO V3S,
na wypadek problemów z dostępnością/ceną dedykowanego chipu.

- **Antena zewnętrzna**: SMA panelowe + pigtail U.FL do PCB, jak zaplanowano (bez zmian) —
  tylko bez funkcji mechanicznego przełączania (patrz 7.1).
- **Antena wewnętrzna**: dobrać pasywną antenę GPS (np. ceramic patch) dopasowaną do 50Ω,
  montowaną wewnątrz obudowy — konkretny model do wyboru przy BOM.

**Źródła (research 2026-07-24)**: [MAX-M10S Integration Manual (u-blox, UBX-20053088)](https://content.u-blox.com/sites/default/files/MAX-M10S_IntegrationManual_UBX-20053088.pdf) · [PE4259 Product Specification (pSemi)](https://www.psemi.com/pdf/datasheets/pe4259ds.pdf) · [MAX2674/MAX2676 GPS/GNSS LNAs with Antenna Switch and Bias (Analog Devices)](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX2674-MAX2676.pdf) · [Autodetect antenna connection explanation (Analog Devices EngineerZone)](https://ez.analog.com/amplifiers/w/documents/22617/can-you-provide-detailed-explanation-on-autodetect-antenna-conenction-feature-in-max2674-max2676).

---

## 8. Ethernet 10/100M

- EPHY (PHY 10/100) jest **zintegrowany w V3S** — nie potrzeba zewnętrznego układu PHY, tylko
  magnetyka (transformator, osobny lub zintegrowany z gniazdem RJ45 — "RJ45 z magnetyką"
  upraszcza layout), ESD na linii, rezystory bias.
- **Mainline**: `dwmac-sun8i` / `sun8i-emac` — **już zweryfikowane na rzeczywistym sprzęcie**
  w tym projekcie (kernel mainline 6.12.95 LTS, potwierdzony DHCP + ping z zewnętrznej
  maszyny, zob. `firmware/CanSensorHub-v3s/README.md`, sekcja "Verified on hardware"). To
  jedyny punkt z całej dziesiątki, który ma już dowód działania na prawdziwym V3S w tym
  konkretnym projekcie — zero ryzyka softwarowego, tylko standardowy layout magnetyki/RJ45.
  Ta weryfikacja pochodzi z migracji kernela opisanej w "Dodatku: migracja kernela" niżej —
  Ethernet został tam **włączony po raz pierwszy w ogóle** (na starym forku 5.2.y nigdy nie
  był aktywny w DT), celowo jako hotplug/nieblokujący start (`ip link up` + `udhcpc -b` w
  tle), żeby terminal polowy nie czekał na kabel przy starcie.

---

## 9. USB — debug / initial flash (niski priorytet)

- V3S ma **jeden port USB OTG** (brak hosta jako osobnej instancji). Rekomendacja: podłączyć
  jako **device-only** (Micro-USB lub USB-C okablowany jako zwykłe urządzenie, bez logiki PD/
  roli — CC1/CC2 przez rezystory 5.1kΩ do GND jeśli USB-C, żeby hosty USB-C rozpoznały
  urządzenie jako device).
- Zastosowania: (a) **FEL mode** — sprzętowy bootloader Allwinner w ROM, pozwala wgrać
  SPL/U-Boot/obraz bezpośrednio przez USB z PC (`sunxi-fel`) bez potrzeby wyjmowania karty —
  od zmiany decyzji na microSD zamiast eMMC (pkt. 6) to już nie jedyna praktyczna ścieżka
  initial-flash (kartę systemową da się teraz też wyjąć i zaprogramować zewnętrznie), ale
  zostaje przydatną drugorzędną ścieżką recovery, gdy karta nie jest fizycznie dostępna bez
  demontażu obudowy; (b) USB gadget (serial/mass storage) do debugu z poziomu działającego
  systemu.
- **Rozdzielone od zasilania systemu, zgodnie z pkt. 1.3 (decyzja 0.1, 2026-07-24)** — ten
  port nie zasila systemu ani nie jest zasilany z systemu do PMIC; niesie D+/D-/GND (dane) +
  VBUS doprowadzone do AXP209 wyłącznie jako sygnał wykrycia podłączenia kabla (insert/remove
  IRQ, `N_VBUSEN` nieaktywny) — bez żadnej roli zasilającej.
- Brak potrzeby dodatkowego mostka USB-UART na tym złączu — pokrywa się funkcjonalnie z
  punktem 10 (patrz niżej), nie dublować.
- **Złącze: JST**, wyłącznie do celów serwisowych/debug (nie panelowe gniazdo USB-A/Micro/C) —
  pasuje do niskiego priorytetu tego punktu z briefu (decyzja 7 w podsumowaniu niżej).

---

## 10. UART debug na PCB

- Rekomendacja: **UART0** wyprowadzone na prosty 3-pinowy header TTL (TX/RX/GND, 3.3V logic)
  — **lokalizacja PB8/PB9** (druga z dwóch dostępnych, bo PF-lokalizacja zajęta przez microSD
  systemową na SDC0, pkt. 6). PB8/PB9 fizycznie dzielą pin z TWI1, ale w tym projekcie nie ma to znaczenia
  — TWI1 celowo poprowadzony przez swoją drugą lokalizację (PE21/PE22), zob. "Przerwania
  (EINT) i magistrale I2C" niżej. Brak wbudowanego mostka USB-UART na płycie (np. CH340/
  CP2102) — użytkownik/serwisant podłącza zewnętrzny adapter FTDI, zgodnie ze standardową
  konwencją urządzeń przemysłowych i tak, by nie dublować funkcji z portem USB debug (pkt. 9).
- Budżet UART: **UART0 → debug console**, **UART2 → GPS** (pkt. 7). V3S w tym pakiecie ma
  faktycznie **trzy** UART-y (UART0, UART1, UART2 — UART1 widoczny na PE21/PE22, tych samych
  pinach zajętych tu pod TWI1), ale tylko dwa są potrzebne w tym projekcie; UART1 zostaje
  nieużyty (świadomie, bo jego pinów potrzebuje TWI1) — nie jest to ograniczenie SoC, tylko
  wynik przydziału zasobów w tym konkretnym projekcie.

---

## 11. Audio (słuchawki/mikrofon) i STM32 — kontroler przycisków headsetu + dodatkowych

**Nowy wymóg (2026-07-25)**: ta sama płyta ma też posłużyć do budowy innego, pokrewnego
urządzenia, które wymaga obsługi dźwięku (słuchawki + mikrofon) oraz obsługi przycisków w
zestawie słuchawkowym (headset: PTT, Vol+, Vol-) i dodatkowych przycisków panelowych **poza
zakresem LRADC** — sekcja 4.3 już wcześniej flagowała dokładnie to ograniczenie ("jeśli
jakikolwiek klawisz musi działać jednocześnie z innym... przenieść ten konkretny klawisz na
zwykłe GPIO+EINT zamiast LRADC"); ten wymóg to systematyczna, generalizowana wersja tamtego
zastrzeżenia, nie coś nowego jakościowo. **Decyzja**: dodać osobny mikrokontroler **STM32**
(kandydat: **STM32L432KCU6**, ARM Cortex-M4, pakiet UFQFPN32/QFN-32, ≤32 piny) jako
koprocesor I2C obsługujący przyciski i (opcjonalnie) sygnalizację audio, podłączony do V3S po
I2C (współdzielona magistrala TWI1) + jeden EINT (żądanie komunikacji).

### 11.1 Ścieżka audio (słuchawki + mikrofon)

Audio **nie** idzie przez STM32 — V3S ma **wbudowany kodek audio** (potwierdzone w pełnej
rozpisce pinów, `V3S_PINOUT_ROZPISKA.md`): piny `HPOUTL`/`HPOUTR` (wyjście słuchawkowe),
`HPCOM`/`HPCOMFB` (common-mode feedback wzmacniacza słuchawkowego), `HBIAS` (bias), `HPVCCIN`/
`HPVCCBP` (zasilanie analogowe wzmacniacza słuchawkowego), `MICIN1P`/`MICIN1N` (wejście
mikrofonowe różnicowe), `AVCC`/`AGND` (zasilanie analogowe kodeka), `VRA1`/`VRA2` (wewnętrzne
referencje audio) — wszystkie te piny były dotąd oznaczone jako "nieużywane" (brak audio w
zakresie pierwotnego brifu); **teraz aktywowane**. `V3S_PINOUT_ROZPISKA.md` zaktualizowany
odpowiednio.

- **Złącze**: standardowe gniazdo audio TRRS 3.5mm (słuchawka L/R lub mono + mikrofon + masa),
  z mechanicznym stykiem detekcji wpięcia (jack-detect) wyprowadzonym na GPIO STM32 — patrz
  11.3.
- **Dokładne wartości komponentów aplikacyjnych kodeka V3S** (kondensatory sprzęgające DC na
  MICIN1P/N, dokładna wartość bias, dopasowanie HPVCCIN/HPVCCBP) — **poza zakresem tej sesji,
  otwarte** — wymaga pełnego application circuit z datasheetu V3S/Allwinner dla sekcji audio
  (nie sprawdzone tu wprost), patrz "Otwarte tematy".

### 11.2 STM32 — wybór i podłączenie do V3S

- **Kandydat: STM32L432KCU6** (ST, Cortex-M4, 256KB flash, UFQFPN32/QFN-32 5×5mm) — pasuje do
  wymogu "≤32 pin, QFN-32". Zasoby wystarczające z dużym zapasem na to zadanie: I2C (master/
  slave), ADC 12-bit (kilka kanałów zewnętrznych — potrzebny do drabinki rezystorowej
  headsetu, patrz 11.3), wiele GPIO z EXTI (debounce/IRQ), działa na wewnętrznym oscylatorze
  HSI16 bez potrzeby zewnętrznego kwarca (I2C slave + ADC + debounce nie wymagają precyzji
  krystalicznej) — mniejszy BOM. Pełna rozpiska pinów: `STM32_AUDIO_BUTTONS_ROZPISKA.md`.
- **I2C**: STM32 jako **slave** na już istniejącej wspólnej magistrali **TWI1 (PE21/PE22)**,
  razem z AXP209 (0x34) i TSC2007 (0x48/0x49) — trzecie urządzenie, adres do wyboru bez
  konfliktu (konfigurowalny w firmware STM32, np. `0x20`). Nie zakładać nowej magistrali —
  Port E i tak nie ma EINT (patrz sekcja EINT niżej), więc dokładanie kolejnego urządzenia na
  TWI1 nie kosztuje nic w budżecie przerwań; jedyny koszt to zwiększone ryzyko, że zawis
  jednego urządzenia chwilowo zablokuje **trzy** peryferia zamiast dwóch (patrz zaktualizowany
  akapit "kompromis" w sekcji EINT/I2C niżej).
- **IRQ (żądanie komunikacji)**: STM32 GPIO skonfigurowany jako open-drain, aktywny nisko,
  zewnętrzny pull-up (~10kΩ do 3.3V) → **V3S PB6** (`PB_EINT6`) — konsumuje **jeden z dwóch
  wolnych pinów rezerwy EINT** (sekcja "Przerwania EINT" niżej), analogicznie do IRQ AXP209 na
  PB5. Budżet EINT spada z 2 wolnych do **1 wolnego (PB7)** — patrz zaktualizowana tabela.
- **Zasilanie**: 3.3V z domeny VCC-IO (ta sama szyna 3.3V co reszta cyfrowej logiki V3S/AXP209
  DCDC3) — brak potrzeby dodatkowej LDO, poziomy logiczne I2C/GPIO od razu zgodne z V3S.
- **Programowanie/debug**: SWD (SWDIO/SWCLK) + NRST wyprowadzone na mały header serwisowy
  (analogicznie do UART0 debug, sekcja 10) — nie wchodzi do BOM produkcyjnego, tylko punkt
  programowania/debugowania firmware STM32 w produkcji/serwisie.

### 11.3 Obsługa przycisków headsetu (PTT, Vol+, Vol−) i przycisków dodatkowych — analiza

**Niepewność co do fizycznego interfejsu headsetu** (świadomie nierozstrzygnięta na sztywno,
żeby nie blokować architektury jednym założeniem — do potwierdzenia z konkretnym produktem):
dwa powszechne wzorce różnią się elektrycznie i mechanicznie:

| | TRRS z drabinką rezystorową (styl telefoniczny/CTIA) | Dedykowane, dyskretne przewody (styl radiotelefonu/PTT) |
|---|---|---|
| Mechanizm | PTT/Vol+/Vol− każdy zwiera inny rezystor między linią MIC a masą — różne napięcie DC na wspólnej linii MIC, nałożone na sygnał audio | Każdy przycisk to osobny przewód, zwierany do masy niezależnie |
| Wykrywanie | 1 kanał ADC STM32 czyta napięcie DC na linii MIC (przez separujący filtr od sygnału audio), progi rozróżniają przyciski | Zwykłe GPIO z pull-up, stan cyfrowy wprost |
| Zalety | tani, powszechnie dostępny standard (zwykłe słuchawki z pilotem) | bardziej niezawodne w hałaśliwym/przemysłowym środowisku, jednoznaczny stan cyfrowy |
| Wady | progi napięciowe różnią się między producentami/modelami (brak jednego uniwersalnego standardu CTIA/OMTP) — wymaga kalibracji | niestandardowe złącze/kabel, mniej dostępnych gotowych akcesoriów |

**Decyzja projektowa: zaprojektować STM32 tak, by obsługiwał oba warianty jednocześnie**, nie
wybierać jednego na sztywno:
- **1 kanał ADC** STM32 podłączony (przez filtr/dzielnik dopasowujący zakres) do linii MIC
  gniazda TRRS — obsługuje wariant z drabinką rezystorową; progi napięciowe **konfigurowalne
  przez rejestr I2C** (nie zaszyte na sztywno w firmware), bo różnią się między modelami
  headsetów.
- **3 zapasowe GPIO** STM32 z wewnętrznym pull-up — obsługują wariant z dyskretnymi przewodami
  (PTT/Vol+/Vol− każdy osobno do masy) **albo**, jeśli finalnie wybrany zostanie wariant TRRS,
  służą jako "inne dodatkowe przyciski poza zakresem LRADC" z pierwotnego wymogu.
- **1 GPIO** dla detekcji wpięcia gniazda (jack-detect, styk mechaniczny w gnieździe).
- **Debounce**: realizowany w firmware STM32 (licznik/timer, rząd 20-50ms), nie sprzętowo —
  nie wymaga dodatkowych RC na płytce.

**Proponowana mapa rejestrów I2C** (własny protokół tego projektu, nie z żadnego datasheetu —
firmware STM32 pisany od zera w tym projekcie):

| Rejestr | Nazwa | Opis |
|---|---|---|
| `0x00` | CHIP_ID/VERSION | RO, identyfikacja + wersja firmware STM32 |
| `0x01` | STATUS/EVENT | bit0 PTT, bit1 Vol+, bit2 Vol−, bit3-5 AUX1-3, bit6 headset_inserted, bit7 rezerwa — czyszczone przy odczycie (read-to-clear) |
| `0x02-03` | RAW_ADC | 16-bit, surowy odczyt linii MIC — do kalibracji progów z poziomu hosta, nie tylko debugowania |
| `0x04` | CONFIG | progi ADC dla PTT/Vol+/Vol− w trybie drabinki rezystorowej + czas debounce — konfigurowalne z V3S, nie zaszyte na sztywno w STM32 |

IRQ (PB6, patrz 11.2) sygnalizuje zmianę `STATUS` — host (V3S) czyta przez I2C i zeruje flagę.

**Strona V3S/Linux**: **bez własnego kernel drivera** — odczyt przez `/dev/i2c-X` z poziomu
userspace, dokładnie ten sam wzorzec co już przyjęty w projekcie dla PWRON/dotyku (evdev z
poziomu aplikacji, nie kernel driver pisany od zera). Przerwanie (PB6) budzi proces przez linię
GPIO skonfigurowaną jako edge-triggered w `/dev/gpiochipN` (libgpiod), zamiast pollingu I2C.

### 11.4 Mainline/ryzyko

STM32 to niestandardowy koprocesor z **własnym firmware pisanym w tym projekcie od zera** (nie
gotowy chip peryferyjny z driverem w mainline) — w odróżnieniu od reszty BOM (AXP209, TSC2007,
MCP2518FD, itd.), gdzie sterowniki są gotowe. Realna praca: firmware STM32 (HAL/LL albo
Zephyr — do wyboru osobno) + mały fragment kodu userspace po stronie V3S (I2C + GPIO IRQ,
podobny rozmiar do już istniejącego kodu obsługi dotyku/PWRON). Ryzyko niskie technicznie
(I2C slave + ADC + GPIO to standardowe, dobrze udokumentowane peryferia STM32), ale to
dodatkowy, samodzielny projekt firmware do napisania i przetestowania, nie "podłącz i działa"
jak większość reszty tego dokumentu.

---

## Przerwania (EINT) i magistrale I2C — analiza konfliktów

Poniższe oparte jest na dokładnym pinout V3S z `pcb/local_lib/V3S_AXP209.kicad_sym` (to
transkrypcja pinów krzemu, nie stan narysowanego schematu — traktowane tu jako źródło
referencyjne, tak samo jak `doc/V3S_CDR_STD_V1_0_20150514.pdf`).

**Peryferia wymagające przerwania od hosta:**

| Peryferium | Sygnał | Dlaczego potrzebne |
|---|---|---|
| MCP2518FD (CAN) | pin **INT** (dedykowany, active-low) | `mcp251xfd` wymaga `interrupts` w DT — bez tego brak zdarzeniowej obsługi ramek |
| TSC2007 (dotyk) | nPENIRQ | standardowe zdarzeniowe wykrywanie dotknięcia (można pollować, ale IRQ jest właściwym podejściem) |
| AXP209 (PMIC) | pin **IRQ/WAKEUP** | `axp20x` multipleksuje przez ten jeden fizyczny pin wiele zdarzeń wewnętrznych (PEK short/long press, insert/remove ACIN, itd.) — **bez tego pinu przycisk zasilania z pkt. 1.4 nie zadziała**, to nie jest opcjonalne |
| STM32 (kontroler audio/przycisków, pkt. 11) | GPIO open-drain dedykowany | żądanie komunikacji I2C (zmiana `STATUS`: PTT/Vol+/Vol−/AUX/jack-detect) — analogicznie do IRQ AXP209, jeden fizyczny pin multipleksuje wiele zdarzeń wewnętrznych STM32 |

Cztery peryferia, cztery przerwania (od dodania STM32, pkt. 11). LRADC (klawisze, pkt. 4.3) i
EPHY (Ethernet, pkt. 8) mają własne wewnętrzne linie przerwań podpięte bezpośrednio do GIC na
poziomie krzemu — **nie zużywają** budżetu pinów EINT poniżej.

**Piny EINT-capable w tym pakiecie**: wyłącznie **PB0-PB9** (`PB_EINT0-9`) i **PG0-PG5**
(`PG_EINT0-5`) — żaden inny port nie ma w tym pinout nazwanej funkcji EINT. PG0-PG5 są w
całości zajęte przez SD/microSD (pkt. 6), więc realny budżet przerwań to wyłącznie Port B.

**Dokładny rozkład Portu B** (skorygowane względem wcześniejszego, mniej precyzyjnego opisu —
UART2, PWM0/1 i TWI0 mają w rzeczywistości **osobne, nienachodzące na siebie piny**, nie
współdzielony blok):

| Pin | Funkcje alternatywne | Przydział w tym projekcie |
|---|---|---|
| PB0 | UART2_TX / PB_EINT0 | UART2_TX → GPS |
| PB1 | UART2_RX / PB_EINT1 | UART2_RX → GPS |
| PB2 | UART2_RTS / PB_EINT2 | **wolny → EINT: MCP2518FD INT** (RTS niepotrzebny dla GPS) |
| PB3 | UART2_CTS / PB_EINT3 | **wolny → EINT: TSC2007 nPENIRQ** (CTS niepotrzebny dla GPS) |
| PB4 | PWM0 / PB_EINT4 | PWM0 → EN podświetlenia (PT4101) |
| PB5 | PWM1 / PB_EINT5 | **wolny → EINT: AXP209 IRQ/WAKEUP** (PWM1 niepotrzebny, jeden kanał PWM wystarcza na backlight) |
| PB6 | TWI0_SCK / PB_EINT6 | **EINT: STM32 IRQ** (kontroler audio/przycisków, pkt. 11) — TWI0 nadal nieużywany jako magistrala, pin wykorzystany tylko jako EINT |
| PB7 | TWI0_SDA / PB_EINT7 | **wolny — ostatnia rezerwa EINT** (TWI0 nieużywany) |
| PB8 | TWI1_SCK / UART0_TX / PB_EINT8 | UART0_TX → debug |
| PB9 | TWI1_SDA / UART0_RX / PB_EINT9 | UART0_RX → debug |

**Odpowiedź wprost: czy jest konflikt UART0 / I2C?** Na pinach PB8/PB9 — **technicznie tak**:
to te same 2 piny co TWI1 (TWI1_SCK/TWI1_SDA), jedna funkcja na raz. Nie jest to jednak problem
w praktyce: TWI1 ma **drugą, całkiem niezależną lokalizację** na `PE21/PE22`
(`CSI_SCK/TWI1_SCK/UART1_TX` i `CSI_SDA/TWI1_SDA/UART1_RX`) — te piny nie są używane przez LCD
(sygnały LCD_D2-D23 leżą na innych pinach Portu E, PE21/PE22 są poza tą grupą) ani przez nic
innego w tym projekcie (brak kamery CSI, brak UART1).

**Konsolidacja I2C na jedną magistralę** — słuszna uwaga: AXP209 (adres 0x34) i TSC2007
(0x48/0x49) nie kolidują adresowo, więc obydwa mogą siedzieć na **jednej** magistrali zamiast
dwóch. Kluczowe jest to, **którą** magistralę/lokalizację się wybiera:

- Gdyby wspólną magistralę poprowadzić przez **TWI0 (PB6/PB7)** — piny te i tak są
  EINT-capable (`PB_EINT6/7`), więc zostałyby zajęte przez I2C niezależnie od tego, czy obsługuje
  jedno urządzenie czy dwa. **Zero zysku** w budżecie EINT.
- Prowadząc wspólną magistralę zamiast tego przez **TWI1 (PE21/PE22)** — Port E **nie ma w
  ogóle** funkcji EINT (patrz wyżej), więc nic tam nie jest "tracone". Efekt: **PB6/PB7
  zostają całkowicie wolne** i to one, będąc EINT-capable, realnie powiększają pulę rezerwy.

**Decyzja: jedna magistrala, TWI1 na PE21/PE22, wszystkie trzy urządzenia (AXP209 + TSC2007 +
STM32, pkt. 11) na niej** — to jest sposób na uzyskanie dodatkowej rezerwy EINT, o który
pytałeś (nadal aktualne po dodaniu STM32, bo Port E nadal nie ma EINT — trzecie urządzenie na
TWI1 nic nie kosztuje w budżecie przerwań). TWI0 (PB6/PB7) jako **magistrala** zostaje
całkowicie nieużyty w tym projekcie — PB6 wykorzystany jednak jako zwykły EINT (IRQ STM32, pkt.
11.2), zostawiając **PB7 jako ostatnią rezerwę** (np. card-detect SD czy drugie przerwanie z
MCP2518FD, gdyby zaszła taka potrzeba). UART0 (PB8/PB9) nadal nie koliduje z niczym — TWI1
fizycznie stoi na PE21/22, nie na PB8/9.

**Kompromis do świadomego zaakceptowania**: jedna wspólna magistrala oznacza, że zawis/glitch
jednego urządzenia (np. TSC2007 przy ESD na panelu dotykowym — realny, znany tryb awarii
tanich kontrolerów rezystancyjnych) może chwilowo zablokować też komunikację z **pozostałymi
dwoma** urządzeniami (AXP209: odczyt baterii, zdarzenia PEK; STM32: stan przycisków), dopóki
magistrala się nie odzyska — ryzyko rośnie z liczbą urządzeń na wspólnej magistrali (teraz
trzy, nie dwa). Mitygacja jest standardowa i już dostępna w mainline: sterownik I2C na V3S
(`sun6i-i2c`/kontroler TWI) ma timeout na transakcję, a jądro Linux ma ogólny mechanizm
odzyskiwania zawieszonej magistrali I2C (bit-banging SCL do odblokowania SDA) — nie jest to
"wszystko martwe do rebootu", tylko chwilowe opóźnienie do czasu recovery. Jeśli w praktyce
(bring-up) okaże się to realnym problemem, rozdział na dwie magistrale (np. TWI0+TWI1) jest
możliwy bez zmiany reszty przydziału pinów (koszt: utrata ostatniej rezerwy EINT z powrotem).

## Tabela alokacji zasobów V3S (podsumowanie sekcji 1-10)

| Zasób | Przeznaczenie | Uwagi |
|---|---|---|
| SDC0 (PF0-PF5) | microSD systemowa (gniazdo z pokrywką blokującą), nie eMMC — decyzja 2026-07-24 | wyklucza JTAG i UART0(PF) na tych samych pinach |
| SDC1 (PG0-PG5) | SD/microSD wymienna (dane) | poza listą bootowalnych urządzeń BROM (celowo); zajmuje cały PG_EINT0-5 |
| SPI0 (SDC2-pins, PC0-PC3) | MCP2518FD (CAN) | SDC2 jako MMC nieużywany w tym projekcie |
| UART0 (PB8/PB9) | debug console | druga lokalizacja (PF zajęte przez microSD systemową); bez konfliktu z TWI1, bo TWI1 przeniesiony na PE21/22 |
| UART2 TX/RX (PB0/PB1) | GPS MAX-M10S | RTS/CTS (PB2/PB3) świadomie NC — uwalnia je pod EINT |
| PB2 (EINT) | IRQ: MCP2518FD (pin INT) | |
| PB3 (EINT) | IRQ: TSC2007 (nPENIRQ) | |
| PB4 (PWM0) | EN podświetlenia (PT4101) | |
| PB5 (EINT) | IRQ: AXP209 (IRQ/WAKEUP) | PWM1 na tym pinie nieużywany — jeden kanał PWM wystarcza |
| PB6 (EINT) | IRQ: STM32 (kontroler audio/przycisków, pkt. 11) | TWI0 nadal nieużywany jako magistrala; pin wykorzystany tylko jako EINT |
| PB7 (EINT) | **wolny — ostatnia rezerwa** | TWI0 nieużywany jako magistrala |
| TWI1 (PE21/PE22) | I2C → **AXP209 + TSC2007 + STM32** (wspólna magistrala, pkt. 11) | świadomie nie na PB8/9, żeby nie kolidować z UART0; Port E nie ma EINT, więc nic tu nie tracimy |
| STM32 (kontroler audio/przycisków) | Headset PTT/Vol+/Vol−, jack-detect, przyciski dodatkowe poza LRADC | pkt. 11; STM32L432KCU6 (QFN-32), zasilanie z VCC-IO 3.3V |
| LRADC0 (+LRADC1?) | klawisze funkcyjne | potwierdzić liczbę kanałów w datasheet; własna linia IRQ do GIC, nie zużywa EINT |
| USB OTG | FEL / debug gadget | device-only; D+/D-/GND + VBUS tylko do wykrywania kabla (AXP209 insert/remove IRQ, `N_VBUSEN` nieaktywny) — decyzja 0.1 |
| GPIO AXP209 (GPIO0/1) | ADC pomiaru Vin 8-40V | zakres wejściowy ADC ~0.7-2.8V (zweryfikować `REG85H`) |
| AXP209 BACKUP | LIR2032, zasila LDO1 → VCC-RTC V3S (wbudowany RTC) | musi być realnie podłączony, nie NC; MCP79410 nieużywany — decyzja 0.3 |
| AXP209 PEK/PWRON | przycisk zasilania | `axp20x-pek`, mainline |
| VCC-RTC V3S (ball 98) | RTC wbudowany, zasilany z LDO1 AXP209 | patrz pkt. 1.2; MCP79410 rozważony, nieużyty (decyzja 0.3) |
| GPIO wolny (np. jeden z PB6/PB7 rezerwy, lub inny wolny PIO poza tabelą EINT) | opcjonalnie: SHDN chipu MAX2674/76 (opcja 7.3) lub sterowanie RF switch anteny GPS (opcja zapasowa 7.4) | nieobowiązkowe — patrz pkt. 7; opcja 7.3 (rekomendowana) w ogóle nie wymaga GPIO do samej decyzji przełączania |

Budżet EINT: 4 przerwania obsadzone (PB2/PB3/PB5/PB6 — ostatni od dodania STM32, pkt. 11) +
**1 w rezerwie** (PB7) — starczy jeszcze np. na card-detect SD, gdyby zaszła taka potrzeba, ale
budżet jest już prawie wyczerpany; kolejne przerwanie wymagałoby albo rozdziału TWI0/TWI1 z
powrotem (odzyskanie PB6 kosztem powrotu do dwóch magistrali), albo przeniesienia funkcji na
GPIO expander/STM32 (który już i tak siedzi na I2C, pkt. 11).

---

## Mainline Linux — zbiorcze podsumowanie ryzyka

| Podsystem | Sterownik | Ryzyko |
|---|---|---|
| V3S core, DDR, EPHY | mainline sun8i-v3s | **brak** — już potwierdzone na sprzęcie w tym projekcie |
| Display (DE2+TCON, fbdev) | `sun8i-v3s-display-engine`/`-tcon` | **brak** — już potwierdzone na sprzęcie (panel 5" 800×480) |
| AXP209 (mfd/regulator/pek/adc/power_supply/gpio) | `axp20x-*` | **brak** — jeden z najdojrzalszych PMIC-ów w mainline |
| MCP2518FD (CAN-FD SPI) | `mcp251xfd` | **brak** — dojrzały, szeroko używany, zweryfikowane bezpośrednio na tagu v6.12 |
| TSC2007 (dotyk I2C) | `tsc2007` | **brak** — prosty, dojrzały |
| RTC wbudowany V3S (wybrany, decyzja 0.3) | `rtc-sun6i` (`allwinner,sun8i-v3-rtc`) | **brak** — zweryfikowane na tagu v6.12, pełny driver |
| RTC MCP79410 (rozważony, nieużyty — patrz 0.3) | `rtc-ds1307` (`microchip,mcp7941x`) | **brak** — uniwersalny, dojrzały driver, nieaktualne dla tego BOM |
| LRADC keys | `sun4i-lradc-keys` | **niskie** — potwierdzić liczbę kanałów wyprowadzonych w tym pakiecie |
| microSD ×2 (systemowa SDC0 + danych SDC1) | `sunxi-mmc` | **brak** — ten sam driver co dla eMMC, bez zmian software'owych po odrzuceniu eMMC (pkt. 6) |
| GPS MAX-M10S | brak potrzeby (userspace NMEA/UBX) | **brak** |
| Antena GPS: MAX2674/MAX2676 (RF switch+LNA+bias) lub PE4259 (fallback) | brak potrzeby (czysto analogowe) | **brak** — ale dostępność MAX2674/76 do potwierdzenia przy BOM (pkt. 7.3) |
| ATA6561, TPS61240, PT4101, SY8088 | brak potrzeby (czysto analogowe/fixed) | **brak** |
| USB OTG (FEL/gadget) | `musb` sunxi + BROM FEL | **brak** |
| A/B updates (RAUC) | `BR2_PACKAGE_RAUC` (Buildroot) + U-Boot bootcount/altbootcmd | **brak** — potwierdzony pakiet Buildroot, ugruntowany mechanizm |
| Audio (kodek wbudowany V3S) | ALSA (`sun8i-codec`/`sun4i-i2s` rodzina, do potwierdzenia dokładny compatible dla V3S) | **niskie, niepotwierdzone tu wprost** — kodek na tej rodzinie SoC generalnie wspierany w mainline, ale nie zweryfikowane w tej sesji dla konkretnie V3S; patrz pkt. 11.1 i "Otwarte tematy" |
| STM32 (kontroler audio/przycisków, pkt. 11) | brak potrzeby kernel drivera — odczyt userspace przez `/dev/i2c-X` + GPIO IRQ (`/dev/gpiochipN`) | **brak ryzyka mainline** (nic nowego w kernelu), ale **cały firmware STM32 pisany od zera** w tym projekcie — patrz pkt. 11.4 |

Cały projekt siedzi bardzo dobrze w mainline — **żaden blok nie wymaga out-of-tree
patcha ani vendor kernela**, poza jednym wyjątkiem historycznym opisanym w "Dodatku: migracja
kernela" (driver dotyku NS2009 na *obecnej* płycie deweloperskiej, nie na terminalu — tam
używamy TSC2007, w pełni mainline). To istotnie różni się od wielu innych tanich SoC IPC,
gdzie peryferia bywają wspierane tylko w kernelu producenta.

---

## Decyzje (stan na 2026-07-23, uzupełnione przy scaleniu 2026-07-24)

1. **Pojemność LiPo** — **później**. Metoda liczenia w pkt. 1.1 gotowa, do zastosowania po
   finalizacji obudowy.
2. **Złącze zasilania 8-40V** — **M12 5-pin, łączone z CAN** (nie osobne złącza), pinout
   wewnętrzny ostateczny: 1-VCC, 2-GND, 3-CAN_H, 4-CAN_L, 5-EARTH/PE — zob. pkt. 2.
3. **IC buck 24V→5V** — **TPS54360B**, potwierdzone.
4. **5" czy 7" LCD** — **5"**, panel już posiadany (BL050S061-17/T050SWV012T, zob. pkt. 4.1).
5. **Ogniwo BACKUP** — **LIR2032**, zob. analiza bezpieczeństwa w pkt. 1.2. Konkretny
   model/pojemność (mAh) wciąż otwarte — patrz "Otwarte tematy".
6. **Izolowany CAN transceiver** — **nieizolowany** (ATA6561), potwierdzone; dodatkowo
   wzmocnione przez to, że złącze M12 (decyzja 2) fizycznie wymusza wspólną masę CAN/zasilanie.
7. **Złącze portu debug/FEL (pkt. 9)** — **JST**, wyłącznie do celów serwisowych/debug (nie
   panelowe gniazdo USB-A/Micro/C) — pasuje do niskiego priorytetu tego punktu z briefu.
8. **Liczba kanałów CAN** — **jeden**. Jeśli w szkicu KiCad istnieje druga instancja ATA6561,
   traktować jako coś do usunięcia/wyjaśnienia przy najbliższym porządkowaniu schematu, nie
   jako zamierzoną redundancję.
9. **Kontroler CAN** — **MCP2518FD** (nie MCP2515/MCP25625/kombo), decyzja architektoniczna:
   osobny kontroler + osobny transceiver. Pełne uzasadnienie i historia w pkt. 5.5.
10. **Baza kernela** — **Linux 6.12.95 LTS** (mainline `buildroot-mainline`), zmigrowana i
    potwierdzona na sprzęcie (LicheePi Zero) 2026-07-19, przed rozpoczęciem prac nad
    terminalem — patrz "Dodatek: migracja kernela". Terminal startuje od razu z tej bazy, nie
    z historycznego forka 5.2.y.
11. **Aktualizacje systemu** — schemat **A/B przez RAUC** + mechanizm U-Boot `bootcount` —
    patrz "Dodatek: aktualizacje systemu — schemat A/B" (pojemność nośnika zaktualizowana pod
    microSD w decyzji 15 niżej, eMMC już nieaktualne).
12. **VBUS AXP209 / port USB debug** — **rozdzielone od zasilania systemu, ale podłączone
    wyłącznie do wykrywania obecności kabla** (`N_VBUSEN` nieaktywny, insert/remove IRQ na
    AXP209) — rozstrzyga sprzeczność 0.1. Zob. pkt. 1.3 i 9.
13. **Transceiver CAN** — **ATA6561 potwierdzone**, bez wymogu certyfikowanej rodziny
    FD-rated — rozstrzyga sprzeczność 0.2. Zob. pkt. 5.1.
14. **RTC** — **wbudowany RTC V3S** (VCC-RTC z LDO1 AXP209), MCP79410 nieużyty w BOM (niższy
    koszt, mniej routingu) — rozstrzyga sprzeczność 0.3. Zob. pkt. 1.2.
15. **Pamięć systemowa — microSD zamiast eMMC** (2026-07-24) — eMMC odrzucone (trudna
    dostępność, ok. 2-4× wyższa cena). Zamiast tego: gniazdo microSD z **pokrywką blokującą**
    na SDC0 (system + aplikacja) + osobna karta SD/microSD wymienna na SDC1 (dane) — ten sam
    logiczny podział ról jak wcześniej, tylko inna realizacja fizyczna. Zob. pkt. 6.
16. **GPS — MAX-M10S + dwie anteny** (2026-07-24) — moduł zmieniony z MAX-M10C na MAX-M10S;
    dodana antena wewnętrzna (pasywna, domyślna) obok zewnętrznej (aktywna, SMA+pigtail
    U.FL), z automatycznym przełączaniem przez dedykowany chip **MAX2674/MAX2676** (rekomendacja
    główna) lub RF switch PE4259 sterowany z wolnego GPIO + odczyt UBX-MON-RF (fallback).
    Switched-SMA z pierwotnej rekomendacji odrzucone — niedostępne w formie pigtail. Zob.
    pkt. 7.
17. **Audio + kontroler przycisków STM32** (2026-07-25) — płyta ma też posłużyć do budowy
    innego urządzenia wymagającego dźwięku (słuchawki+mikrofon) i obsługi przycisków headsetu
    (PTT/Vol+/Vol−) + dodatkowych przycisków poza zakresem LRADC. Audio: aktywacja wbudowanego
    kodeka V3S (piny wcześniej nieużywane). Przyciski: dodany **STM32L432KCU6** (QFN-32) jako
    koprocesor I2C na współdzielonej magistrali TWI1 + EINT na **PB6** (ostatnia rezerwa EINT
    spada do jednego pinu, PB7). Zob. pkt. 11.

Wszystkie trzy sprzeczności z sekcji 0 oraz trzy dodatkowe zmiany decyzji (15-17) zostały
wprowadzone przez użytkownika 2026-07-24/25 — sekcja 0 zachowuje historię wcześniejszych
stanowisk dla kontekstu.

---

## Dodatek: migracja kernela 5.2.y → 6.12.95 LTS

*(Treść w całości z notatnika 2026-07-19, punkt 6 — zrobiona i potwierdzona na sprzęcie przed
otwarciem prac nad terminalem; zachowana tu w pełni, bo dokument decyzji 2026-07-23 zakłada
tę bazę kernela jako już gotową, nie tłumacząc skąd się wzięła.)*

**Problem**: pierwotny kernel projektu to fork Lichee-Pi, branch `zero-5.2.y` (Linux 5.2,
połowa 2019). Pytanie: czy da się to podnieść bez utraty działającego LCD/dotyku, i czy stary
kernel to problem bezpieczeństwa, zwłaszcza przy ewentualnym Ethernecie.

**Ustalenia (sprawdzone bezpośrednio w źródłach mainline Linux, nie z pamięci)**:
- Linux 5.2 to release **nie-LTS** — ostatnia wersja 5.2.21 wyszła w październiku 2019,
  ok. 4 miesiące wsparcia. Od tego czasu (stan na 2026-07-19: **ponad 6.5 roku**) zero
  backportów bezpieczeństwa.
- Powód użycia forka `zero-5.2.y` (a nie `zero-4.13.y`) opisany w `PRZEWODNIK_BUDOWY.md`:
  pusty `sun8i_v3s_quirks` bez flagi `has_channel_0` w `sun4i_tcon.c` na starszej gałęzi
  Lichee-Pi. **Sprawdzone w aktualnym mainline** (`drivers/gpu/drm/sun4i/sun4i_tcon.c`,
  torvalds/linux): `sun8i_v3s_quirks` ma tam `.has_channel_0 = true` — poprawka już jest
  upstream. To był najpewniej artefakt wczesnej/niekompletnej gałęzi w repo Lichee-Pi, nie
  realna luka w mainline.
- Driver dotyku **NS2009** — **korekta po realnym sprawdzeniu przy migracji**: patch
  dodający `ns2009.c` został zgłoszony do mainline w 2017 razem z bindingiem DT, ale sam
  **sterownik nigdy nie został scalony** — sprawdzone bezpośrednio w drzewie źródeł
  (`git ls-tree` na tagu 6.12.95 i na aktualnym `master`, `drivers/input/touchscreen/ns2009.c`
  nie istnieje w żadnym z nich). To co jest w mainline to tylko sam generyczny binding
  `touchscreen_parse_properties()`, którego sterownik używa — sam plik sterownika trzeba
  było przenieść ręcznie jako patch, uwspółcześniony do dzisiejszego API kernela (stary
  `input-polldev`, usunięty z mainline, zastąpiony `input_setup_polling()`). Szczegóły w
  `PRZEWODNIK_BUDOWY.md` pkt 12.5. **To jedyne miejsce w tej analizie, gdzie oryginalne
  ustalenie okazało się błędne** — TCON quirk faktycznie był już naprawiony w mainline,
  sterownik dotyku nie. Nie zmienia to końcowego wniosku: żadna z tych dwóch rzeczy nie
  okazała się blokerem, obie dały się przenieść bez odtwarzania oryginalnego bringupu od
  zera.

**Co realnie zostało do zrobienia przy migracji** (niezerowa praca, ale dużo mniejsza niż
oryginalny bringup): port węzła DT dla LCD (timing panelu, backlight, NS2009 na I2C, LRADC
keypad) pod konwencje nowszego jądra; ponowna weryfikacja tricku z odpinaniem vtcon1
(workaround na korupcję fbcon); U-Boot prawdopodobnie bez zmian (już poprawnie przekazuje
framebuffer przez `simplefb`).

**Wersja wybrana**: LTS **6.12.95** (konkretny tag, nie "6.12 lub 6.6 ogólnie" — wybrany m.in.
dlatego że miał już ~20 miesięcy realnej eksploatacji na starych SBC na 2026-07, mniej
regresji niż świeższy 6.18).

**Efekt uboczny dla CAN (sekcja 5)**: przy 6.12/6.6 znika problem backportu sterowników CAN
FD (`mcp25xxfd` od ~5.9/5.10, `tcan4x5x` od 5.4) — oba już wbudowane. Dodatkowy argument za
zrobieniem tego upgrade'u przed projektowaniem CAN front-endu terminala.

**Bezpieczeństwo**: obawa zasadna. Bez Ethernetu ryzyko ograniczone (głównie
lokalne/fizyczne wektory), ale **z Ethernetem realnie rośnie powierzchnia ataku** — pełny
stos TCP/IP, netfilter, wszystko sieciowe w kernelu miało 6.5 roku nienaprawionych CVE na
starym forku.

### Wynik: zrobione i potwierdzone na sprzęcie (2026-07-19)

Migracja wykonana dokładnie jak zaplanowano powyżej — na LicheePi Zero, w osobnym drzewie
Buildroota (`~/lichee/V3S/buildroot-mainline`, oryginalne `~/lichee/V3S/buildroot` zostało
nietknięte jako punkt odwrotu), równolegle do prac nad terminalem. **Pełny opis techniczny:
`PRZEWODNIK_BUDOWY.md`, sekcja 12.**

- **Wersja: 6.12.95** (LTS).
- **LCD (timing panelu, tcon0) i U-Boot** — bez niespodzianek, zgodnie z przewidywaniem.
  U-Boot faktycznie bez zmian.
- **Sterownik dotyku NS2009** — jedyne miejsce gdzie oryginalne ustalenie było błędne (patrz
  korekta wyżej) — wymagał realnego portu, nie tylko podpięcia DT.
- **Dwie rzeczy odkryte dopiero na sprzęcie, nieprzewidziane wcześniej**: (1) biały ekran po
  starcie mimo poprawnie zbindowanego `sun4i-drm` — brakująca `CONFIG_DRM_FBDEV_EMULATION` w
  bazowym `sunxi_defconfig` (nie ma tego problemu przy `multi_v7_defconfig`, ale ten jest za
  duży/zbyt ogólny jak na ten cel); (2) busybox nie kompilował apletu `tc` przeciw nowym
  nagłówkom (stare uapi struktury CBQ usunięte z mainline) — nieużywane, wyłączone jednym
  wpisem w konfiguracji busyboksa.
- Trick z odpinaniem `vtcon1` (workaround na korupcję fbcon) **zostawiony bez zmian, nie
  przetestowany osobno pod kątem czy nadal jest potrzebny** — działa, więc nie było powodu
  ryzykować regresji tylko żeby to sprawdzić; **wciąż otwarte pytanie na przyszłość** (patrz
  "Otwarte tematy").
- **Ethernet włączony po raz pierwszy w ogóle** (na starym 5.2.y nigdy nie był aktywowany w
  DT) — celowo hotplug/nieblokujący start (`ip link up` + `udhcpc -b` w tle), zgodnie z
  wymaganiem żeby terminal polowy nie czekał na kabel przy starcie. Potwierdzone realnym
  DHCP + odpowiedzią na ping z zewnętrznej maszyny.
- **Efekt uboczny dla CAN teraz potwierdzony, nie tylko teoretyczny**: ten sam
  `buildroot-mainline` ma już dostępne `mcp25xxfd`/`tcan4x5x` (CAN FD) — patrz sekcja 5.5.

---

## Dodatek: aktualizacje systemu — schemat A/B (RAUC)

*(Treść w całości z notatnika 2026-07-19, punkt 8 — temat nieobecny w dokumencie decyzji
2026-07-23, uzupełnia sekcję 6 (microSD systemowa + SD danych, dawniej eMMC+SD) o warstwę
softwarową aktualizacji. Zaktualizowane 2026-07-24 pod zmianę decyzji eMMC→microSD, patrz
decyzja 15 — logika A/B i RAUC opisana niżej jest niezależna od tego, czy nośnik jest
lutowany czy w gnieździe, zmienia się tylko pojemność/wytrzymałość nośnika.)*

**Problem**: co daje podział A/B, czy pozwala na aktualizację bez wgrywania na kartę SD, ile
pamięci potrzeba, czy pozwala na aktualizację przez Ethernet, wady/zalety. Sprawdzone na
ugruntowanych mechanizmach: mainline U-Boot
(`bootcount`/`upgrade_available`/`bootlimit`/`altbootcmd`) + RAUC.

**Co daje**: atomowa, bezpieczna aktualizacja. Nowa wersja pisana do NIEaktywnego slotu karty
systemowej (SDC0) podczas gdy urządzenie normalnie działa na aktywnym (zero przestoju w
trakcie zapisu). Po zapisie bootloader przełącza aktywny slot i robi reboot. Jeśli nowy slot
nie wstanie poprawnie (limit nieudanych bootów), **automatyczny rollback** do poprzedniego,
sprawdzonego slotu — urządzenia w polu nie da się łatwo zcegłować nieudaną aktualizacją,
nawet przy zaniku zasilania w trakcie.

**Karta SD (danych, SDC1) niepotrzebna do rutynowych aktualizacji**: paczka aktualizacji
trafia dowolnym kanałem (Ethernet, USB, karta SD tylko jako nośnik pliku) i jest zapisywana
na karcie systemowej (SDC0) przez sam działający system — bez zewnętrznego przeflashowania z
PC (choć od decyzji 15, wyjęcie i zewnętrzne zaprogramowanie karty systemowej jest teraz też
możliwe jako opcja recovery, patrz pkt. 9).

**Ethernet**: tak, to główny use case tego mechanizmu (zob. sekcja 8 głównego dokumentu dla
statusu Ethernetu — potwierdzony na sprzęcie). Narzędzie: **RAUC** — potwierdzony pakiet
Buildroot (`BR2_PACKAGE_RAUC`, od Buildroot 2017.08), zbudowany na standardowym mechanizmie
U-Boot (`upgrade_available`/`bootcount`/`bootlimit`/`altbootcmd`). Dokłada format podpisanych
paczek, zarządzanie slotami, instalację przez HTTP(S) (`rauc install
http://.../bundle.raucb`) albo lokalnie.

**Pamięć — czy karta 4GB wystarczy**: policzone realnie —
- obecny obraz CanSensorHub: 69MB całości (kernel+rootfs+app),
- z zapasem na cięższy mainline 6.12.95 + większy rootfs terminala: ~150MB/slot,
- 2 sloty (A+B): ~300MB,
- osobna, **WSPÓLNA** (nieduplikowana) partycja danych (logi/CSV/kalibracja/config) — **musi
  żyć poza wymianą A/B**, inaczej aktualizacja kasuje kalibrację/logi. Nawet szczodrze:
  500MB-1GB,
- U-Boot + margines: kilkanaście MB.

Razem ~1-1.5GB zajęte → **karta 4GB+ daje 2.5-3GB+ zapasu, więcej niż wystarczająco** — ta
kalkulacja nie zależy od tego, czy nośnik to eMMC czy microSD, tylko od pojemności. **Zmiana
2026-07-24 (decyzja 15)**: nośnik systemowy to teraz microSD w gnieździe z pokrywką blokującą,
nie lutowany eMMC — patrz pkt. 6 dla pełnego uzasadnienia (dostępność/cena) i kompromisu
(mniejsza odporność mechaniczna niż eMMC, częściowo zrekompensowana gniazdem z blokadą i
rekomendacją karty przemysłowej/wysokiej wytrzymałości ze względu na cykle zapisu generowane
właśnie przez aktualizacje A/B). Dodatkowa furtka awaryjna niezależna od A/B: tryb FEL
Allwinnera (boot przez USB od hosta, patrz sekcja 9) jako sprzętowy recovery path — teraz
drugorzędny, bo kartę systemową da się też wyjąć i zaprogramować zewnętrznie (patrz pkt. 9).

**Wady**: realna praca wdrożeniowa (skrypty U-Boota, integracja RAUC, dyscyplina trzymania
danych poza slotami A/B, podpisywanie paczek przy aktualizacji sieciowej — wraca temat
bezpieczeństwa Ethernetu z "Dodatku: migracja kernela"), plus 2× miejsce na obraz systemu
(nieistotne przy 4GB+, patrz wyżej), plus niższa wytrzymałość zapisu karty microSD vs eMMC
(zaadresowane wyborem karty przemysłowej, patrz pkt. 6).

**Decyzja/rekomendacja**: microSD (4GB+, klasa przemysłowa/wysokiej wytrzymałości) + A/B przez
RAUC + mechanizm U-Boot bootcount. Kluczowy punkt projektowy do zapamiętania przy
partycjonowaniu: partycja danych (logi/kalibracja) musi być osobna i wspólna dla obu slotów,
nie duplikowana. **Wciąż otwarte** (patrz "Otwarte tematy"): dobór konkretnych rozmiarów
partycji, konfiguracja RAUC (bundle format, klucze do podpisywania), integracja skryptów
U-Boota z aktualnym `buildroot-mainline`, oraz konkretny model karty/gniazda (pkt. 6).

---

## Dodatek: panel 7" — porównanie 40-pin (obecny) vs 50-pin (np. AT070TN92)

*(Treść w całości z notatnika 2026-07-19, punkt 9 — kontekst dla ewentualnej przyszłej zmiany
na 7", odłożony bo sekcja 4.1 zdecydowała na razie o pozostaniu przy 5".)*

**Problem**: jaka jest różnica między obecnie używanym panelem 40-pin a panelem 50-pin typu
AT070TN92. Sprawdzone bezpośrednio w prawdziwym datasheecie Innolux AT070TN92 (złącze Hirose
FH12A-50S-0.5SH) oraz w datasheecie V3S (nie z pamięci).

**Rozdzielczość ta sama**: AT070TN92 to też 800×480, tylko fizycznie **7"** (vs obecne 5").
Siatka pikseli LVGL się nie zmienia, zmienia się tylko fizyczny rozmiar ekranu.

**Głębia koloru**: AT070TN92 niesie pełne **RGB888** (8 bit/kanał, R0-R7/G0-G7/B0-B7 = 24
linie danych). Sprawdzone: **V3S to udźwignie** — piny LCD_D0-LCD_D23 istnieją w datasheecie,
TCON ma tryb "8bit/3cycle RGB serial mode (RGB888)", Display Engine 2.0 wspiera pixel format
RGB888. Typowy mały panel 40-pin (jak obecny) to zwykle RGB666 (18-bit) — mniej linii danych,
stąd mniejsze złącze. Dla płaskiej UI (zakładki, przyciski, nie zdjęcia) różnica wizualnie
mało istotna — V3S ma sprzętowy dither RGB666→RGB888, banding już zamaskowany.

**Decydująca różnica — AT070TN92 to "surowy" panel TFT, nie "smart" moduł**. Host musi sam
wygenerować i **prawidłowo zsekwencjonować** dodatkowe szyny zasilające (wartości z
datasheetu, nie szacunkowe):

| Sygnał | Napięcie (typ.) |
|---|---|
| DVDD (logika) | 3.3V |
| AVDD (analog) | ~10.4V |
| VGH (gate ON) | ~16.0V |
| VGL (gate OFF) | ~-7.0V |

Wymagana kolejność włączania wprost z datasheetu: **DVDD → VGL → VGH → Data → Backlight**
(odwrotnie przy wyłączaniu). Typowe małe panele 40-pin (jak obecny BL050S061-17) mają
generator VGH/VGL zintegrowany na własnej taśmie FPC — host podaje tylko 3.3V + prosty
2-przewodowy backlight. AT070TN92 wymaga dedykowanego układu bias/sekwencjonowania (np.
rodzina TI TPS6516x) — realny dodatkowy komponent, koszt i miejsce na płytce, nie tylko
więcej ścieżek do routingu.

**Dodatkowo**:
- podświetlenie: ~9.9V @ 180mA (~1.8W) — potrzebny boost driver LED,
- złącze fizycznie inne (50-pin 0.5mm Hirose) — nowy footprint na płytce niezależnie od
  wyboru,
- dotyk: obecny 5" NS2009 nie pasuje fizycznie do 7" (ani docelowy TSC2007 terminala) —
  potrzebna nowa nakładka dotykowa dopasowana rozmiarem.

**Decyzja/rekomendacja**: AT070TN92 (i cała rodzina AT070TN9x) to inna *klasa* panelu, nie
"ten sam panel z większym złączem". Jeśli w przyszłości celem będzie głównie większy ekran
przy zachowanej prostocie zasilania, szukać 7" panelu ze **zintegrowanym gate driverem na
szkle** (jak obecny), nie surowego modułu wymagającego zewnętrznego bias — oszczędza cały
dodatkowy układ zasilania LCD.

---

## Otwarte tematy / do dalszego rozpracowania

Skonsolidowane z obu dokumentów źródłowych, z zaznaczeniem co już zostało rozstrzygnięte od
czasu notatnika 2026-07-19.

**Wciąż otwarte:**

- **Pojemność/model ogniwa LiPo głównego** — metoda liczenia gotowa (pkt. 1.1), do
  zastosowania po finalizacji obudowy/ergonomii terminala (też nieomówionej jeszcze).
- **Konkretny model/pojemność ogniwa na pinie BACKUP** (LIR2032 — chemia/rodzina
  potwierdzona, decyzja 5 — ale nie konkretny mAh/dostawca); teraz zasila wyłącznie RTC
  wbudowany V3S (decyzja 0.3/14, patrz niżej), więc wymiar podtrzymania do przeliczenia pod
  ten jeden odbiornik. Rozważana alternatywa **MS621FE-FL11E** (SMD, mniejszy footprint,
  kosztem ~7.3× krótszego podtrzymania) przeanalizowana w pkt. 1.2 (podsekcja "Analiza:
  rozważana zamiana LIR2032 → MS621FE-FL11E", 2026-07-30) — wciąż nierozstrzygnięte.
- **Weryfikacja realnego wsparcia suspend-to-RAM na sun8i-v3s** (pkt. 1.4) — niepewne dla tego
  SoC, nie sprawdzone w żadnym z dokumentów źródłowych.
- **EARTH/PE vs GND na złączu M12** (pkt. 2) — czy PE w ogóle musi się stykać z GND obwodu, do
  ustalenia przy layoucie.
- **Dokładne progi napięciowe ADC GPIO0/1 AXP209** (`REG85H`, pkt. 2) — do zweryfikowania w
  datasheecie przy projektowaniu dzielnika 8-40V.
- **Częstotliwość PWM podświetlenia PT4101** (pkt. 3) — do dobrania eksperymentalnie na
  prototypie.
- **Liczba kanałów LRADC faktycznie wyprowadzonych w tym pakiecie V3S** (pkt. 4.3) — do
  potwierdzenia w `doc/V3S_CDR_STD_V1_0_20150514.pdf`.
- **Dostępność/cena MAX2674 vs MAX2676 vs fallback PE4259** (pkt. 7.3/7.4) — do potwierdzenia
  przy BOM, w tym czy PE4259 (lub inny kandydat) przepuszcza DC bias jeśli fallback okaże się
  potrzebny.
- **Konkretny model anteny wewnętrznej pasywnej (GPS)** i konkretne złącze SMA panelowe +
  pigtail U.FL dla anteny zewnętrznej (pkt. 7) — do wyboru u dostawcy.
- **Konkretny model karty microSD systemowej (klasa przemysłowa) i gniazda z pokrywką
  blokującą** (pkt. 6) — do wyboru u dostawcy przy BOM.
- **Trick odpinania `vtcon1`** — działa na 6.12.95, ale nie sprawdzone osobno czy nadal
  potrzebne na tym kernelu, czy nowszy fbcon/DRM już nie ma tego problemu (patrz "Dodatek:
  migracja kernela").
- **Wdrożenie A/B (RAUC)** — dobór konkretnych rozmiarów partycji, konfiguracja RAUC (bundle
  format, klucze do podpisywania paczek), integracja skryptów U-Boota
  (`bootcount`/`altbootcmd`) z aktualnym `buildroot-mainline` (patrz "Dodatek: aktualizacje
  systemu").
- **Docelowa obudowa/ergonomia terminala** — nieomawiana jeszcze w żadnym z dokumentów
  źródłowych.
- **Panel 7"** — odłożone, nie blokujące (decyzja 4: 5" na v1). Jeśli temat wróci: szukać
  panelu ze zintegrowanym gate driverem, patrz "Dodatek: panel 7"".
- **Realna różnica cenowa MCP2518FD+ATA6561 vs MCP25625** przy finalizacji BOM (pkt. 5.5) —
  nieoceniona w żadnej z dotychczasowych sesji.
- **Fizyczny interfejs headsetu — TRRS z drabinką rezystorową czy dedykowane przewody** (pkt.
  11.3) — świadomie zaprojektowane uniwersalnie (STM32 obsługuje oba), ale nie potwierdzone z
  konkretnym produktem/headsetem.
- **Dokładne wartości komponentów aplikacyjnych kodeka audio V3S** (pkt. 11.1) — kondensatory
  sprzęgające MIC, dopasowanie HPVCCIN/HPVCCBP, bias mikrofonu — wymaga pełnego application
  circuit z datasheetu Allwinner dla sekcji audio, nie sprawdzone w tej sesji.
- **Dokładny compatible/driver ALSA dla kodeka wbudowanego V3S** (pkt. 11.1/11.4) — nie
  zweryfikowane wprost w mainline w tej sesji, tylko założone przez analogię do rodziny sun8i.
- **Adres I2C STM32** (pkt. 11.2) — dowolny wolny 7-bit adres, konkretna wartość do ustalenia
  przy pisaniu firmware (nie koliduje z AXP209 0x34 ani TSC2007 0x48/0x49).
- **Dokładne progi ADC drabinki rezystorowej headsetu** (pkt. 11.3) — zależą od konkretnego
  modelu headsetu, stąd konfigurowalne rejestrem I2C zamiast zaszyte na sztywno; do kalibracji
  przy dostępności realnego headsetu.

**Rozstrzygnięte od 2026-07-19 (nie trzeba już wracać)**:
- ~~Dobór konkretnej przetwornicy 24V→5V~~ — **TPS54360B**, decyzja 3.
- ~~Wybór ogniwa BACKUP: LIR2032 vs supercap~~ — **LIR2032** (chemia), decyzja 5.
- ~~CAN FD: backport sterowników~~ — **rozwiązane**, `mcp251xfd`/`tcan4x5x` potwierdzone na
  v6.12, patrz pkt. 5.5.
- ~~Upgrade kernela 5.2.y → LTS~~ — **zrobione i potwierdzone na sprzęcie**, 6.12.95, patrz
  "Dodatek: migracja kernela".
- ~~5" czy 7" LCD~~ — **5"**, decyzja 4.
- ~~AXP203 czy AXP209~~ — **AXP209**, pkt. 1.5.
- ~~Kontroler CAN: MCP2515/MCP25625/kombo czy osobny~~ — **MCP2518FD + osobny transceiver**,
  decyzja 9 / pkt. 5.5.
- ~~Transceiver CAN: ATA6561 vs rodzina FD-rated~~ — **ATA6561 potwierdzone**, decyzja 13,
  patrz pkt. 5.1 i sprzeczność 0.2.
- ~~VBUS AXP209 / arbitraż zasilania USB debug~~ — **rozdzielone od zasilania, podłączone
  tylko do wykrywania kabla**, decyzja 12, patrz pkt. 1.3/9 i sprzeczność 0.1.
- ~~RTC: wbudowany V3S vs MCP79410 vs oba~~ — **wbudowany RTC V3S**, MCP79410 nieużyty,
  decyzja 14, patrz pkt. 1.2 i sprzeczność 0.3.
- ~~eMMC vs microSD dla pamięci systemowej~~ — **microSD w gnieździe z pokrywką blokującą**,
  eMMC odrzucone (dostępność/cena), decyzja 15, patrz pkt. 6.
- ~~Jak przełączać antenę GPS wewnętrzna/zewnętrzna~~ — **automatycznie, chipem
  MAX2674/MAX2676** (fallback: RF switch PE4259 + GPIO + UBX-MON-RF), switched SMA odrzucone
  jako niedostępne w formie pigtail, decyzja 16, patrz pkt. 7.

---

## Plan realizacji projektu

**Faza 0 — domknięcie otwartych decyzji.** Większość rozstrzygnięta (lista wyżej), w tym
wszystkie trzy sprzeczności z sekcji 0 (VBUS/USB, transceiver CAN, RTC) — rozstrzygnięte
2026-07-24 (decyzje 12-14). Jedyna realnie otwarta pozycja to pojemność LiPo (decyzja 1)
zależna od obudowy — nie blokuje startu prac schematowych.

**Faza 1 — schematy (KiCad), kolejność wg zależności, nie wg numeracji briefu:**
1. Zasilanie rdzeniowe: AXP209 + backup battery (LDO1 → VCC-RTC V3S) + power-path ACIN-only
   + VBUS jako sygnał detekcji USB + PEK (sekcja 1, decyzje 12/14).
2. Wejście 8-40V → 5V + ochrona + dzielnik ADC (sekcja 2) — zależy od wyboru IC (decyzja 3).
3. SY8088 DDR + PT4101 backlight (sekcja 3).
4. Przydział pinów V3S wg tabeli alokacji wyżej (storage, UART, EINT, TWI0/TWI1) — zrobić to
   jako jeden przebieg, całościowo, tak jak już rozpisane w tym dokumencie.
5. CAN front-end: MCP2518FD + ATA6561 + terminacja + TPS61240 (sekcja 5, decyzja 13).
6. Pamięci: microSD systemowa z gniazdem blokowanym (SDC0) + SD danych (SDC1) (sekcja 6,
   decyzja 15).
7. GPS: MAX-M10S + dwie anteny (wewnętrzna/zewnętrzna) + MAX2674/76 lub fallback + backup
   (sekcja 7, decyzja 16).
8. Ethernet: magnetyka + RJ45 (sekcja 8) — najniższe ryzyko, można zrobić szybko.
9. USB debug/FEL (sekcja 9) + UART debug (sekcja 10).
10. LCD + dotyk I2C + klawisze LRADC (sekcja 4) — po decyzji 5"/7" (już podjętej: 5").
11. Audio (kodek V3S, gniazdo TRRS) + STM32 kontroler przycisków headsetu/dodatkowych
    (sekcja 11, decyzja 17) — zależy od pinów I2C/EINT już przydzielonych w kroku 4.

**Faza 2 — przegląd projektu przed layoutem.** Skorzystać z KiCad-owego skilla analitycznego
(`kicad-happy:kicad`) do przeglądu ERC, drzewa zasilania i spójności netlisty po zamknięciu
schematów; `spice` do weryfikacji filtrów/dzielników (np. dzielnik ADC z sekcji 2); `emc` do
wstępnej analizy ryzyka EMC przed wysłaniem do fabrykacji (szczególnie wejście 8-40V i CAN).

**Faza 3 — layout PCB.** Rozdzielić masy/zasilanie wg stref (cyfrowa V3S/DDR, analogowa
AXP209/ADC, CAN bus z zapasem pod przyszłą izolację — decyzja 6). DRC + review.

**Faza 4 — prototyp i bring-up, w kolejności najniższego ryzyka do najwyższego:**
1. Zasilanie (AXP209 sekwencjonowanie rail-i, pomiar napięć) — bez V3S podłączonego.
2. UART debug (sekcja 10) — pierwszy sygnał życia z bootloadera.
3. Boot z microSD systemowej (SDC0) — potwierdzić kolejność BROM z sekcji 6.
4. Ethernet (sekcja 8) — najniższe ryzyko softwarowe, szybka weryfikacja (już potwierdzone na
   dev-boardzie, tu tylko powtórka na nowym hardware).
5. LCD + fbdev (sekcja 4.1) — już zweryfikowana ścieżka softwarowa z dev-boardu.
6. Dotyk TSC2007 + kalibracja (sekcja 4.2) — istniejący kod `touch_calib.h` powinien działać
   bez zmian po stronie firmware.
7. CAN loopback + `mpswp_simulator`/`mpcc_simulator` (już gotowe w firmware) (sekcja 5).
8. GPS fix + `gpsd` (sekcja 7).
9. LRADC klawisze funkcyjne (sekcja 4.3).
10. USB FEL/gadget (sekcja 9) — potwierdzić jako alternatywna ścieżka odzyskiwania.
11. STM32 firmware bring-up (sekcja 11) — I2C slave + IRQ (PB6) najpierw z prostym rejestrem
    testowym, potem ADC drabinki rezystorowej headsetu (kalibracja progów na realnym
    akcesorium) i GPIO przycisków dodatkowych; audio ALSA (kodek V3S) osobno, równolegle.

**Faza 5 — integracja z `firmware/CanSensorHub-v3s`.** `csh_power_info_present()` zacznie
zwracać `true` (real AXP209 zamiast placeholdera — zob. `power_info.h`), potwierdzić realne
wartości z `/sys/class/power_supply/axp20x-*`; `CSH_THERMAL_ZONE_PATH` do zweryfikowania na
nowym sprzęcie (może być inny numer strefy niż na dev-boardzie); potwierdzić na sprzęcie, że
`rtc-sun6i` binduje się poprawnie do węzła DT wbudowanego RTC (decyzja 14, pkt. 1.2).

**Faza 6 — testy środowiskowe / pre-compliance.** EMC (input 8-40V + CAN, transientom
poświęcić szczególną uwagę), test podtrzymania RTC wbudowanego V3S na baterii backup LIR2032
(rzeczywisty czas vs szacunek ~25-30µA z pkt. 1.2 — jeśli w praktyce za krótki, MCP79410
zostaje udokumentowaną ścieżką odwrotu), test pracy na baterii LiPo (realny czas pracy vs
oszacowanie z pkt. 1.1).

**Faza 7 — aktualizacje w polu.** Wdrożenie schematu A/B (RAUC) — patrz "Dodatek:
aktualizacje systemu" — konfiguracja partycji, integracja skryptów U-Boota, test rollbacku po
symulowanej nieudanej aktualizacji (w tym zanik zasilania w trakcie zapisu).
