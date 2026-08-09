# V3S — pełna rozpiska 128 pinów (LQFP128) i gałęzie zasilania

Dokument uzupełniający do [`TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md`](TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md) —
tam są decyzje koncepcyjne per-blok (AXP209, CAN, LCD, itd.), tu jest **pełna, płaska rozpiska
wszystkich 128 ponumerowanych pinów** chipu Allwinner V3S w obudowie LQFP128, pin po pinie, z
gałęzią zasilania i przydziałem funkcji w tym konkretnym projekcie.

Zob. też [`AXP209_PINOUT_ROZPISKA.md`](AXP209_PINOUT_ROZPISKA.md) — analogiczna rozpiska PMIC
AXP209 (drzewo DC/DC1-3 + LDO1-4, realne wartości L/C trzech zewnętrznych przetwornic z
`pcb/dcdc.kicad_sch`, połączenia AXP209↔V3S). Sekcja 2 poniżej (gałęzie zasilania) jest od
2026-07-24 zaktualizowana o **potwierdzone** tam mapowanie AXP209→V3S (wcześniej "do
potwierdzenia"). Od 2026-07-25 zob. też
[`STM32_AUDIO_BUTTONS_ROZPISKA.md`](STM32_AUDIO_BUTTONS_ROZPISKA.md) — kontroler audio/
przycisków headsetu dodany na TWI1 + PB6 (sekcja 11 dok. nadrzędnego), zmienił przydział pinów
45/46 (Port B) oraz status pinów audio (113-125) niżej.

Stan na 2026-07-25 (piny 45/46/113-125 zaktualizowane o dodanie audio+STM32; reszta z
2026-07-24).

## Źródła i metoda

- **Numery pinów, pełne nazwy (z wszystkimi funkcjami alternatywnymi) i typ elektryczny** —
  wyciągnięte programowo z `pcb/local_lib/V3S_AXP209.kicad_sym` (symbol `V3S_1_0`). To ta sama
  transkrypcja pinów krzemu, którą `TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md` już traktuje jako
  źródło referencyjne. Zweryfikowano: dokładnie **128 ponumerowanych pinów (1-128)**, bez dziur,
  plus 9 dodatkowych punktów `EGND` (pad termiczny/masowy pod spodem obudowy — zob. niżej).
  Etykieta na samym symbolu: **`V3S-eLQFP128`**.
- **Napięcia gałęzi zasilania, mapowanie port→domena I/O, liczba kanałów LRADC, pakiet, kwarce** —
  research w oparciu o oficjalny **Allwinner V3s Datasheet V1.0** (rozdz. 3 Pin Characteristics /
  GPIO Multiplex Functions, rozdz. 9 Electrical Characteristics — Tabele 9-1 do 9-5), skrzyżowany
  z trzema niezależnymi, publicznie dostępnymi projektami referencyjnymi wykorzystującymi ten sam
  chip: oficjalny **Allwinner V3S_STD_CDR** (ten sam plik co
  `doc/V3S_CDR_STD_V1_0_20150514.pdf` w tym repo), **LicheePi Zero** (obecna płyta deweloperska
  tego projektu) i **FunKey S** (już przywoływany w `TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md` jako
  punkt odniesienia dla SY8088). Numery pinów z tego researchu **zgadzają się co do jednego** z
  numeracją wyciągniętą lokalnie z `V3S_AXP209.kicad_sym` (sprawdzone krzyżowo dla VCC-IO0-3,
  VCC-PE0/1, RESET, LRADC0) — traktuję to jako silne potwierdzenie spójności obu źródeł.
- **Przydział funkcji w tym projekcie** — z decyzji już podjętych w
  `TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md` (sekcje 1-10 + tabela alokacji zasobów + sekcja
  EINT/I2C).
- Jak w dokumencie nadrzędnym: pozycje niepewne oznaczone **(do potwierdzenia)** — nie zgaduję
  liczb, których nie mogłem zweryfikować z pierwotnego źródła.

---

## 1. Obudowa

**V3S-eLQFP128** — LQFP 128-pin, 14×14 mm, rozstaw 0.4 mm, z **eksponowanym paddem
termicznym/masowym** pod spodem (stąd przedrostek "e" w eLQFP). W datasheecie pad ten nazywa się
**EPAD**; w lokalnym symbolu KiCad rozbity jest na 9 osobnych punktów `GND$1`…`GND$9`, wszystkie
nazwane `EGND`, prawdopodobnie żeby ułatwić rysowanie siatki przelotek (via stitching) pod
paddem — dokładnie ta sama konwencja co w symbolu AXP209 w tym samym pliku (`EP`, też 9 punktów,
tam faktyczny exposed pad QFN-48). **Pad nie ma osobnego numeru w numeracji 1-128** — to dodatkowy,
129. punkt lutowniczy, elektrycznie = GND. Brak w datasheecie/design guide sztywno zalecanej
liczby przelotek pod paddem — standardowa praktyka dla exposed-pad QFP (siatka drobnych via,
np. 0.2-0.3 mm, połączona z wewnętrzną płaszczyzną masy).

---

## 2. Gałęzie zasilania — tabela zbiorcza

| Gałąź | Napięcie (typ.) | Zakres wg datasheet | Co zasila | AXP209 / źródło w tym projekcie | Pewność |
|---|---|---|---|---|---|
| **VDD-CPU** (VDD-CPU0-7, 8 pinów) | 1.2 V (realne płyty: 1.2-1.25 V) | Typ 1.2V, min/max niepodane wprost w datasheet (Min/Max = "TBD") | rdzeń CPU (Cortex-A7) | **AXP209 DCDC2, 1.25V/1.6A — potwierdzone**, zob. [`AXP209_PINOUT_ROZPISKA.md`](AXP209_PINOUT_ROZPISKA.md) §2 | Potwierdzone (3 niezależne referencje: Allwinner CDR, LicheePi Zero, FunKey S) |
| **VDD-SYS** (VDD-SYS0-5, 6 pinów) | 1.2 V | jw. | logika systemowa/peryferia | **ta sama fizyczna szyna co VDD-CPU** — **AXP209 DCDC2** wspólnie z VDD-CPU — **potwierdzone** | Potwierdzone |
| **VCC-IO** (VCC-IO0-3, 4 piny) | 3.3 V | 1.7 V min, 1.8-3.3 V typ (zakres aplikacyjny), 3.6 V max | Port B, C, F, G (**jedna wspólna domena**, nie 4 niezależne) | **AXP209 DCDC3, 3.3V/1.2A — potwierdzone**; wymagane 3.3V dla bootowania z SD/eMMC na etapie BROM | Potwierdzone |
| **VCC-DRAM** (VCC-DRAM0-11, 12 pinów) | **1.8 V** | 1.7/1.8/1.9 V, abs. max 1.98V | zintegrowana pamięć DDR2 512Mbit (SiP wewnątrz V3S) | **już zdecydowane**: SY8088AAC, sekcja 3 dok. nadrzędnego | Potwierdzone — DDR2 1.8V to jedyny wspierany typ pamięci (zintegrowana w pakiecie, nie do wyboru) |
| **VCC-PLL** | 3.0 V | 2.7/3.0/3.3 V | analogowe zasilanie PLL | dzielone z AVCC (jedna LDO) — **AXP209 LDO2, 3.0V/200mA — potwierdzone** | Potwierdzone |
| **VCC-PE0/VCC-PE1** (2 piny) | 3.3 V (aplikacyjnie) | 1.7-3.6V abs., 1.8-3.3V typ — **identyczny zakres jak VCC-IO, ale osobna domena krzemowa** | Port E (LCD RGB / CSI kamera, multipleksowane) | w tym projekcie **bez kamery CSI** → można spiąć z VCC-IO na wspólne 3.3V (tak jak LicheePi Zero) — **(do potwierdzenia przy layoucie)** | VCC-PE jako osobna, niezależnie zasilalna domena: potwierdzone; konkretna wartość 3.3V dla tego projektu: rekomendacja, nie z datasheetu |
| **VCC-MCSI** | 3.3 V | 3.0/3.3/3.6 V | zasilanie różnicowych pinów MIPI-CSI (kamera) | **nieużywane w tym projekcie** (brak kamery, sekcja 4 dok. nadrzędnego) — pin można zostawić NC lub podłączyć do 3.3V wg dobrej praktyki referencyjnej | Potwierdzone (wartość); nieużywane (zastosowanie) |
| **VCC-USB** | 3.3 V | 3.0/3.3/3.45 V | zasilanie PHY USB OTG | wymagane nawet przy USB "device-only" (sekcja 9) — PHY potrzebuje zasilania niezależnie od tego, czy port dostarcza VBUS na zewnątrz | Potwierdzone |
| **VCC-RTC** | 3.3 V | 3.0/3.3/3.6 V | zewnętrzny pin zasilania domeny RTC | **AXP209 LDO1, 3.3V Always-On — potwierdzone** (backup z LIR2032, sekcja 1.2 dok. nadrzędnego; LDO1SET strap → VINT daje LDO1=3.3V) | Potwierdzone |
| **AVCC / AGND** | 3.0 V / 0V (masa analog.) | 2.8/3.0/3.3 V | zasilanie analogowe ADC/kodeka audio | kodek audio **używany od 2026-07-25** (sekcja 11 dok. nadrzędnego — audio na innym urządzeniu z tej samej płyty), AVCC zasilane wspólnie z VCC-PLL z **AXP209 LDO2 — potwierdzone** | Potwierdzone (wartość) |
| **HPVCCIN / HPVCCBP** | ~3.3 V | brak wprost w Tabeli 9-2 | zasilanie wzmacniacza słuchawkowego / jego bypass | **używane** — słuchawki przez gniazdo TRRS 3.5mm (sekcja 11 dok. nadrzędnego) | Niepewne/"likely" — tylko jedna płyta referencyjna (LicheePi Zero) daje konkretną wartość |
| **VRA1 / VRA2** | **nieznane** | brak w dostępnych źródłach | wewnętrzne referencje LDO kodeka audio (piny wyjściowe, tylko bypass-cap na zewnątrz) | **używane** — zostawić z samym kondensatorem bypass wg referencyjnego layoutu, nie doprowadzać zasilania z zewnątrz | **Nieznane** — świadomie nie zgaduję liczby, żadne sprawdzone źródło jej nie podaje |
| **SVREF0 / SVREF1** | **niepotwierdzone** (przypuszczalnie ~VCC-DRAM/2 ≈ 0.9V, konwencja DDR2) | brak w Tabeli 9-2 | referencja napięcia dla kontrolera/PHY DDR | tylko kondensatory dekupling w projektach referencyjnych, brak widocznego dzielnika z VCC-DRAM — sugeruje, że może być generowane wewnętrznie | **Niepewne** — nie traktować ~0.9V jako potwierdzonej wartości |
| **SZQ** | n/d (to nie szyna zasilania) | — | pin kalibracji ZQ dla DDR — rezystor precyzyjny do masy | **240 Ω 1%** do GND (potwierdzone w oficjalnym reference design Allwinner) | Potwierdzone |
| **EPAD / EGND** (pad termiczny, 9 punktów w symbolu) | 0 V (masa) | — | pad termiczny/masowy pod spodem obudowy | lutować do płaszczyzny GND, siatka przelotek wg dobrej praktyki QFP e-pad | Potwierdzone (obecność padu); liczba przelotek — do ustalenia przy layoucie |

**Uwaga o VDD-CPU/VDD-SYS**: we wszystkich trzech sprawdzonych projektach referencyjnych
(Allwinner CDR, LicheePi Zero, FunKey S) te dwie domeny są **fizycznie tą samą szyną** (jedno
wyjście DCDC ~1.2-1.25V). To ważne dla sekwencjonowania zasilania w Fazie 4 (bring-up) — nie ma
tu dwóch niezależnych rail-i do sekwencjonowania osobno.

**Mapowanie AXP209→V3S — potwierdzone (2026-07-24)**: DCDC2 (1.25V/1.6A) → VDD-CPU/VDD-SYS,
DCDC3 (3.3V/1.2A) → VCC-IO (+VCC-USB/VCC-MCSI/VCC-PE), LDO2 (3.0V/200mA) → AVCC/VCC-PLL, LDO1
(3.3V Always-On) → VCC-RTC. Pełne uzasadnienie, realne wartości dławików/kondensatorów i
połączenia AXP209↔V3S (I2C, IRQ, PWRON, ADC pomiaru napięcia wejściowego) — zob.
[`AXP209_PINOUT_ROZPISKA.md`](AXP209_PINOUT_ROZPISKA.md). DCDC2/DCDC3 max prądy (1.6A/1.2A)
potwierdzone jako wystarczające dla rdzenia V3S przy 1GHz i domeny IO.

---

## 3. Mapowanie portów GPIO → domena zasilania I/O

| Port | Piny | Zasilany z | Uwaga |
|---|---|---|---|
| **Port B** (PB0-PB9) | 39,40,41,42,43,44,45,46,48,49 | VCC-IO (wspólna domena z C/F/G) | UART2 (GPS), EINT×3, PWM0, TWI0 (rezerwa), UART0 |
| **Port C** (PC0-PC3) | 52-55 | VCC-IO (jw.) | SPI0 (alt. SDC2) → MCP2518FD (CAN) |
| **Port E** (PE0-PE24) | 6-11,13-18,22-24,27-28,30-37 | **VCC-PE0/VCC-PE1** (domena osobna od VCC-IO!) | LCD RGB (panel 5"), PE21/PE22 wyjątkowo na TWI1 (patrz niżej) |
| **Port F** (PF0-PF6) | 100-103,105-107 (+100 bez alt-funkcji) | VCC-IO (jw.) | SDC0 (eMMC), JTAG nieużywany, alt. lokalizacja UART0 nieużywana |
| **Port G** (PG0-PG5) | 1-5,128 | VCC-IO (jw.) | SDC1 (microSD) |

Port E ma **własną, elektrycznie osobną domenę zasilania** (VCC-PE0/1, piny 12 i 29) — nie jest to
ta sama szyna co VCC-IO, mimo identycznego dozwolonego zakresu napięć (1.8-3.3V). To zamierzone
przez Allwinner: pozwala zasilić Port E inaczej niż Porty B/C/F/G (np. 1.8V dla kamery CSI przy
3.3V na reszcie do bootowania z SD). **W tym projekcie, bez kamery CSI**, nie ma powodu
różnicować napięć — VCC-PE0/1 można spiąć z tą samą 3.3V co VCC-IO (tak robi LicheePi Zero).

**Uwaga o Ethernet (EPHY)**: badanie zewnętrzne miało wątpliwość, czy V3S (z małym "s") w ogóle ma
zintegrowany PHY Ethernet, bo to zwykle cecha odróżniająca od zwykłego "V3" — **ale to nie dotyczy
tego projektu**: `TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md` (sekcja 8) opisuje to jako **już
zweryfikowane na rzeczywistym sprzęcie** (DHCP + ping, `firmware/CanSensorHub-v3s/README.md`), a
lokalny symbol `V3S_AXP209.kicad_sym` ma 9 dedykowanych pinów `EPHY-*` (patrz tabela pełna niżej).
Rozstrzygnięte jednoznacznie na korzyść "V3S w tym projekcie ma zintegrowany EPHY" — nie trzeba tego
dalej weryfikować.

---

## 4. Pełna tabela — wszystkie 128 pinów

Legenda typu elektrycznego: **we/wy** = bidirectional, **zas.we** = power_in, **zas.wy** =
power_out, **pas.** = passive/analogowy, **we** = input, **wy** = output, **we↓** = input
inverted (aktywne nisko).

| Pin | Nazwa (funkcje alternatywne) | Typ | Grupa | Gałąź zasilania / domena | Przydział w tym projekcie |
|---|---|---|---|---|---|
| 1 | PG4/SDC1_D2/PG_EINT4 | we/wy | Port G | VCC-IO | SD/microSD — SDC1_D2 (sekcja 6) |
| 2 | PG3/SDC1_D1/PG_EINT3 | we/wy | Port G | VCC-IO | SD/microSD — SDC1_D1 |
| 3 | PG2/SDC1_D0/PG_EINT2 | we/wy | Port G | VCC-IO | SD/microSD — SDC1_D0 |
| 4 | PG1/SDC1_CMD/PG_EINT1 | we/wy | Port G | VCC-IO | SD/microSD — SDC1_CMD |
| 5 | PG0/SDC1_CLK/PG_EINT0 | we/wy | Port G | VCC-IO | SD/microSD — SDC1_CLK |
| 6 | PE24/LCD_D23 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D23 (mapowanie na R/G/B do potwierdzenia z datasheetem panelu BL050S061-17) |
| 7 | PE23/LCD_D22 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D22 |
| 8 | PE22/CSI_SDA/TWI1_SDA/UART1_RX | we/wy | Port E / I2C | VCC-PE | **TWI1_SDA** → magistrala I2C AXP209+TSC2007+STM32 (decyzja EINT/I2C w dok. nadrzędnym, STM32 dodane sekcja 11); brak alt-funkcji LCD na tym pinie, więc nie koliduje z 24-bit LCD |
| 9 | PE21/CSI_SCK/TWI1_SCK/UART1_TX | we/wy | Port E / I2C | VCC-PE | **TWI1_SCK** → jw. |
| 10 | PE20/CSI_FIELD/CSI_MIPI_MCLK | we/wy | Port E / CSI | VCC-PE | nieużywane (brak kamery CSI) |
| 11 | PE19/CSI_D15/LCD_D21 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D21 |
| 12 | VCC-PE0 | zas.we | Zasilanie Port E | **VCC-PE** (3.3V zalecane, patrz sekcja 3) | zasilanie Portu E — spiąć z VCC-IO (brak kamery) |
| 13 | PE18/CSI_D14/LCD_D20 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D20 |
| 14 | PE17/CSI_D13/LCD_D19 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D19 |
| 15 | PE16/CSI_D12/LCD_D18 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D18 |
| 16 | PE15/CSI_D11/LCD_D15 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D15 |
| 17 | PE14/CSI_D10/LCD_D14 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D14 |
| 18 | PE13/CSI_D9/LCD_D13 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D13 |
| 19 | VDD-SYS3 | zas.we | Zasilanie rdzenia | **VDD-SYS** (~1.2V) | logika systemowa |
| 20 | VDD-CPU3 | zas.we | Zasilanie rdzenia | **VDD-CPU** (~1.2V) | rdzeń CPU |
| 21 | VDD-CPU2 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 22 | PE12/CSI_D8/LCD_D12 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D12 |
| 23 | PE11/CSI_D7/LCD_D11 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D11 |
| 24 | PE10/CSI_D6/LCD_D10 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D10 |
| 25 | VDD-CPU1 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 26 | VDD-CPU0 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 27 | PE9/CSI_D5/LCD_D7 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D7 |
| 28 | PE8/CSI_D4/LCD_D6 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D6 |
| 29 | VCC-PE1 | zas.we | Zasilanie Port E | **VCC-PE** | zasilanie Portu E — jw. (pin 12) |
| 30 | PE7/CSI_D3/LCD_D5 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D5 |
| 31 | PE6/CSI_D2/LCD_D4 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D4 |
| 32 | PE5/CSI_D1/LCD_D3 | we/wy | Port E / LCD | VCC-PE | LCD RGB — bit D3 |
| 33 | PE4/CSI_D0/LCD_D2 | pas. | Port E / LCD | VCC-PE | LCD RGB — bit D2 (najmłodszy wyprowadzony bit; D0/D1 nie istnieją w tym pakiecie) |
| 34 | PE3/CSI_VSYNC/LCD_VSYNC | we/wy | Port E / LCD | VCC-PE | LCD — VSYNC |
| 35 | PE2/CSI_HSYNC/LCD_HSYNC | we/wy | Port E / LCD | VCC-PE | LCD — HSYNC |
| 36 | PE1/CSI_MCLK/LCD_DE | we/wy | Port E / LCD | VCC-PE | LCD — DE (data enable) |
| 37 | PE0/CSI_PCLK-/LCD_CLK | we/wy | Port E / LCD | VCC-PE | LCD — CLK (pixel clock) |
| 38 | VDD-CPU4 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 39 | PB0/UART2_TX/PB_EINT0 | we/wy | Port B / UART2 | VCC-IO | GPS — **UART2_TX** (sekcja 7) |
| 40 | PB1/UART2_RX/PB_EINT1 | we/wy | Port B / UART2 | VCC-IO | GPS — **UART2_RX** |
| 41 | PB2/UART2_RTS/PB_EINT2 | we/wy | Port B / EINT | VCC-IO | **EINT: MCP2518FD INT** (CAN, RTS niepotrzebny dla GPS) |
| 42 | PB3/UART2_CTS/PB_EINT3 | we/wy | Port B / EINT | VCC-IO | **EINT: TSC2007 nPENIRQ** (dotyk, CTS niepotrzebny) |
| 43 | PB4/PWM0/PB_EINT4 | we/wy | Port B / PWM | VCC-IO | **PWM0 → PT4101 EN** (podświetlenie LCD) |
| 44 | PB5/PWM1/PB_EINT5 | we/wy | Port B / EINT | VCC-IO | **EINT: AXP209 IRQ/WAKEUP** (PWM1 nieużywany) |
| 45 | PB6/TWI0_SCK/PB_EINT6 | we/wy | Port B / EINT | VCC-IO | **EINT: STM32 IRQ** (kontroler audio/przycisków headsetu, sekcja 11 dok. nadrzędnego — dodane 2026-07-25); TWI0 nadal nieużywany jako magistrala, pin tylko jako EINT |
| 46 | PB7/TWI0_SDA/PB_EINT7 | we/wy | Port B / rezerwa | VCC-IO | wolny — **ostatnia rezerwa EINT** |
| 47 | VDD-CPU5 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 48 | PB8/TWI1_SCK/UART0_TX/PB_EINT8 | we/wy | Port B / UART0 | VCC-IO | **UART0_TX** (debug console, sekcja 10) — mimo nazwy "TWI1_SCK" tutaj, TWI1 fizycznie na PE21/22, nie tu |
| 49 | PB9/TWI1_SDA/UART0_RX/PB_EINT9 | we/wy | Port B / UART0 | VCC-IO | **UART0_RX** — jw. |
| 50 | VCC-IO3 | zas.we | Zasilanie I/O | **VCC-IO** (3.3V) | zasilanie Port B/C/F/G |
| 51 | VDD-CPU6 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 52 | PC0/SDC2_CLK/SPI_MISO | we/wy | Port C / SPI | VCC-IO | **SPI_MISO → MCP2518FD** (CAN, sekcja 5) |
| 53 | PC1/SDC2_CMD/SPI_CLK | we/wy | Port C / SPI | VCC-IO | **SPI_CLK → MCP2518FD** |
| 54 | PC2/SDC2_RST/SPI_CS | we/wy | Port C / SPI | VCC-IO | **SPI_CS → MCP2518FD** |
| 55 | PC3/SDC2_D0/SPI_MOSI | we/wy | Port C / SPI | VCC-IO | **SPI_MOSI → MCP2518FD** |
| 56 | VDD-CPU7 | zas.we | Zasilanie rdzenia | **VDD-CPU** | rdzeń CPU |
| 57 | VCC-IO2 | zas.we | Zasilanie I/O | **VCC-IO** | zasilanie Port B/C/F/G |
| 58 | VDD-SYS4 | zas.we | Zasilanie rdzenia | **VDD-SYS** | logika systemowa |
| 59 | VCC-DRAM8 | zas.we | Zasilanie DDR2 | **VCC-DRAM** (1.8V) | pamięć DDR2 zintegrowana (SiP) |
| 60 | VCC-DRAM9 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 61 | VCC-DRAM10 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 62 | VCC-DRAM11 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 63 | SVREF1 | pas. | Referencja DDR | **SVREF** (niepotwierdzone, ~VCC-DRAM/2?) | referencja napięcia DDR |
| 64 | VDD-SYS5 | zas.we | Zasilanie rdzenia | **VDD-SYS** | logika systemowa |
| 65 | VCC-DRAM0 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | pamięć DDR2 |
| 66 | VCC-DRAM1 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 67 | VCC-DRAM2 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 68 | VCC-DRAM3 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 69 | VCC-DRAM4 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 70 | VCC-DRAM5 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | jw. |
| 71 | SVREF0 | pas. | Referencja DDR | **SVREF** | referencja napięcia DDR |
| 72 | VCC-DRAM6 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | pamięć DDR2 |
| 73 | SZQ | pas. | Kalibracja DDR | n/d (rezystor 240Ω 1% do GND) | kalibracja ZQ DDR — rezystor precyzyjny |
| 74 | X24MOUT | pas. | Oscylator główny | — | kwarc 24.000 MHz (wyjście) |
| 75 | X24MIN | pas. | Oscylator główny | — | kwarc 24.000 MHz (wejście); ±40/50ppm; trzymać ≥25mm od krawędzi płytki wg design guide |
| 76 | VCC-PLL | zas.we | Zasilanie PLL | **VCC-PLL** (3.0V) | zasilanie analogowe PLL |
| 77 | EPHY-LINK-LED | pas. | Ethernet | VCC-EPHY | LED linku Ethernet (sekcja 8) |
| 78 | EPHY-SPD-LED | pas. | Ethernet | VCC-EPHY | LED prędkości Ethernet |
| 79 | VCC-DRAM7 | zas.we | Zasilanie DDR2 | **VCC-DRAM** | pamięć DDR2 |
| 80 | VDD-SYS0 | zas.we | Zasilanie rdzenia | **VDD-SYS** | logika systemowa |
| 81 | MCSI-D0P | we | MIPI CSI | VCC-MCSI | nieużywane (brak kamery) |
| 82 | MCSI-D0N | we | MIPI CSI | VCC-MCSI | nieużywane |
| 83 | MCSI-D1P | we | MIPI CSI | VCC-MCSI | nieużywane |
| 84 | MCSI-D1N | we | MIPI CSI | VCC-MCSI | nieużywane |
| 85 | VCC-MCSI | we *(sic, wg symbolu)* | Zasilanie MIPI | **VCC-MCSI** (3.3V) | nieużywane funkcjonalnie — można NC lub podpiąć do 3.3V wg dobrej praktyki |
| 86 | MCSI-CKP | wy | MIPI CSI | VCC-MCSI | nieużywane |
| 87 | MCSI-CKN | wy | MIPI CSI | VCC-MCSI | nieużywane |
| 88 | EPHY-VDD | pas. | Ethernet | VCC-EPHY (cyfrowe) | zasilanie rdzenia PHY Ethernet |
| 89 | EPHY-RXN | pas. | Ethernet | — | para różnicowa RX− |
| 90 | EPHY-RXP | pas. | Ethernet | — | para różnicowa RX+ |
| 91 | EPHY-TXN | pas. | Ethernet | — | para różnicowa TX− |
| 92 | EPHY-TXP | pas. | Ethernet | — | para różnicowa TX+ |
| 93 | EPHY-VCC | pas. | Ethernet | VCC-EPHY (analogowe) | zasilanie analogowe PHY Ethernet |
| 94 | EPHY-RTX | pas. | Ethernet | — | rezystor bias precyzyjny (kalibracja linii) |
| 95 | X32KOUT | pas. | Oscylator RTC | — | kwarc 32.768 kHz (wyjście) — domena RTC/wake-up |
| 96 | X32KIN | pas. | Oscylator RTC | — | kwarc 32.768 kHz (wejście) |
| 97 | RTC-VIO | pas. | RTC | wewn. bypass RTC_VIO (reg. 0.7-1.4V, domyślnie 1.1V) | pin bypass/filtr wewnętrznej domeny RTC, nie zewnętrzne zasilanie |
| 98 | VCC-RTC | zas.we | Zasilanie RTC | **VCC-RTC** (3.3V) | zasilanie zewn. domeny RTC — **AXP209 LDO1, potwierdzone** (backup, sekcja 1.2) |
| 99 | RESET | we↓ | Reset | VCC-IO (poziom logiczny) | wejście resetu, **aktywne nisko** (potwierdzone typem pinu w symbolu); brak wewn. pull — wymaga zewn. kondensatora 10nF do GND blisko chipu + obwodu resetu/przycisku. **Rozważane (nie zdecydowane) źródło: AXP209 PWROK** — zob. [`AXP209_PINOUT_ROZPISKA.md`](AXP209_PINOUT_ROZPISKA.md) §7 dla analizy za/przeciw (zastrzeżenie: PWROK nie monitoruje zewnętrznej przetwornicy DRAM SY8088) |
| 100 | PF6 | we/wy | Port F | VCC-IO | brak alt-funkcji w tym symbolu (GPIO ogólnego przeznaczenia?) — **(do potwierdzenia w pełnym datasheet)**; nieużywane w tym projekcie |
| 101 | PF5/SDC0_D2/JTAG_CK1 | we/wy | Port F / eMMC | VCC-IO | eMMC — **SDC0_D2** (JTAG nieużywany, sekcja 6) |
| 102 | PF4/SDC0_D3/UART0_RX | we/wy | Port F / eMMC | VCC-IO | eMMC — **SDC0_D3** (alt. lokalizacja UART0 nieużywana — UART0 jest na PB8/9) |
| 103 | PF3/SDC0_CMD/JTAG_DO1 | we/wy | Port F / eMMC | VCC-IO | eMMC — **SDC0_CMD** |
| 104 | VCC-IO0 | zas.we | Zasilanie I/O | **VCC-IO** | zasilanie Port B/C/F/G |
| 105 | PF2/SDC0_CLK/UART0_TX | we/wy | Port F / eMMC | VCC-IO | eMMC — **SDC0_CLK** (alt. UART0 nieużywana) |
| 106 | PF1/SDC0_D0/JTAG_DI1 | we/wy | Port F / eMMC | VCC-IO | eMMC — **SDC0_D0** |
| 107 | PF0/SDC0_D1/JTAG_MS1 | we/wy | Port F / eMMC | VCC-IO | eMMC — **SDC0_D1** |
| 108 | VDD-SYS1 | zas.we | Zasilanie rdzenia | **VDD-SYS** | logika systemowa |
| 109 | VCC-USB | zas.we | Zasilanie USB | **VCC-USB** (3.3V) | zasilanie PHY USB — wymagane nawet w trybie device-only (sekcja 9) |
| 110 | USB-DM | pas. | USB | — | USB OTG D− (FEL/debug gadget) |
| 111 | USB-DP | pas. | USB | — | USB OTG D+ |
| 112 | LRADC0 | pas. | LRADC | — | **klawisze funkcyjne** (sekcja 4.3) — **jedyny kanał LRADC w tym pakiecie (potwierdzone: brak LRADC1 na V3S)**, rozstrzyga otwartą kwestię z dok. nadrzędnego |
| 113 | MICIN1P | pas. | Audio | — | **mikrofon (wejście różnicowe +)** — aktywowane 2026-07-25 (sekcja 11 dok. nadrzędnego: audio na innym urządzeniu z tej samej płyty), przez złącze TRRS 3.5mm |
| 114 | MICIN1N | pas. | Audio | — | **mikrofon (wejście różnicowe −)** |
| 115 | AVCC | zas.we | Zasilanie analog. | **AVCC** (3.0V) | analogowe ADC/kodeka audio — **używane** (LDO2 AXP209, wspólnie z VCC-PLL) |
| 116 | AGND | zas.we *(sic)* | Masa analog. | AGND (0V) | masa analogowa |
| 117 | VRA1 | zas.wy | Audio | wewn. LDO audio (nieznane V) | **używane** — zostawić z kondensatorem bypass wg referencyjnego layoutu, nie doprowadzać zasilania z zewnątrz (pin wewnętrznej referencji, nie zewnętrznego zasilania) |
| 118 | VRA2 | zas.wy | Audio | wewn. LDO audio (nieznane V) | **używane** — jw. |
| 119 | HBIAS | pas. | Audio | — | bias wzmacniacza słuchawkowego — **używane** |
| 120 | HPOUTR | pas. | Audio | — | **wyjście słuchawkowe R** — do gniazda TRRS 3.5mm |
| 121 | HPOUTL | pas. | Audio | — | **wyjście słuchawkowe L** (lub mono) — do gniazda TRRS 3.5mm |
| 122 | HPVCCIN | zas.we | Zasilanie audio | **HPVCCIN** (~3.3V, niepewne) | **używane** — zasilanie wzmacniacza słuchawkowego |
| 123 | HPVCCBP | zas.wy | Audio (bypass) | — | **używane** — pin bypass wzmacniacza słuchawkowego |
| 124 | HPCOMFB | pas. | Audio | — | common-mode feedback wzmacniacza słuchawkowego — **używane** |
| 125 | HPCOM | pas. | Audio | — | common-mode wzmacniacza słuchawkowego — **używane** |
| 126 | VDD-SYS2 | zas.we | Zasilanie rdzenia | **VDD-SYS** | logika systemowa |
| 127 | VCC-IO1 | zas.we | Zasilanie I/O | **VCC-IO** | zasilanie Port B/C/F/G |
| 128 | PG5/SDC1_D3/PG_EINT5 | we/wy | Port G | VCC-IO | SD/microSD — SDC1_D3 |
| GND$1…GND$9 | EGND ×9 (= EPAD w datasheet) | zas.we | Pad termiczny | GND (masa) | pad termiczny/masowy pod obudową — lutować do płaszczyzny GND (siatka przelotek, poza numeracją 1-128) |

---

## 5. Rzeczy jednoznacznie rozstrzygnięte tym researchem (wcześniej otwarte w dok. nadrzędnym)

- **Liczba kanałów LRADC** (otwarte pytanie w sekcji 4.3 dok. nadrzędnego: "ile kanałów LRADC V3S
  faktycznie wyprowadza w tym pakiecie") → **rozstrzygnięte: tylko jeden, LRADC0 (pin 112)**. V3S
  nie ma LRADC1 (w przeciwieństwie do części innych chipów sunxi, np. A10/A20/A31). Budżet
  klawiszy funkcyjnych z sekcji 4.3 nie podwaja się — jedna drabinka rezystorowa, jeden kanał.
- **Polaryzacja pinu RESET** — potwierdzona jako aktywna nisko wprost z typu elektrycznego pinu
  w lokalnym symbolu KiCad (`input inverted`), co jest spójne z ogólną konwencją rodziny sunxi.
- **Obecność zintegrowanego PHY Ethernet na V3S** — potwierdzona (patrz sekcja 3 wyżej) mimo
  ogólnej wątpliwości, że "V3s" (małe "s") czasem bywa mylone z wariantem bez PHY — w tym
  projekcie jest to i tak już zweryfikowane na sprzęcie.

## 6. Rzeczy nadal otwarte (przenieść do Fazy 1/2 dok. nadrzędnego)

- ~~Dokładne mapowanie AXP209 DCDC2/DCDC3/LDO1-4 → gałęzie V3S~~ — **rozstrzygnięte 2026-07-24**,
  zob. sekcja 2 wyżej i [`AXP209_PINOUT_ROZPISKA.md`](AXP209_PINOUT_ROZPISKA.md).
- Napięcie VRA1/VRA2 i SVREF0/1 — nieznalezione w żadnym sprawdzonym źródle; nie blokuje
  schematu (to piny wewnętrznego bypass/referencji, nie zewnętrznego zasilania), ale warto
  zweryfikować bezpośrednio w `doc/V3S_CDR_STD_V1_0_20150514.pdf`, jeśli będzie potrzebna
  dokładna wartość kondensatorów dekuplingowych. **Teraz istotniejsze niż wcześniej** — piny
  audio (113-125, w tym VRA1/VRA2) są od 2026-07-25 rzeczywiście używane (sekcja 11 dok.
  nadrzędnego), nie tylko teoretyczne.
- **Dokładne wartości komponentów aplikacyjnych kodeka audio V3S** (sprzęganie DC na MICIN1P/N,
  dopasowanie HPVCCIN/HPVCCBP) — otwarte, zob. `STM32_AUDIO_BUTTONS_ROZPISKA.md` i sekcja 11.1
  dok. nadrzędnego.
- Funkcja pinu 100 (`PF6`, brak alt-funkcji w lokalnym symbolu) — do sprawdzenia w pełnym
  datasheet, czy ma dodatkowe przeznaczenie w tym pakiecie.
- Dokładna liczba/rozmiar przelotek pod paddem EPAD/EGND — do ustalenia przy layoucie (brak
  twardego wymogu z datasheet).
- Dokładne mapowanie LCD_D2-D23 na kanały R/G/B dla panelu BL050S061-17/T050SWV012T — do
  potwierdzenia z datasheetem konkretnego panelu (czy to RGB666 z pominiętymi D8/D9/D16/D17, czy
  pełne RGB888).
- **Decyzja o RESET z PWROK (pkt. 99)** — rozważone, nie rozstrzygnięte; zob.
  `AXP209_PINOUT_ROZPISKA.md` §7 (rekomendacja: PWROK→EINT rezerwowy zamiast wprost na RESET, ze
  względu na brak monitorowania szyny DRAM SY8088 przez PWROK).
