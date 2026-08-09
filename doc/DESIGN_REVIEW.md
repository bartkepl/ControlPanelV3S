# ControlPanel V3s — przegląd projektowy

**Projekt:** ControlPanel (KiCad 10.0.4, 11 arkuszy hierarchicznych, PCB 4-warstwowe 120 × 75 mm)
**Data:** 2026-08-01 (przebieg 3, 15:38)
**Analizatory:** `analyze_schematic.py`, `analyze_pcb.py --full`, `cross_analysis.py`,
`analyze_emc.py`, `analyze_thermal.py`, `simulate_subcircuits.py` (**ngspice**),
`diff_analysis.py` — oraz natywne **KiCad DRC/ERC** przez `kicad-cli 10.0.4`
**Wyniki surowe:** `analysis/2026-08-01_1538/`

---

## Werdykt

**Płytka jest elektrycznie i topologicznie gotowa.** Trzy niezależne mechanizmy kontroli są czyste:
KiCad ERC 0 naruszeń, KiCad DRC 0 elementów niepołączonych i 0 rozjazdów ze schematem,
ngspice 78/80 przebiegów pass bez ani jednego `fail`. Wszystkie blokery z poprzednich dwóch
przebiegów zostały naprawione i zweryfikowane w plikach.

**Do produkcji brakuje jednej rzeczy obowiązkowej: numerów MPN** (38 / 106 unikalnych części, 35,8 %).
Bez tego nie da się złożyć zamówienia ani zweryfikować pinoutów wobec kart katalogowych.

Poza tym otwarte pozostają wyłącznie pozycje jakościowe: jedna realna kolizja obrysów,
brak fiducialów, brak punktów testowych i brak oznaczeń na płytce.

---

## Przegląd

Terminal sterujący na SoC **Allwinner V3S** (U2, LQFP-128, DDR2 w obudowie) z PMIC **AXP209** (U7)
i koprocesorem **STM32L432KC** (U5). Peryferia: CAN FD (MCP2518FD + ATA6561 z przełączaną
programowo terminacją na przekaźnikach PhotoMOS), Ethernet 100BASE-TX przez wewnętrzny EPHY V3S
i magnetykę H1102NL, GPS MAX-M10S z LNA MAX2674, wzmacniacz słuchawkowy TPA6132A2, kontroler
dotyku NS2009, LCD RGB666 przez FPC 40-pin, dwa sloty microSD, zasilanie 9–60 V przez buck TPS54360.

309 komponentów (219 top / 90 bottom), 100 % SMD, 242 sieci, 5016 mm ścieżek, 3182 przelotki.

---

## Delta wobec poprzednich przeglądów

| Metryka | Przebieg 1 (00:28) | Przebieg 2 (15:03) | **Przebieg 3 (15:38)** |
|---------|-------------------|--------------------|------------------------|
| KiCad DRC — naruszenia | 16 | 26 | **12** |
| KiCad DRC — niepołączone | 0 | **2** | **0** |
| KiCad DRC — zwarcia | 0 | **8** | **0** |
| KiCad ERC | 0 | 0 | **0** |
| `cross_analysis` | 21 | 61 | **20** |
| `XV-002` (rozjazd wartości sch↔PCB) | — | 41 | **0** |
| PCB — zgłoszenia | 218 | 133 | **133** |
| SPICE | brak (LTspice = same skip) | 78/80 pass | **78/80 pass** |
| Przecinki dziesiętne w BOM | 23 | 3 | **0** |
| Pokrycie MPN | 35,2 % | 35,2 % | **35,8 %** |

### Naprawione i zweryfikowane w plikach

| Pozycja | Weryfikacja |
|---------|-------------|
| **Y2 → kwarc 24 MHz** | `CX2016FB24000D0FZZ` (Kyocera, 24,000 MHz), footprint zmieniony na `Crystal_SMD_2016-4Pin_2.0x1.6mm`. Pozycja 36 mm od krawędzi — wymóg design guide V3S ≥ 25 mm zachowany. |
| **LDO24IN / LDO3IN AXP209** | Pady 13 i 40 `U7` → `IPSOUT`. Zapas napięcia dla LDO2/3/4 i PSRR dla GPS odzyskane. |
| **8 zwarć GND ↔ `/AUDIO/DATA-/GPIO1`** | `shorting_items` = 0. Przelotki zszywające przestały kolidować ze ścieżką In1.Cu. |
| **GPS `UART2_RX`/`UART2_TX`** | Pady 2 i 3 `U6` mają przypisane sieci, `unconnected_items` = 0, `track_dangling` = 0. |
| **Synchronizacja schemat → PCB** | `XV-002`: 41 → **0**. `C131` ma teraz `2.2u` po obu stronach. |
| **Przecinki dziesiętne** | 23 → 3 → **0**. Analizator poprawnie parsuje `32.768kHz`, ngspice potwierdził CL Y3 = 6,1 pF. Wykrywalność dzielników wzrosła z 4 do 7. |
| **Rozjazdy footprintów** | 7 → **3** (`R57`, `R68`, `R69`, `D43` naprawione; zostały `T1`, `U11`, `D42`). |

### ⚠ Sprostowanie do przeglądu nr 2

W poprzednim raporcie napisałem, że nakładki obrysów zostały naprawione („14 → 0, `PM-001`
zniknęło całkowicie"). **To było błędne.** Wyciągnąłem ten wniosek z porównania, które wypisywało
wyłącznie *zmienione* reguły — brak `PM-001` na liście oznaczał, że nic się nie zmieniło, a nie
że problem zniknął. Zestaw `PM-001` jest **identyczny we wszystkich trzech przebiegach**:
1 error + 13 warning + 3 info, ta sama lista par. Pozycja wraca jako otwarta.

Analogicznie `VP-001` (przelotki w padach): zestaw jest bit w bit taki sam jak w przebiegu 2 —
26 wystąpień, 21 unikalnych, żadne nie ubyło ani nie przybyło. W całym pliku `.kicad_pcb`
występuje jeden token `tenting`. Odnotowuję bez rozstrzygania, czy poprawka miała inny charakter
niż mierzy analizator.

---

## Wyniki krytyczne

| Waga | Problem | Sekcja |
|------|---------|--------|
| **BLOKER PRODUKCYJNY** | Pokrycie MPN 38 / 106 (35,8 %) — nie da się zamówić ani zweryfikować pinoutów | [Sourcing](#sourcing) |
| **WARNING** | Kolizja obrysów `C16` ↔ `J3` (2,321 mm²) — jedyna nakładka na poziomie error | [Rozmieszczenie](#rozmieszczenie) |
| **WARNING** | Brak fiducialów na obu stronach (219 + 90 elementów SMD, BGA 0,4 mm) | [DFM](#dfm-i-uwagi-produkcyjne) |
| **WARNING** | 0 punktów testowych na 232 sieci | [Testowalność](#testowalność) |
| **SUGESTIA** | Szyny `+3V3` (DCDC3) i `+3.3V` (LDO4) różnią się tylko kropką | [Drzewo zasilania](#drzewo-zasilania) |
| **SUGESTIA** | Brak nazwy płytki i oznaczenia rewizji (`board_text_count: 0`) | [Silkscreen](#silkscreen) |

Brak wyników na poziomie CRITICAL. Wszystkie trzy blokery z przeglądu nr 1 i cztery regresje
z przeglądu nr 2 są zamknięte.

---

## Weryfikacja natywna KiCad

Najmocniejszy dowód w przeglądzie — silnik łączności KiCada, nie heurystyka analizatora.

```
kicad-cli pcb drc --severity-all  →  12 naruszeń, 0 niepołączonych, 0 rozjazdów ze schematem
kicad-cli sch erc --severity-all  →  0 naruszeń
```

Wszystkie 12 naruszeń to kwestie kosmetyczne lub biblioteczne:

| Typ | Liczba | Szczegóły |
|-----|--------|-----------|
| `silk_edge_clearance` | 7 | Opis `J13` przycięty krawędzią płytki |
| `lib_footprint_mismatch` | 3 | `T1`, `U11`, `D42` |
| `text_height` | 2 | `U2` i `U4` — 0,579 mm przy minimum 0,8 mm |

ERC ma wyłączone tylko 3 reguły (`footprint_filter`, `four_way_junction`, `simulation_model_issue`),
więc „0 naruszeń" ma realną wartość dowodową.

### Czego DRC nadal nie sprawdza

W `.kicad_pro` część ograniczeń jest ustawiona na zero — odpowiednie testy nie mają czego łamać:

| Reguła | Ustawienie | Konsekwencja |
|--------|-----------|--------------|
| `min_copper_edge_clearance` | 0,0 mm | Miedź może dotykać krawędzi (świadoma decyzja — producent cofnie wylewkę) |
| `solder_mask_to_copper_clearance` | 0,0 mm | Brak kontroli maski względem miedzi |
| `min_connection` | 0,0 mm | Brak kontroli szerokości połączenia z polem |

Dodatkowo wyłączone: `silk_overlap`, `silk_over_copper`, `missing_courtyard`,
`footprint_type_mismatch`, `track_not_centered_on_via`, `footprint_filters_mismatch`.

---

## Podsumowanie komponentów

| Typ | Liczba |
|-----|--------|
| Kondensatory | 137 |
| Rezystory | 69 |
| Diody (w tym TVS/GDT) | 40 |
| Układy scalone | 17 |
| Złącza | 14 |
| Cewki / koraliki ferrytowe | 7 / 7 |
| Kwarce | 3 |
| Pozostałe (transformator, bateria, bezpiecznik, tranzystor, filtr, TP, otwory montażowe) | 15 |

**309 komponentów, 105 unikalnych, 4 oznaczone DNP.**
Obudowy: 0603 × 222, 0402 × 12, 1206 × 9, SOT-23 × 6, QFN × 4, SOP/SOIC × 5, LQFP × 1,
**BGA × 1**, 1210/1812 × 2. Złożoność montażu **40/100**, 18 elementów trudnych, 62 unikalne footprinty.

---

## Drzewo zasilania

```
J12 (pin 1,2) ── F1 (PTC 2A) ── FB2 ── D8 ∥ D11 (PMEG6030EP, ochrona odwrotnej polaryzacji)
   │                                       │  D7/D10 (SMBJ36A TVS)
   │  GD1 (GDT 3R090-5S) → Earth           ▼
   │  D5 (SMBJ33CA)                     DC_IN  (9–60 V)
   │                                       │
   └── (pin 3,4) ── FB4 ── GND             ▼
                                  U3 TPS54360  buck, fsw ≈ 597 kHz (R34 = 162k)
                                    FB: R36 53.6k / R37 10.2k, Vref 0,8 V
                                    → Vout = 0,8 × (1 + 53,6/10,2) = 5,004 V  ✓
                                    EN: R38 523k / R39 84.5k → Von ≈ 9,3 V / Voff ≈ 7,5 V
                                       │
                                    ACIN 5,0 V
                                       │
                                 U7 AXP209 (PMIC, sterowany po I2C TWI1)
                                       │
                                    IPSOUT ──┬── U4 SY8088 buck → +1V8 (VCC-DRAM ×12)
                                             │      FB: R32 200k / R33 100k, Vref 0,6 V
                                             │      → 0,6 × 3 = 1,800 V  ✓ (ngspice: Vfb 0,600 V)
                                             │      EN: R31 100k pull-up z +3V0
                                             │
                                             ├── DCDC2 → +1V25   (VDD-CPU ×8, VDD-SYS ×6)
                                             ├── DCDC3 → +3V3    (VCC-IO ×4, VCC-PE ×2, VCC-USB,
                                             │                    VCC-MCSI, HPVCCIN, NS2009, U8)
                                             ├── LDO1  → +3V3_AO (VCC-RTC, GPS V_BCKP)
                                             ├── LDO2  → +3V0    (AVCC, VCC-PLL)          ✓ z IPSOUT
                                             ├── LDO3  → +3V3_GPS (MAX-M10S)              ✓ z IPSOUT
                                             └── LDO4  → +3.3V   (STM32, MCP2518FD,
                                                                  ATA6561 VIO, TPS61240)  ✓ z IPSOUT

    +3.3V ── IC2 TPS61240 boost → +5V (ATA6561 VCC, sterowanie PhotoMOS)
    +3V3  ── U8 TPS61041 boost  → VLED+ (podświetlenie LCD), FB przez R42 62 Ω
                                   ILED = 1,233 V / 62 Ω = 19,9 mA, D3 BZV55C24 jako klamra
```

**Wejścia LDO2/LDO3/LDO4 są teraz zasilane z `IPSOUT`** — zgodnie z `doc/AXP209_PINOUT_ROZPISKA.md`
(w. 145 i 169). W przeglądzie nr 1 były podpięte do wyjścia DCDC3, co dawało LDO3 i LDO4 zerowy
zapas napięcia (3,3 V → 3,3 V) i kasowało PSRR na szynie GPS. Naprawione.

### Uwaga do nazewnictwa

`+3V3` (DCDC3, 39 pinów zasilających) i `+3.3V` (LDO4, 14 pinów) to dwie różne szyny o nazwach
różniących się jedną kropką. Dokumentacja projektu sygnalizuje to sama
(`AXP209_PINOUT_ROZPISKA.md` w. 374). Przy każdej przyszłej zmianie to realne ryzyko podłączenia
czegoś do niewłaściwej szyny 3,3 V. Propozycja: `+3V3_IO` / `+3V3_CAN`.

---

## Weryfikacja symulacyjna (ngspice)

`D:\Spice64\bin\ngspice.EXE`, **80 symulacji w 7,7 s: 78 pass, 2 warn, 0 fail, 0 skip.**

| Kategoria | Liczba | Wynik |
|-----------|--------|-------|
| Filtry RC | 22 | wszystkie w granicach błędu **0,23–0,35 %** |
| Dzielniki napięcia | 4 | `R27/R28` → 0,900 V · `R32/R33` → 0,600 V · `R6/R7` → 0,0971 V (błąd 0,001 %) |
| Sprzężenie zwrotne regulatora | 1 | `R32/R33` (SY8088) → Vfb 0,600 V, błąd **0,0 %** → **Vout 1,800 V potwierdzone** |
| Kwarc | 1 | `Y3` 32 768 Hz, CL = 6,1 pF |
| Odsprzęganie | 11 szyn | impedancje niżej |
| Prąd rozruchowy | 5 szyn | wszystkie ustalają się na wartości docelowej, szczyty 28–107 mA |
| Tranzystor | 1 | `Q2` (BSS138): Vth 1,542 V, Ion 3,3 mA |
| Urządzenia ochronne | 33 | pass |
| Filtry LC | 2 | **2 × warn — fałszywe, patrz niżej** |

Impedancje odsprzęgania (minimum modułu):

| Szyna | Z_min | Szyna | Z_min |
|-------|-------|-------|-------|
| IPSOUT | 4,5 mΩ | +1V8 | 12,6 mΩ |
| ACIN | 5,9 mΩ | +3V0 / +3V3_GPS | 18,5 mΩ |
| +1V25 | 7,7 mΩ | +3.3V | 19,0 mΩ |
| +5V | 8,0 mΩ | +BATT | 31,6 mΩ |
| +3V3 | 9,1 mΩ | **+3V3_AO** | **67,2 mΩ** |

`+3V3_AO` jest najsłabsza, ale to szyna always-on zasilająca wyłącznie VCC-RTC V3S i V_BCKP
modułu GPS — pobór rzędu mikroamperów, impedancja bez znaczenia.

**Oba ostrzeżenia są fałszywe.** `L10/C12` i `L10/C21` wykryte jako filtr LC o Q = 115
i szczycie +40 dB przy 23,2 / 73,4 kHz. `L10` (4,7 µH) to dławik przetwornicy ładowarki AXP209
na węźle LX1, nie filtr — w rzeczywistości tłumią go pętla regulacji AXP209 i impedancja
akumulatora, których model ideal-passive nie uwzględnia.

---

## Analiza sygnałowa

### Ethernet 100BASE-TX

Standardowa topologia dla wewnętrznego EPHY V3S (jak Lichee Zero / FunKey S), prześledzona
sieć po sieci:

```
U2.89–92 (ETH_RX_N/P, ETH_TX_N/P)
   ├─ RX: → U10 (USBLC6) → RD+/RD− → C73/C83 (6.8n, sprzężenie AC) → T1.6/T1.8
   │       bias: T1.7 (RD CT) → R13/R15 (82 Ω) → RD+/RD−, C45 do masy
   └─ TX: → U9 (USBLC6) → TD+/TD− → T1.1/T1.3
           bias: +3V3 → FB1 → węzeł → R17 (10 Ω) → T1.2 (TD CT)
                 R14/R16 (49.9 Ω) → TD−/TD+
                 ten sam węzeł zasila piny VBUS U9 i U10 (odniesienie klamry) ✓

Strona liniowa: T1.9/11 (RX±), T1.14/16 (TX±) → J13
Bob Smith: R20/R26 (150 Ω) z RX± + R18/R19 (75 Ω) z T1.10/T1.15 → węzeł → C52 (1n/2kV) → Earth
```

Dopasowanie par:

| Para | Rozjazd | Ocena |
|------|---------|-------|
| `ETH_TX_P` / `ETH_TX_N` | asymetria przelotek 0 vs 2, przejść warstw 1 vs 2 | zamknięte decyzją — bez znaczenia dla 10/100M |
| `/ETH/RD+` / `RD−` | 2,1 mm (30 %) | w tolerancji 100BASE-TX |
| `/ETH/TD+` / `TD−` | 2,3 mm (27 %) | w tolerancji 100BASE-TX |

### CAN FD

```
U2 (SPI: PC0–PC3) → U1 MCP2518FD (Y1 20 MHz ✓, VDD +3.3V)
                  → IC3 ATA6561 (VCC +5V, VIO +3.3V — logika odniesiona do VIO ✓)
                  → R44/R45 (10 Ω pulse-proof) → FL1 (dławik wspólny) → GD2 → J12.5/J12.6
                     C40/C41 (4.7p) do masy, D9 CDSOT23-T24CAN przy złączu

Terminacja przełączana programowo:
   CAN_H → R22 (59 Ω) → U701 (PhotoMOS) → węzeł ── C39 (4.7n) → GND
   CAN_L → R21 (59 Ω) → U702 (PhotoMOS) → węzeł
   Razem 118 Ω — terminacja dzielona z bypassem AC
   Sterowanie: +5V → R23 (560 Ω) → LED U701 → LED U702 (szeregowo)
```

### Pozostałe interfejsy

| Interfejs | Realizacja | Stan |
|-----------|-----------|------|
| I2C (TWI1) | V3S PE21/PE22 + STM32 PB6/PB7 + NS2009 + AXP209, pull-upy `R10`/`R11` 2,2 kΩ do +3V3 | ✓ zgodne z kartą AXP209; multi-master świadomy (STM32 jako slave z przerwaniem) |
| USB | `U2.110/111` → `J19`/`J20` (pady testowe 1 × 1 mm), TVS `D24`/`D25` 2,2–2,9 mm | tylko do trybu FEL, świadome |
| LCD | RGB666 przez FPC 40-pin `J9`, masy rozłożone równomiernie | ✓ |
| microSD ×2 | `J1` (SDC1) i `J2` (SDC0) | ✓ |
| Audio | TPA6132A2 + `J14` (8-pin, masy na pinach 3 i 5, przeplecione) | ✓ |
| GPS | MAX-M10S + LNA MAX2674, UART2 do V3S | ✓ połączenie przywrócone |
| Debug | `J11` TC2030 (SWD do STM32), `J16`/`J17` (UART0 V3S) | ✓ |

### Kwarce

| Ref | Wartość | Podłączenie | Kondensatory | Ocena |
|-----|---------|-------------|--------------|-------|
| `Y2` | **CX2016FB24000D0FZZ (24 MHz)** | `U2.74` X24MOUT / `U2.75` X24MIN | `C3`, `C4` = 18 pF | ✓ naprawione |
| `Y1` | 3225-20M-SR (20 MHz) | `U1.5`/`U1.6` (MCP2518FD) | `C1`, `C2` = 18 pF | ✓ 20 MHz poprawne dla MCP2518FD |
| `Y3` | 32.768 kHz (WTL1X80739BEL) | `U2.95`/`U2.96` | `C48`, `C49` = 6,2 pF | ✓ ngspice: CL 6,1 pF |

`Y2` leży 36 mm od krawędzi przy wymogu ≥ 25 mm z design guide V3S. Wartości `C3`/`C4` (18 pF)
warto skonfrontować z CL nowego kwarcu przy okazji uzupełniania MPN.

---

## Analiza zasilania

### Odsprzęganie

| Szyna | Kondensatorów | Pinów `power_in` | Wartości |
|-------|--------------|------------------|----------|
| +1V8 (VCC-DRAM) | 8 | 17 | 4 × 100n, 2 × 1u, 2 × 10u |
| +1V25 (VDD-CPU/SYS) | 14 | 27 | 6 × 100n, 4 × 1u, 3 × 10u, 1n |
| +3V3 (VCC-IO) | 19 | 39 | 14 × 100n, 3 × 10u, 1u, 1n |
| +3.3V | 8 | 14 | 5 × 100n, 1u, 2.2u, 4.7u |
| +3V0 | 4 | 8 | 2 × 100n, 4.7u, 10u |
| +3V3_GPS | 4 | 4 | 2 × 100n, 4.7u, 10u |
| +3V3_AO | 3 | 6 | 2 × 100n, 1u |
| +5V | 3 | 3 | 100n, 4.7u, 10u |
| IPSOUT | 9 | 12 | 7 × 10u, 2 × 220n |
| ACIN | 4 | 4 | 100n, 3 × 10u |
| +BATT | 2 | 3 | 10u, 1u |

Szyna `+1V8` ma 4 × 100 nF na 12 pinów VCC-DRAM V3S — poniżej guideline'u Allwinnera
(~1 × 100 nF na pin) i zgodnie ze zgłoszeniem EMC `PD-001`. **Zamknięte decyzją projektanta**
(rozwiązanie zgodne z inną, działającą płytką) — odnotowane, nie otwieram ponownie.

### Odległości kondensatorów odsprzęgających

`C85` @ `U7` 0,21 mm · `C109` @ `U2` 0,21 mm · `C120` @ `U5` 0,24 mm · `C31` @ `U9` 0,28 mm —
bardzo dobrze. Kondensatory TPA6132A2 (`C129`/`C130`/`C131`) w promieniu 2,9–3,6 mm przy
wymaganych przez TI 5 mm ✓.

Nowe zgłoszenie **`DC-003`**: `C37` (100 nF na `+3V3_GPS`) leży daleko od przelotki masy.
Przy odsprzęganiu układu RF warto dołożyć przelotkę bezpośrednio przy padzie masy.

---

## Analiza PCB

### Stackup

| Warstwa | Typ | Grubość | Wykorzystanie |
|---------|-----|---------|---------------|
| F.Cu | signal | 0,035 mm | 1273 segmenty |
| prepreg 7628 | εr 4,4 | 0,2104 mm | |
| In1.Cu | signal | 0,0152 mm | 224 segmenty wewnątrz wylewki GND |
| core | εr 4,6 | 1,065 mm | |
| In2.Cu | power | 0,0152 mm | 1 segment — praktycznie lita płaszczyzna |
| prepreg 7628 | εr 4,4 | 0,2104 mm | |
| B.Cu | signal | 0,035 mm | 1067 segmentów |

Suma **1,586 mm** (standard 1,6 mm). Wykończenie **HAL lead-free** — świadoma decyzja
(montaż ręczny, koszt ENIG). Wylewka GND obejmuje wszystkie cztery warstwy; nie ma dedykowanej
płaszczyzny zasilania, wszystkie szyny prowadzone ścieżkami.

`In1.Cu` niesie 224 segmenty w 24 sieciach wewnątrz wylewki GND, co wycina szczeliny
w płaszczyźnie odniesienia dla sygnałów na F.Cu odległej o 0,21 mm. To źródło zgłoszeń
`GP-001` (52 error + 54 warning) i `SU-001`. **Zamknięte decyzją projektanta** — przeniesiono
na B.Cu co się dało, resztę obudowano przelotkami zszywającymi.

### Przelotki

| Rozmiar (średnica / otwór) | Liczba | Pierścień |
|---------------------------|--------|-----------|
| 0,6 / 0,3 mm | 3125 | 0,15 mm |
| 0,4 / 0,3 mm | 57 | 0,05 mm |

Obie geometrie mieszczą się w bieżących możliwościach JLCPCB, który formułuje ograniczenie
jako **różnicę** średnicy i otworu, nie jako pierścień:

> Via diameter should be **0.1 mm** (0.15 mm preferred) larger than Via hole size.

0,4 − 0,3 = 0,1 mm → minimum spełnione. Opcja zamówienia `0.3mm/(0.4/0.45mm)` jest domyślna
i bez dopłaty. Nota o wyższej cenie dotyczy tylko otworów 0,15 mm oraz 0,2/0,25 mm ze średnicą
< 0,45 mm — tu otwór to 0,3 mm.

> ⚠ **Ograniczenie 0,1 mm jest specyficzne dla JLCPCB.** PCBWay wymaga pierścienia ≥ 0,1 mm
> (różnicy 0,2 mm) — tam te 57 przelotek nie przeszłoby. Przy zmianie producenta sprawdzić ponownie.

### Rozmieszczenie

219 elementów na F.Cu, 90 na B.Cu, gęstość 2,4 / 1,0 na cm². Na spodzie leżą najważniejsze
układy: `U2` (V3S), `U7` (AXP209), `U3` (TPS54360), `J9` (FPC LCD), `U9`.

**Nakładki obrysów — 17 par, niezmienione we wszystkich trzech przebiegach:**

| Para | Powierzchnia | Waga |
|------|-------------|------|
| **`C16` ↔ `J3`** | **2,321 mm²** | **error** |
| `U701` ↔ `C32` / `C39`, `U702` ↔ `C39` | 0,71–0,77 mm² | warning |
| `J4` ↔ `J3`, `C131` ↔ `U11`, `U5` ↔ `C117`, `D30` ↔ `J3`, `R40` ↔ `J3` | 0,18–0,41 mm² | warning |
| `FB2`/`GD1` ↔ `F1`, `IC2` ↔ `C47`, `C35` ↔ `IC3`, `R4` ↔ `BT1` | < 0,02 mm² | warning |
| `U6` ↔ `C36`/`C37`/`C38` | 0,007 mm² | info — obrys `U6` zawiera strefę zakazu RF, nie kolizja mechaniczna |

Realnego sprawdzenia wymaga wyłącznie **`C16` ↔ `J3` (2,321 mm²)** — reszta to zbyt hojne
obrysy bibliotek.

**Złącza przy krawędzi** (`PM-002`, 6 sztuk): `J1` −0,9 mm, `J13` −0,34, `J14` −0,3,
`J15` −0,25, `J12` 0,2, `J3` 0,05. Dla JST PH w wersji bocznej i U.FL zamierzone,
zamknięte decyzją („w marginesie błędu").

### Termika

| Układ | Obudowa | Przelotki | Minimum | Ocena |
|-------|---------|-----------|---------|-------|
| `U3` TPS54360 | SO PowerPAD-8 | 9 | 9 | ✓ |
| `U5` STM32L432 | QFN-32 EP | 5 | 5 | ✓ |
| `U11` TPA6132A2 | WQFN-16 EP | 4 | 5 | zamknięte decyzją (25 mW/kanał) |
| `U1` MCP2518FD | DFN-14 EP | 2 | 5 | pobór znikomy |

`analyze_thermal.py`: 0 zgłoszeń, wynik 100/100 — **ale bez danych o mocy strat**
(brak MPN i kart katalogowych). To brak informacji, nie potwierdzenie.

### Testowalność

**0 punktów testowych na 232 sieci** (`TE-001`). Są tylko 4 pady kontrolne:
`J16`/`J17` (UART0_TX/RX) i `J19`/`J20` (USB_D_P/D_N), plus `J11` (TC2030 SWD do STM32).
Brak dostępu do szyn zasilania utrudni bring-up i wyklucza ICT.

### Silkscreen

`board_text_count: 0` — na płytce nie ma żadnego tekstu poza oznaczeniami elementów.
Brak nazwy płytki i oznaczenia rewizji. Świadoma decyzja („płytka będzie bez oznaczeń"),
ale rewizja w rogu obrysu kosztuje zero i rozróżnia serie.

---

## EMC

`analyze_emc.py`: 202 zgłoszenia (59 error, 112 warning, 31 info), 13 kategorii.
Struktura praktycznie niezmieniona od przeglądu nr 1.

### Zgłoszenia potwierdzone geometrią, zamknięte decyzją projektanta

| Reguła | Treść | Decyzja |
|--------|-------|---------|
| `SW-003` | Duża pętla mocy `U3` (`C123` 10,55 mm, `L7` 10,63 mm od układu) i `U8` | „lepiej nie będzie" |
| `ES-001` | `D9` (TVS CAN) 19,2 mm od `J12`, `GD2` 15,5 mm, `FL1` 23,1 mm | „ciężko poprawić w dobry sposób" |
| `BE-001` | Sygnały przy krawędzi: `ETH_RX_P`, `ETH_TX_P`, `SDC0.CLK` (error) + 5 warning | „nic to nie zmienia" |
| `GP-001` / `SU-001` | 224 segmenty na In1.Cu wycinające szczeliny w płaszczyźnie odniesienia | przeniesiono co się dało + przelotki zszywające |
| `PD-001` | Antyrezonans PDN na `+1V8` | odsprzęganie jak na działającej płytce |
| `CK-001` / `CK-003` | Zegary `U1.OSC1` na warstwie zewnętrznej, `J2.CLK` blisko złącza | — |

### `GP-005` — dwie domeny masy

`GND` (261 pinów) i `Earth` (9 pinów: `H1`–`H4`, `GD1`, `GD2`, `C52`, `J12.7`, `J13.3`).
Sprzężenie przez koralik ferrytowy oraz `C52` (1 nF / 2 kV). Poprawna topologia pływającej masy
szasy, potwierdzona przez projektanta.

---

## Fałszywe alarmy i decyzje recenzenta

| Zgłoszenie | Liczba | Uzasadnienie odrzucenia |
|-----------|--------|------------------------|
| **`VM-001`** „3.3V / 1.8V domain crossing" | 13 | Analizator widzi, że `U2` jest zasilany z +3V3, +1V8, +1V25 i +3V0, i zakłada, że GPIO może być 1,8 V. **Wszystkie banki I/O V3S (VCC-IO0–3, VCC-PE0/1) są na +3V3**, a +1V8 zasila wyłącznie VCC-DRAM (DDR2 w obudowie, brak sygnałów zewnętrznych). |
| **`VM-001`** „5.0V / 3.3V domain crossing" | 3 | Nety TXD/RXD/STBY między MCP2518FD a ATA6561. ATA6561 ma osobny pin **VIO na +3.3V**; VCC = +5V zasila tylko stopień liniowy. |
| **`PP-001`** `U11.12` (HPVDD), `U11.8` (HPVSS) | 2 | Potwierdzone kartą katalogową TI: *„Connect the HPVDD pin only to a 2.2 μF, X5R or better, capacitor. **Do not connect HPVDD to an external voltage supply.**"* To wyjścia wewnętrznej pompy ładunkowej. |
| **`PP-001`** `U3.2` (VIN na `DC_IN`) | 1 | `DC_IN` to surowa szyna wejściowa, nie symbol power. ERC KiCada z włączonym testem „power pin not driven" zgłasza 0 naruszeń. |
| **`PS-002`** „plane split" dla +3.3V, +3V3, +3V0, +3V3_GPS, +1V25 | 5 | Union-find analizatora daje fałszywe rozłączenia — udowodnione w przeglądzie nr 1 (ścieżka +3V3 kończy się 0,062 mm od pada `FB1.1`, a analizator twierdzi, że jest odcięta). Szyny są prowadzone ścieżkami, nie płaszczyznami; „12 wysp" oznacza topologię gwiazdy. |
| **`PS-002`/`GP-001`** dla GND | 55+ | Wszystkie 261 padów GND leży na jednej wyspie. |
| **`RP-002`** „crosses +3.3V plane gap" | 6 | `+3.3V` nie jest płaszczyzną odniesienia — jest nią GND na In1/In2. Sygnał nie „widzi" granic wysp szyny, do której się nie odnosi. |
| **`KO-001`** `H1`–`H4` w strefie zakazu | 4 | Cztery strefy 10 × 10 mm w narożnikach mają `copperpour: not_allowed` i są celowo narysowane **wokół** otworów montażowych. |
| **`DFM-001`/`DFM-002`** „annular ring 0,05 mm poniżej minimum" | 2 | Reguła analizatora (`standards-compliance.md`, „verified 2025-01") podaje 0,125 mm. Bieżąca specyfikacja JLCPCB wymaga **różnicy średnica − otwór ≥ 0,1 mm** — spełnione. Stąd też zawyżony poziom DFM `challenging`; realnie **`standard`**. |
| **`CG-AUD`** „J12/J13 has no ground pins" | 2 | Oba mają pin `Earth` (`J12.7`, `J13.3`) — analizator nie rozpoznaje `Earth` jako masy. |
| **`ES-001`** „U9/U10 far from J10/J11" | 2 | USBLC6 chronią pary EPHY po stronie SoC; `J10` to 2-pinowy header RESET, `J11` to Tag-Connect. Zestawienie bez sensu. |
| **`PU-001`** brak pull-upów | 8 | `U3.EN` ma dzielnik UVLO `R38`/`R39`, `U5.NRST` ma `C121`, `U6.SDA/SCL` są na magistrali z `R10`/`R11`. `U6.EXTINT`/`LNA_EN` — piny nieużywane. |
| **`DP-005`** `Y1+`/`Y1−`, `VLED+`/`VLED−` | 2 | To nie są pary różnicowe — kwarc RTC i zasilanie podświetlenia LCD. |
| **`PM-001`** `U6` ↔ `C36`/`C37`/`C38` | 3 | Obrys `U6` zawiera strefę zakazu RF — analizator sam to oznacza jako `info`. |
| **ngspice** `L10/C12`, `L10/C21` | 2 | `L10` to dławik przetwornicy ładowarki AXP209 (LX1), nie filtr LC. Model ideal-passive nie zna pętli regulacji ani impedancji akumulatora. |

---

## Sourcing

| Metryka | Wartość |
|---------|---------|
| Unikalnych części | 106 |
| Z numerem MPN | **38 (35,8 %)** |
| Z kartą katalogową | 20 (18,9 %) |

Zgłoszenie **`SS-001`** (error): pokrycie MPN poniżej 50 % — **bramka przedprodukcyjna**.
Brakuje m.in. dla `U2`, `U3`, `U4`, `U5`, `U6`, `U8`, `Y1`, `J7`, `J8`, `J10`, `L1`, `L3`
i większości elementów biernych.

Konsekwencje wykraczają poza samo zamówienie: bez MPN nie da się zsynchronizować kart
katalogowych, a wtedy **weryfikacja pinoutów pozostaje kontrolą spójności, nie poprawności**.

---

## DFM i uwagi produkcyjne

| Parametr | Wartość | Uwaga |
|----------|---------|-------|
| Warstwy | 4 (F.Cu / In1.Cu / In2.Cu / B.Cu) | |
| Wymiary | 120 × 75 mm | powyżej progu 100 × 100 mm → dopłata |
| Grubość | 1,586 mm | standard 1,6 mm |
| Miedź | 35 µm zewnętrzne, 15,2 µm wewnętrzne | 1 oz / 0,5 oz |
| Wykończenie | HAL lead-free | świadoma decyzja (montaż ręczny) |
| Min. ścieżka | 0,15 mm | ✓ powyżej 0,127 mm |
| Min. wiert | 0,3 mm | ✓ powyżej 0,2 mm |
| Min. przelotka | 0,4 / 0,3 mm (57 szt.) | ✓ opcja `0.3mm/(0.4/0.45mm)`, bez dopłaty |
| Poziom DFM | **standard** | analizator zwraca `challenging` przez nieaktualną regułę pierścienia |
| Szablon | wymagany | 100 % SMD, 309 elementów; przy BGA 0,4 mm rozważyć ramkowy |
| Montaż | dwustronny (219 / 90) | **brak fiducialów** |

**Fiduciale** (`FD-001`, error na obu stronach): F.Cu 207 elementów SMD, B.Cu 90,
najdrobniejszy pad 0,20 mm, obecny BGA 0,4 mm (`IC4` MAX2674). Przy montażu maszynowym
to blokada precyzyjnego pick-and-place — 3 na stronę, w narożnikach, asymetrycznie.
Przy montażu ręcznym pozycja bez znaczenia.

**26 nietentowanych przelotek w padach** (`VP-001`), zestaw niezmieniony od przeglądu nr 2:
`L8` (7 ×, dławik DCDC2), `IC2:7` (pad termiczny TPS61240), `C12`, `C137`, `C18`, `C20`, `C24`,
`C26`, `C43`, `C44`, `C45`, `C48`, `C74`, `C85`, `D3:2`, `D8:1`, `J7:2`, `J16:1`, `J17:1`, `R2:2`.
Wszystkie mają tę samą sieć co pad. Przy dużych padach `L8`/`IC2` ryzyko zaciągnięcia spoiwa
jest małe.

**3 rozjazdy footprintów z bibliotekami** (`T1`, `U11`, `D42`):
- `U11` — instancja w PCB ma **22 pady**, biblioteka `WQFN-16-1EP_..._ThermalVias` ma **26**.
  Brakujące 4 to wbudowane przelotki termiczne w padzie 17 (`thru_hole`, Ø 0,5 mm, wiert 0,2 mm).
  Aktualizacja z biblioteki je przywróci i podniosłaby licznik przelotek termicznych z 4 do 8.
- `T1`, `D42` — różnica czysto kosmetyczna (nowszy format grafiki w bibliotece).
  Pady identyczne pod względem liczby, typu, kształtu i rozmiaru.

---

## Nie wykonano / ograniczenia przeglądu

- **Karty katalogowe: brak katalogu `datasheets/`**, brak kluczy API dystrybutorów
  (`DIGIKEY_CLIENT_ID` itd. nieustawione) — zgłoszenie `DS-002` zasadne.
  Weryfikacja wobec kart producenta objęła wyłącznie **TPA6132A2** (przez wyszukiwanie webowe).
  Dla **V3S** i **AXP209** podstawą była własna dokumentacja projektu (`doc/*_ROZPISKA.md`) —
  solidna, ale wtórna. **Pozostałe stwierdzenia o pinach to kontrola spójności, nie poprawności.**
- **H1102NL, piny 10 i 15** — nierozstrzygnięte przez brak czytelnej karty (PDF producenta
  jest skanem bez warstwy tekstowej). Zamknięte decyzją projektanta.
- **Analiza termiczna** — 0 zgłoszeń przy zerowych danych o mocy strat. Wynik 100/100
  nie jest potwierdzeniem. Powtórzyć po uzupełnieniu MPN.
- **Audyt cyklu życia komponentów: nie wykonano** — brak kluczy API dystrybutorów.
- **Analiza Gerberów: nie wykonano** — w projekcie nie ma jeszcze plików produkcyjnych.
- **Weryfikacja mechaniczna** złączy wystających poza obrys wymaga modelu obudowy —
  poza zakresem analizy plików KiCad.

---

## Mocne strony projektu

1. **Trzy niezależne mechanizmy kontroli są czyste jednocześnie** — ERC 0, DRC 0 niepołączonych
   i 0 rozjazdów parity, ngspice 0 fail. Przy 309 elementach i 242 sieciach to solidny wynik.
2. **Wszystkie blokery z dwóch poprzednich przeglądów naprawione**, łącznie z czterema regresjami,
   które sam wprowadziłeś przy poprzedniej rundzie poprawek.
3. **Ochrona wejścia z prawdziwego zdarzenia:** PTC 2 A + dwa GDT do Earth + TVS
   SMBJ33CA/SMBJ36A + dwie Schottky PMEG6030EP równolegle na odwrotną polaryzację + koraliki.
4. **Przełączana programowo terminacja CAN** na przekaźnikach PhotoMOS, dzielona 59 + 59 Ω
   z bypassem AC 4,7 nF.
5. **Rozdzielona masa szasy `Earth`** z pierścieniem obwodowym, otworami montażowymi
   i sprzężeniem przez koralik oraz 1 nF / 2 kV.
6. **Odsprzęganie bardzo blisko układów** — 0,21–0,28 mm dla `U7`, `U2`, `U5`, `U9`.
7. **Wszystkie napięcia wyjściowe potwierdzone symulacyjnie**, nie tylko rachunkowo:
   `U4` → Vfb 0,600 V przy błędzie 0,0 %, 22 filtry RC w granicach 0,35 %.
8. **`In2.Cu` praktycznie lita** (1 segment ścieżki) — dobra ciągła płaszczyzna odniesienia.
9. **Kondensatory TPA6132A2 w promieniu wymaganym przez TI**, wartość HPVDD podniesiona
   do 2,2 µF zgodnie z kartą katalogową.
10. **Własna dokumentacja projektowa** (`doc/*_ROZPISKA.md`, 3334 wiersze) była na tyle dokładna,
    że pozwoliła wykryć dwie najpoważniejsze rozbieżności pierwszego przeglądu — kwarc 24 MHz
    i źródło LDO24IN/LDO3IN. To realny atut projektu.

---

## Luki analizatorów

1. **`connectivity_graph` (union-find) daje fałszywe rozłączenia.** Nie ufać `PS-002`
   ani pochodnym `RP-002`/`GP-001`. Autorytatywne: `kicad-cli pcb drc` → `unconnected_items`.
   W przebiegu 2 analizator raportował `routing_complete: true` przy dwóch realnie urwanych
   sieciach GPS — czyli myli się w obie strony.
2. **`validate_voltage_levels` przypisuje układowi wszystkie domeny, z których jest zasilany**,
   zamiast śledzić rail I/O konkretnego pinu — stąd 16 fałszywych alarmów na V3S i ATA6561.
3. **`analyze_thermal.py` zwraca 100/100 przy zerowych danych wejściowych.**
4. **`power_budget` zwraca 10 mA na każdy układ** (wartość domyślna) i błędnie przypisuje
   regulatory. Bezużyteczne bez kart katalogowych.
5. **Tabela możliwości fabów w skillu jest nieaktualna** (`standards-compliance.md`,
   „verified 2025-01") — podaje pierścień 0,125 mm, JLCPCB wymaga różnicy średnica−otwór ≥ 0,1 mm.
   Sprawdzać bieżącą stronę producenta.
6. **Backend LTspice nie emituje `.meas` ani nie parsuje `.raw`** — zwraca same `skip`.
   **Używać ngspice** (`D:\Spice64\bin`): te same 80 symulacji w 7,7 s, zero skipów.
7. **`XV-002` z `cross_analysis.py` to jedyny mechanizm łapiący rozjazd wartości schemat ↔ PCB.**
   KiCad DRC („schematic parity") sprawdza wyłącznie łączność, więc poprawka wartości wprowadzona
   tylko w schemacie przechodzi bez śladu i trafia do BOM-u ze starą wartością.
   W przebiegu 2 wykryło 41 takich rozjazdów, w tym `C131` (2,2 µF vs 1 µF).
   **Uruchamiać po każdej sesji edycji schematu.**
8. **`diff_analysis.py` wypisuje wyłącznie zmienione reguły** — brak reguły na liście oznacza
   „bez zmian", nie „naprawione". Ta pułapka spowodowała błędne sprostowanie w przeglądzie nr 2.
9. **Skrypty wymagają `PYTHONUTF8=1`** — bez tego wywalają się na polskich znakach
   i symbolach `→`/`²` przy domyślnym kodowaniu cp1250.

---

## Pozostałe działania

Lista do odhaczania: [`TODO.md`](../TODO.md) w katalogu głównym projektu.

---

*Raport wygenerowany przez skill `kicad-happy:kicad` v1.3.1.
Surowe wyniki: `analysis/2026-08-01_1538/{schematic,pcb,cross_analysis,emc,thermal,spice}.json`.*
