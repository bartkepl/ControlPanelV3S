# AXP209 — pełna rozpiska pinów, drzewo zasilania i połączenia z V3S

Dokument uzupełniający do [`TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md`](TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md) i
[`V3S_PINOUT_ROZPISKA.md`](V3S_PINOUT_ROZPISKA.md) — pełna rozpiska AXP209 (49 pinów: 48 + pad),
przydział DC/DC1-3 i LDO1-4, realne, prześledzone po przewodach wartości dławików/kondensatorów
trzech zewnętrznych przetwornic (TPS54360, TPS61240, SY8088), oraz połączenia AXP209↔V3S
(I2C, IRQ, PWRON, Reset, ADC pomiaru napięcia wejściowego).

Od 2026-07-25 trzecie urządzenie na tej samej magistrali TWI1 to STM32 (kontroler audio/
przycisków headsetu) — zob. sekcja 11 dok. nadrzędnego i
[`STM32_AUDIO_BUTTONS_ROZPISKA.md`](STM32_AUDIO_BUTTONS_ROZPISKA.md). Nie zmienia niczego w
drzewie zasilania AXP209 poniżej (STM32 zasilany z VCC-IO/DCDC3, nie z LDO AXP209).

Stan na 2026-07-24 (bez zmian od dodania STM32 2026-07-25, patrz akapit wyżej).

## Źródła i metoda

- **Numery pinów AXP209, nazwy, typ** — wyciągnięte z dwóch niezależnych symboli KiCad w tym
  repo: `pcb/local_lib/V3S_AXP209.kicad_sym` (symbol `AXP20`) i symbol biblioteczny
  `SamacSys_Parts:AXP209` osadzony w `pcb/cpu_pwr.kicad_sch`. **Obie transkrypcje pinów zgadzają
  się co do jednego** (49/49 pinów, te same numery/nazwy) — potraktowane jako mocne
  potwierdzenie poprawności numeracji. **Zgodnie z instrukcją, nie użyto realnego stanu
  narysowanego schematu ani z `cpu_pwr.kicad_sch`, ani z `cpu_pwr_wip.kicad_sch`** — oba są
  pracą w toku przy tym samym symbolu i nie zawierają jeszcze żadnych realnych połączeń (sam
  plik `cpu_pwr.kicad_sch` to w tej chwili tylko postawiony symbol IC1 + kilka wiszących flag
  zasilania, bez jednego wire'a) — użyto z niego wyłącznie definicji pinów symbolu, nie topologii.
- **Napięcia/prądy DC/DC1-3, LDO1-4, aplikacyjne wartości L/C, PWRON, IRQ, GPIO ADC, PWROK** —
  z oficjalnego **AXP209 Datasheet v1.0 (X-Powers)**, w tym bezpośredni odczyt schematu
  aplikacyjnego (§3, str. 5) i tabeli opisu pinów (§7, str. 12-13), skrzyżowany z jądrem Linux
  (`drivers/input/misc/axp20x-pek.c`) i realnym produktem **FunKey S** (AXP209 + V3s, ten sam
  chip, ten sam SoC), którego schemat i dyskusja projektowa („PMIC default voltages”) potwierdzają
  docelowe napięcia z tego dokumentu.
- **Realne wartości dławików/kondensatorów/dzielników trzech zewnętrznych przetwornic** — z
  **`pcb/dcdc.kicad_sch`** (jedyny plik z realnym schematem tych przetwornic, zgodnie z
  instrukcją), **prześledzone po współrzędnych pinów i realnych segmentach `(wire ...)`**, nie na
  oko po samej bliskości elementów na arkuszu — tam gdzie topologia była nieoczywista (dużo
  podobnie ułożonych rezystorów), zweryfikowano dokładnym dopasowaniem współrzędnych pinów IC do
  końców przewodów.
- Jak w poprzednich dokumentach: pozycje niepewne oznaczone **(do potwierdzenia)**.

---

## 1. Obudowa

**AXP209 — QFN-48, 6×6 mm, rozstaw 0.4 mm, z eksponowanym paddem (EP)**. Footprint w tym repo:
`QFN40P600X600X80-49N-D` (0.4mm pitch / 6.00×6.00mm / 0.8mm wysokości / 49 pól lutowniczych = 48
wyprowadzeń + 1 EP). EP (pin 49) — masa, lutować do płaszczyzny GND.

---

## 2. Drzewo zasilania AXP209 (zdecydowane w tej rozmowie)

| Regulator | Napięcie/prąd | Zasila | Sposób ustawienia napięcia |
|---|---|---|---|
| **DC/DC1** | — (nie jest to regulator wyjściowy) | **Wewnętrzna ładowarka PWM Li-ion** (fixed function) | brak — to nie jest konfigurowalna szyna, tylko dedykowany stopień ładowania baterii. Potwierdzone wprost w datasheet: *"DC-DC1: The PWM charger"*, w odróżnieniu od DC-DC2/DC-DC3 opisanych tam jako "adjustable". To dokładnie odpowiada założeniu z brifu: "DC/DC1 bateria to jest by design" |
| **DC/DC2** | **1.25 V / 1.6 A** | V3S **VDD-CPU + VDD-SYS** (rdzeń + logika systemowa — w każdym sprawdzonym projekcie referencyjnym to jedna fizyczna szyna, zob. `V3S_PINOUT_ROZPISKA.md`) | I2C, **REG23H[5:0]**, `Vout = 0.7 + kod×0.025V`, krok 25mV, brak zewnętrznego dzielnika FB |
| **DC/DC3** | **3.3 V / 1.2 A** | V3S **VCC-IO** (Port B/C/F/G); prawdopodobnie też VCC-USB, VCC-MCSI, VCC-PE (te same 3.3V co VCC-IO w tym projekcie, bez kamery CSI) — wzorem FunKey S, gdzie DC3 opisane jako `Vcc_io/Vcc_ephy/Vcc_mcsi/Vusb` | I2C, **REG27H[6:0]**, `Vout = 0.7 + kod×0.025V`, krok 25mV, brak zewnętrznego dzielnika FB |
| **LDO1** | **3.3 V, Always-On** | V3S **VCC-RTC** + **GPS MAX-M10C V_BCKP** (backup, sekcja 1.2/7 dok. nadrzędnego — jedna bateria/LDO obsługuje RTC hosta i GPS jednocześnie) | strap **LDO1SET → VINT** (patrz tabela pinów, pin 27) |
| **LDO2** | **3.0 V / 200 mA** | V3S **AVCC + VCC-PLL** (wg `V3S_PINOUT_ROZPISKA.md`); dodatkowo, jak ustalono przy śledzeniu `dcdc.kicad_sch`, sieć **`+3V0`** zasila też pull-up EN przetwornicy SY8088 (R31=100k) — czyli LDO2 pośrednio bramkuje włączenie regulatora DRAM | rejestr I2C (REG28H, do potwierdzenia dokładnego adresu) |
| **LDO3** | 3.3 V (typowo) | **propozycja: GPS (MAX-M10C)** — osobna, "cicha" domena liniowa z dala od przełączanych zakłóceń CAN/Ethernet, korzystne dla czułego frontendu RF GPS | — |
| **LDO4** | 3.3 V (typowo) | **propozycja: logika CAN** (MCP2518FD VDD + ATA6561 VIO, obie 3.3V, niski pobór) | — |

LDO3/LDO4 to **propozycja, nie ostateczna decyzja** — zgodnie z brifem ("do użycia według
uznania, nie mam planu"). Podział wynika z chęci elektrycznego odseparowania czułego GPS od
przełączanego cyfrowego CAN, każde na osobnej LDO. Do przemyślenia przy layoucie.

**Rezerwa prądowa LDO2 (200mA)**: obsługuje AVCC+VCC-PLL (pobór rzędu pojedynczych/kilkunastu mA)
oraz pull-up SY8088 EN (R31=100kΩ z `+3V0` — prąd rzędu ~30µA przy pełnym naładowaniu, pomijalny)
— zapas prądowy komfortowy.

---

## 3. Zewnętrzne przetwornice DC/DC (poza AXP209, schemat: `pcb/dcdc.kicad_sch`)

### 3.1 TPS54360DDA — 8.5-60V EXTERNAL → 5V/3.5A ACIN

Zasila **bezpośrednio pin ACIN AXP209** (potwierdzone: szyna VOUT tej przetwornicy łączy się wprost,
bez żadnych elementów pośrednich, z symbolem zasilania nazwanym „ACIN").

| Element | Wartość/ref | Rola |
|---|---|---|
| Wejście `DC_IN` | — | 8.5-60V zewnętrzne (M12, sekcja 2 dok. nadrzędnego) |
| C5, C6 | 2×2.2µF/100V | kondensatory wejściowe (równolegle, ~4.4µF, rating 100V pod transienty) |
| D1 | STPS5L60S (Schottky 5A/60V) | dioda catch/freewheel na węźle SW — **TPS54360 to układ asynchroniczny** (bez wewnętrznego synchronicznego FET-a dolnego), wymaga zewnętrznej diody |
| L7 | 8.2 µH | dławik wyjściowy (SW→VOUT) |
| C58 | 100n | kondensator bootstrap (BOOT) |
| C62, C66, C61 | 10µF + 10µF + 100n | kondensatory wyjściowe na szynie VOUT/ACIN |
| **R36 (top, 53.6k) / R37 (bottom, 10.2k)** | dzielnik FB | `Vout = 0.8V × (1 + 53.6k/10.2k) = 5.00 V` — **zgodne z etykietą "5V/3.5A"** |
| R35 (13k) + C71 (6.8nF), równolegle C68 (39pF) | sieć COMP | standardowa 3-elementowa kompensacja typu II na pinie COMP |
| R38 (523k, VIN→EN) / R39 (84.5k, EN→GND) | dzielnik EN/UVLO | próg załączenia ≈ 1.2V × (523+84.5)/84.5 ≈ **8.63 V** — **zgodne z etykietą "8.5-60V EXTERNAL"**; przetwornica załącza się sama napięciem wejściowym, bez sterowania software'owego |
| R34 (162k, RT/CLK→GND) | rezystor częstotliwości | pojedynczy rezystor ustawiający częstotliwość przełączania (nie dzielnik) |

Napięcie i próg UVLO **potwierdzone obliczeniowo i zgodne z opisami na schemacie** — brak
znalezionych niezgodności w tej gałęzi.

### 3.2 TPS61240DRVT — 3.3V → 5V/350mA CAN

Zasila VCC transceivera CAN (ATA6561, sekcja 5.4 dok. nadrzędnego).

| Element | Wartość/ref | Rola |
|---|---|---|
| Wejście | `+3.3V` | z DCDC3 AXP209 (pośrednio) |
| C46 | 2.2µF | kondensator wejściowy |
| L1 | 1 µH | dławik boost |
| C47 | 4.7µF | kondensator wyjściowy na `+5V`/CAN |
| **EN** | zwarte wprost do VIN (`+3.3V`) | **brak sterowania software'owego** — boost włącza się zawsze, gdy jest 3.3V |
| **FB** | zwarte wprost do VOUT | **stały wariant 5.0V** (nie regulowany zewnętrznym dzielnikiem) — potwierdzone, że to nie wersja adjustable |

Brak rezystorów w tej gałęzi w ogóle — żaden z R31-R39 do niej nie należy (sprawdzone
wprost po przewodach).

### 3.3 SY8088AAC — IPSOUT → 1.8V/1.0A Vdram

| Element | Wartość/ref | Rola |
|---|---|---|
| Wejście | `IPSOUT` | z AXP209 (bateria/ACIN, wybrane przez IPS) |
| C51 | 10µF | kondensator wejściowy |
| L2 | LQH3NPN2R2MMEL, 2.2µH (Murata, 1212/3030) | dławik wyjściowy |
| C54, C57 | 100n + 10µF | kondensatory wyjściowe na szynie `+1V8` |
| R31 (100k) | pull-up EN z `+3V0` (LDO2 AXP209) | włącza SY8088, gdy LDO2 jest aktywne — brak dzielnika, sam pull-up |
| **R32 (top, 200k) / R33 (bottom, 100k)** | dzielnik FB | `Vout = 0.6V × (1 + 200k/100k) = 1.80 V` — **zgodne z etykietą "1.8V/1.0A Vdram"** |

Przy pierwszym śledzeniu (2026-07-24) R32 miało wartość 100k (dawałoby policzone 1.20V zamiast
wymaganych 1.8V dla VCC-DRAM V3S) — poprawione tego samego dnia na 200k, zweryfikowane wprost w
`pcb/dcdc.kicad_sch`. Obecny stan liczy się poprawnie na 1.8V.

---

## 4. Pełna tabela pinów AXP209 (49 = 48 + EP)

| Pin | Nazwa | Gałąź / funkcja | Przydział w tym projekcie |
|---|---|---|---|
| 1 | SDA | I2C dane | → **TWI1** V3S (PE22), pull-up **2.2kΩ do 3.3V** (potwierdzone wprost w datasheet) |
| 2 | SCK | I2C zegar | → **TWI1** V3S (PE21), pull-up **2.2kΩ do 3.3V** |
| 3 | GPIO3 | GPIO ogólne (open-drain) | wolne — rezerwa |
| 4 | N_OE | Power on/off switch: GND=on, IPSOUT=off | **zewrzeć do GND** (zawsze zezwolone na włączenie; PWRON/PEK obsługuje miękki przycisk) |
| 5 | GPIO2 | GPIO ogólne | wolne — rezerwa |
| 6 | N_VBUSEN | VBUS→IPSOUT select: GND=wybierz VBUS, High=nie wybieraj | **podciągnąć do stanu wysokiego** (VINT/3.3V) — VBUS całkowicie nieużywane (sekcja 1.3 dok. nadrzędnego) |
| 7 | VIN2 | wejście DC/DC2 | z IPSOUT |
| 8 | LX2 | węzeł przełączający DC/DC2 | dławik **4.7µH** (cel <2.5V → większa indukcyjność wg zalecenia datasheet) |
| 9 | PGND2 | masa mocy DC/DC2 | GND |
| 10 | DCDC2 | wyjście/FB DC/DC2, ustawiane **I2C REG23H** | **1.25V/1.6A → V3S VDD-CPU+VDD-SYS**; kondensator ≥10µF X7R |
| 11 | LDO4 | wyjście LDO4 | propozycja: **CAN logika 3.3V**; kondensator 4.7µF |
| 12 | LDO2 | wyjście LDO2 | **3.0V/200mA → V3S AVCC+VCC-PLL** + pull-up EN SY8088; kondensator 4.7µF |
| 13 | LDO24IN | wspólne wejście LDO2+LDO4 | z IPSOUT |
| 14 | VIN3 | wejście DC/DC3 | z IPSOUT |
| 15 | LX3 | węzeł przełączający DC/DC3 | dławik **2.2µH** (cel >2.5V) |
| 16 | PGND3 | masa mocy DC/DC3 | GND |
| 17 | DCDC3 | wyjście/FB DC/DC3, ustawiane **I2C REG27H** | **3.3V/1.2A → V3S VCC-IO** (+VCC-USB/VCC-MCSI/VCC-PE); kondensator ≥10µF X7R |
| 18 | GPIO1 | GPIO/ADC (zakres rozszerzony 0.7-2.7475V w trybie ADC) | wolne — zakres rozszerzony słabo pasuje do szerokiego dzielnika 8.5-60V (zob. sekcja 6), zostawić jako zwykłe GPIO |
| 19 | GPIO0 | GPIO/ADC (zakres domyślny 0-2.0475V) | **ADC pomiaru napięcia wejściowego 8.5-60V** — zob. sekcja 6 |
| 20 | EXTEN | wyjście, zał. zewn. zasilania pomocniczego (REG12H bit0) | **NC** (nieużywane) |
| 21 | APS | węzeł sensu zasilania systemowego | **zewrzeć z IPSOUT** (0Ω) |
| 22 | AGND | masa analogowa | masa analogowa/cicha |
| 23 | BIAS | referencja prądowa | **rezystor precyzyjny 200kΩ 1% do AGND** (wprost z datasheet) |
| 24 | VREF | wewnętrzna referencja napięcia | **kondensator 1µF do AGND** |
| 25 | PWROK | wyjście power-good | zob. sekcja 7 (reset V3S) |
| 26 | VINT | wewnętrzne zasilanie logiki 2.5V (wyjście) | **kondensator bypass 1µF do AGND** |
| 27 | LDO1SET | strap wyboru domyślnego LDO1 | **→ VINT** (daje LDO1=3.3V; GND dałoby 1.3V) |
| 28 | LDO1 | wyjście LDO1 | **3.3V Always-On → V3S VCC-RTC + GPS V_BCKP**; kondensator 1µF |
| 29 | DC3SET | strap wyboru domyślnego DC/DC3 | **→ APS** (docelowo 3.3V wg tabeli stanów, ale **niepewne** — sekcja "Default configuration instructions" brakuje w datasheet; i tak nadpisać wcześnie w SPL/U-Boot przez I2C REG27H) |
| 30 | BACKUP | bateria podtrzymania RTC | **LIR2032** (już zdecydowane, sekcja 1.2 dok. nadrzędnego) |
| 31 | VBUS | USB VBUS do PMIC | **NC** (nieużywane, sekcja 1.3) |
| 32/33 | ACIN_1/ACIN_2 | wejście ACIN (2 fizyczne piny) | z wyjścia **TPS54360 (5V)** — zob. sekcja 3.1 |
| 34/35 | IPSOUT_1/IPSOUT_2 | główna szyna systemowa (2 fizyczne piny) | zasila: SY8088 VIN, VIN2, VIN3, LDO24IN, LDO3IN, VIN1 (ładowarka), APS |
| 36 | CHGLED | LED statusu ładowania (open-drain) | opcjonalny wskaźnik LED + rezystor do 3.3V |
| 37 | TS | sens temperatury baterii (NTC) | **zewrzeć do GND** — pakiet LiPo z wbudowanym PCM, brak NTC; AXP209 **automatycznie wyłącza** monitorowanie temperatury przy TS=GND (potwierdzone wprost w datasheet); dodatkowo można wyłączyć czujnik programowo REG84H[1:0]=00 |
| 38/39 | BAT_1/BAT_2 | ogniwo LiPo (2 fizyczne piny) | złącze **SM02B-SRSS-TB** (sekcja 1.1) |
| 40 | LDO3IN | wejście LDO3 | z IPSOUT |
| 41 | LDO3 | wyjście LDO3 | propozycja: **GPS 3.3V**; kondensator 4.7µF |
| 42 | BATSENSE | Kelvin-sense prądu ładowania (strona BAT) | przy złączu BAT, przez rezystor sense **0.03Ω**; kondensator 10µF do GND |
| 43 | CHSENSE | Kelvin-sense prądu ładowania (strona ładowarki/LX1) | druga strona rezystora 0.03Ω; kondensator 10µF do GND |
| 44 | VIN1 | wejście ładowarki (DC/DC1) | z **IPSOUT** (nie z BAT); kondensator 10µF |
| 45 | LX1 | węzeł przełączający ładowarki (DC/DC1) | dławik **4.7µH** do węzła CHSENSE→BAT |
| 46 | PGND1 | masa mocy ładowarki | GND |
| 47 | PWRON | przycisk zasilania (PEK) | **wewnętrzny pull-up 100kΩ do APS — nie trzeba zewnętrznego pull-upa**; zalecany zewn. obwód: przycisk chwilowy do GND + rezystor szeregowy **1kΩ** + kondensator filtrujący **100pF** do GND; domyślne czasy: start 3s, long-press 1.5s, shutdown 6s, opóźnienie PWROK 64ms (REG36H=5DH) |
| 48 | IRQ_/WAKEUP | przerwanie (open-drain) | pull-up **51kΩ do 3.3V** (wprost z datasheet — nie 10k, jak wstępnie szacowano); → **V3S EINT: PB5** (zgodnie z `V3S_PINOUT_ROZPISKA.md`) |
| 49 | EP | pad eksponowany | masa (GND) |

---

## 5. Połączenia AXP209 ↔ V3S — podsumowanie

| Sygnał | AXP209 | V3S | Uwaga |
|---|---|---|---|
| I2C SDA | pin 1 | PE22 (TWI1_SDA) | wspólna magistrala z TSC2007, pull-up 2.2kΩ |
| I2C SCK | pin 2 | PE21 (TWI1_SCK) | jw. |
| IRQ (PEK/zdarzenia) | pin 48 | PB5 (PB_EINT5) | pull-up 51kΩ; multipleksuje PEK short/long press, insert/remove ACIN itd. |
| Przycisk zasilania | pin 47 (PWRON) | — (obsłużone sprzętowo w AXP209) | `axp20x-pek` w mainline, generuje `KEY_POWER` |
| Reset | pin 25 (PWROK) | pin 99 (RESET) | **rozważane, nie zdecydowane — zob. sekcja 7** |
| ADC napięcia wejściowego | pin 19 (GPIO0) | — (odczyt przez I2C, nie bezpośrednie podłączenie do V3S) | dzielnik 8.5-60V→ADC, zob. sekcja 6 |

---

## 6. Pomiar napięcia wejściowego 8.5-60V przez GPIO0 ADC

Z datasheetu (potwierdzone): GPIO ADC to **12-bit, 0.5mV/LSB**; zakres wybierany rejestrem
**REG85H** — bit=0 (domyślnie): **0-2.0475V**; bit=1: **0.7-2.7475V**. Włączenie ADC: **REG83H**
(bity GPIO0/GPIO1 ADC enable — **nie REG82H**, jak wstępnie zakładano w dok. nadrzędnym), oraz
funkcja pinu musi być ustawiona na ADC osobno (**REG90H[2:0]=100** dla GPIO0).

**Rekomendacja (propozycja, do weryfikacji przy layoucie)**: użyć **GPIO0 w zakresie domyślnym
(0-2.0475V)** — zakres rozszerzony (0.7-2.7475V) źle pasuje do tak szerokiego zakresu wejściowego,
bo przy niskim końcu (8.5V) po przeskalowaniu spadalibyśmy poniżej progu 0.7V tego trybu.

Proponowany dzielnik: **R_top = 330kΩ** (od węzła pomiarowego 8.5-60V), **R_bottom = 10kΩ** (do
GND, równolegle z wejściem GPIO0) → współczynnik ≈ 1:34 (10/340):

| V_in | V_ADC (obliczone) | Margines do 2.0475V |
|---|---|---|
| 8.5 V (min) | 0.25 V | — (496 kodów z 4096, dobra rozdzielczość) |
| 24 V (nominalne, sekcja 2 dok. nadrzędnego) | 0.71 V | — |
| 60 V (max wg specyfikacji) | 1.77 V | ~13% zapasu |

To **propozycja projektowa**, nie wartość z datasheetu — do potwierdzenia: (a) rzeczywisty
absolutny limit napięcia na pinie GPIO (nie tylko "zakres ADC") przy transientach powyżej 60V, (b)
czy warto dodać TVS/ogranicznik na węźle dzielnika pod kątem tych samych transientów, o których
mowa w sekcji 2 dok. nadrzędnego (load-dump, do 40V+ marginesu).

---

## 7. Reset V3S z sygnału PWROK AXP209 — do rozważenia

Z datasheetu (potwierdzone, §9.1): *"PWROK w AXP209 może być użyty jako sygnał resetu systemu.
Podczas startu AXP209, PWROK wyprowadza stan niski, który następnie jest podciągany wysoko i
resetuje system po osiągnięciu regulacji przez wszystkie napięcia wyjściowe. W normalnej pracy
AXP209 stale monitoruje napięcie i obciążenie — przy przeciążeniu lub podnapięciu PWROK natychmiast
wraca do stanu niskiego, resetując system i zapobiegając utracie danych."* Opóźnienie po
ustabilizowaniu napięć: **64ms domyślnie** (REG36H bit2, potwierdzone).

**Realny precedens**: Pine64 Clusterboard (AXP803 + Allwinner A64, ta sama rodzina PMIC i ten sam
producent SoC) dokumentuje dokładnie tę technikę: *"PWROK jest podpięty do AP-RESET# na SoC A64."*
To potwierdza, że pomysł jest sensowny i praktykowany — **ale ten sam dokument opisuje też realny
problem**: sprzężenie zwrotne (back-EMF) z sieci resetu zakłócało własne sekwencjonowanie PMIC,
wymagając diody Schottky do izolacji.

**Kluczowe zastrzeżenie specyficzne dla tego projektu**: PWROK monitoruje wyłącznie **własne**
wyjścia AXP209 (DC/DC2, DC/DC3, LDO1-4) — **nie** monitoruje szyny DRAM (1.8V), która w tym
projekcie pochodzi z **zewnętrznej** przetwornicy SY8088 (sekcja 3.3), całkowicie poza kontrolą
AXP209. Podpięcie PWROK wprost do RESET V3S **nie gwarantuje**, że pamięć DDR2 jest już stabilna,
gdy reset się zwalnia.

Łagodząca okoliczność znaleziona przy śledzeniu `dcdc.kicad_sch`: SY8088 EN jest podciągnięty
(R31=100k) z sieci `+3V0`, czyli **z LDO2 AXP209** — więc regulator DRAM zaczyna się załączać
mniej więcej w tym samym oknie czasowym co własne szyny AXP209, nie zupełnie niezależnie. To
zmniejsza, ale nie eliminuje ryzyko (DDR2 ma swój własny czas narastania po załączeniu EN, a
64ms to niewiele czasu na pewność stabilizacji dodatkowego, zewnętrznego stopnia).

**Dwie opcje do rozważenia:**

1. **(Rekomendacja, bezpieczniejsza)** PWROK → wolny pin EINT/GPIO V3S (np. jedno z pinów
   `PB6/PB7`, rezerwa EINT z `V3S_PINOUT_ROZPISKA.md`) jako **przerwanie "power fault"**, nie
   twardy reset. Firmware reaguje programowo (bezpieczne wyłączenie/zapis stanu) zamiast
   nieskontrolowanego resetu sprzętowego w trakcie np. zapisu do eMMC. RESET V3S zostaje na
   niezależnym obwodzie RC (10nF + pull-up, już ustalone w `V3S_PINOUT_ROZPISKA.md`, pkt. 99).
2. **(Alternatywa, jeśli zależy na sprzętowym auto-reset przy zaniku zasilania)** PWROK → RESET
   V3S bezpośrednio (przez prosty bufor/diodę), *pod warunkiem* zweryfikowania empirycznie na
   prototypie, że 64ms + czas narastania SY8088 wystarcza, zanim V3S zacznie inicjalizować DDR2;
   rozważyć diodę izolującą (wzorem Pine64 Clusterboard) między siecią resetu a PWROK, jeśli
   sprzężenie zwrotne okaże się problemem w bring-up.

Nie podjęto tu ostatecznej decyzji — zgodnie z prośbą, to "rozważenie", z rekomendacją opcji 1
jako bezpieczniejszej domyślnej.

---

## 8. Otwarte pytania / do potwierdzenia

- **Dokładna wartość rezystora DC3SET** dającego pewne 3.3V na starcie (przed przejęciem kontroli
  przez I2C) — sekcja datasheet "Default configuration instructions" jest przywoływana, ale
  nieobecna w dostępnym PDF. Nie blokuje projektu (firmware i tak ustawi REG27H wcześnie w
  SPL/U-Boot), ale warto zweryfikować bezpośrednio z X-Powers/FAE przy BOM.
- **Adres rejestru LDO2 (REG28H?)** — użyty tu z pamięci wzorem sąsiednich rejestrów DCDC2/DCDC3,
  do potwierdzenia bezpośrednio w datasheet przy pisaniu kodu inicjalizacji.
- **Absolutny limit napięcia na pinie GPIO0** przy transientach powyżej 60V (nie tylko zakres ADC)
  — do sprawdzenia przed finalizacją dzielnika z sekcji 6.
- **PWROK: push-pull czy open-drain** — datasheet nie precyzuje wprost (wywnioskowane jako
  push-pull z braku adnotacji "open drain", jaką ma np. CHGLED/GPIO3) — istotne przy projektowaniu
  ewentualnego łączenia z siecią RESET (opcja 2 z sekcji 7).

---

## 9. Przegląd szkicu PMIC z 2026-07-25 — lista do domknięcia

Przegląd zrzutu ekranu roboczego szkicu arkusza "PMIC" (U2/AXP209) — **ten szkic w chwili
przeglądu nie odpowiada jeszcze zawartości `pcb/cpu_pwr_wip.kicad_sch`** (plik nie zawiera tego
okablowania), więc poniższe traktować jako listę do przeniesienia/domknięcia przy realnym
rysowaniu w KiCad, nie jako opis stanu repo.

**Kompletność**: wszystkie 48 pinów sygnałowych + EP (pad, 9× narysowane) obecne — nic nie
brakuje.

**Zgodne z resztą tego dokumentu** (potwierdzone na szkicu):
- Kondensatory LDO1-4: 1µF/4.7µF/4.7µF/4.7µF — dokładnie jak w sekcji 2/4.
- VREF: 1µF — zgodne z sekcją 4 (pin 24).
- BIAS: rezystor 200kΩ 1% (R2) — dokładnie zgodne z sekcją 4 (pin 23).
- BATSENSE/CHSENSE: rezystor sense 0.03Ω (R13) między węzłem BAT a LX1/ładowarką — poprawna
  topologia Kelvin-sense z sekcji 4 (piny 42/43).
- PWROK poprowadzone w stronę sygnału "reset" (przez R12=47k) — realizacja w duchu opcji 2 z
  sekcji 7 (PWROK→RESET bezpośrednio); pamiętać o zastrzeżeniu stamtąd (PWROK nie monitoruje
  szyny DRAM z SY8088).

**Do poprawy / potwierdzenia przed przeniesieniem do KiCad (wg ważności):**

1. **GPIO0/GPIO1 oznaczone jako NC** — funkcja ADC pomiaru napięcia wejściowego 8.5-60V (sekcja 6)
   jeszcze niepodpięta; zdjąć NC z GPIO0 i dociągnąć proponowany dzielnik 330k/10k.
2. **N_VBUSEN wygląda na współdzielenie sieci z AGND/PGND/EP** (czyli GND) — sprzeczne z
   zaleceniem z sekcji 4 (pin 6): musi być **HIGH** (do VINT/3.3V), bo GND = "wybierz VBUS", a
   VBUS ma być całkowicie nieużywane (sekcja 1.3 dok. nadrzędnego). Priorytet: sprawdzić najpierw.
3. **DC3SET (pin 29) wygląda na niepodłączony** — podciągnąć do APS (dla domyślnego 3.3V) albo
   GND, nie zostawiać w powietrzu nawet jeśli firmware nadpisze przez I2C.
4. **Brak widocznych pull-upów I2C (2.2kΩ SDA/SCK) i pull-upu IRQ (51kΩ)** na tym arkuszu —
   potwierdzić, że istnieją gdzieś w projekcie (np. przy V3S/TSC2007), albo dodać.
5. **VBUS (pin 31) ma etykietę sieci + kondensator 4.7µF zamiast NC** — potwierdzić, czy celowe
   (np. złącze testowe), czy zostawić całkiem odłączone zgodnie z sekcją 1.3 dok. nadrzędnego.
6. **BACKUP (pin 30): widoczny tylko C6=100µF** — potwierdzić, że złącze LIR2032 (sekcja 1.2 dok.
   nadrzędnego) jest podłączone do tej samej sieci gdzieś na arkuszu.
7. **LDO1SET wygląda na zwarte z VINT** (zgodnie z zaleceniem z sekcji 4, pin 27) — potwierdzić
   elektrycznie, nie tylko z układu graficznego.
8. **Wartości L1/L2/L3** (DCDC3/DCDC2/ładowarka) nieczytelne na zrzucie — potwierdzić
   odpowiednio 2.2µH / 4.7µH / 4.7µH (sekcja 4, piny 15/8/45).

Żadna z powyższych pozycji nie wymaga zmiany architektury — to lista konkretnych połączeń do
domknięcia, nie przeprojektowania.

---

## 10. Realny stan `pcb/cpu_pwr.kicad_sch` (2026-07-25) — po dokładnym śledzeniu przewodów

W odróżnieniu od sekcji 9 (przegląd zrzutu ekranu), to jest wynik **dokładnego prześledzenia
realnych `(wire ...)`/`(junction ...)` w faktycznym pliku** `pcb/cpu_pwr.kicad_sch` (symbol
`Allwinner:AXP209`), nie odczytu wizualnego. Zastępuje ustalenia sekcji 9 tam, gdzie się różnią.

**Potwierdzone jako poprawne (wire-trace):**
- DC3SET(29) → APS(21) — zgodnie z zaleceniem.
- LDO1SET(27) → VINT(26), wspólny węzeł z kondensatorem bypass C82=1µF — zgodnie z zaleceniem.
- VBUS(31) → jawne NC — zgodnie z decyzją (sekcja 1.3 dok. nadrzędnego).
- PWROK(25) → węzeł również eksportowany jako `global_label "reset"`, przez R12=47k → sieć "VDD"
  — realizacja opcji 2 z sekcji 7 (PWROK jako źródło resetu); pamiętać o zastrzeżeniu stamtąd.
- Masa: AGND(22)/PGND1-3(46,9,16)/EP(GND$1-9) wszystkie w jednej wspólnej sieci GND (51 członków
  razem z kondensatorami) — nie gwiazda, jedna płaszczyzna.
- Dławiki: L8 (DCDC2/LX2) = 4.7µH ✓, L10 (ładowarka/LX1) = 4.7µH ✓.
- N_VBUSEN(6) → sieć **IPSOUT** (nie GND) — technicznie spełnia wymóg "nie GND/nie wybieraj VBUS",
  choć inaczej niż sugerowane w sekcji 4 (VINT/3.3V); to częsty, akceptowalny wzorzec w
  referencyjnych projektach AXP209 — do świadomego zaakceptowania, nie do zmiany na siłę.

**⚠️ Realne braki — piny fizycznie niepodłączone (część nawet bez flagi no-connect):**

1. **SDA(1) i SCK(2) — zero przewodów.** Magistrala I2C w ogóle nie wychodzi z tego arkusza: brak
   pull-upów 2.2kΩ i brak jakiegokolwiek połączenia dalej (np. do TWI1 V3S). To odcina cały PMIC od
   hosta programowo.
2. **IRQ/WAKEUP(48) — zero przewodów.** Brak pull-upu 51kΩ.
3. **GPIO0(19) — zero przewodów, nawet bez NC** (gorzej niż zwykłe "nieużywane"). Dzielnik ADC do
   pomiaru napięcia wejściowego 8.5-60V (sekcja 6) nie istnieje w tym pliku.
4. **BACKUP(30) — zero przewodów.** Brak złącza LIR2032 i brak kondensatora — obwód podtrzymania
   RTC (sekcja 1.2 dok. nadrzędnego) nieobecny.
5. **PWRON(47) i N_OE(4) — zero przewodów, bez NC.** PWRON jest krytyczne: bez podłączenia do
   przycisku (choćby przez wewnętrzny pull-up 100k do APS + zewnętrzny przycisk do GND) nie da się
   włączyć układu. CHGLED(36) też pływające, ale to tylko opcjonalny wskaźnik LED, niższy priorytet.

**Do potwierdzenia (nie błąd, wymaga decyzji):**

- **L9 (DCDC3/LX3) = 4.7µH**, a sekcja 4 zalecała ~2.2µH (próg >2.5V wg datasheet) — do
  zweryfikowania, czy 4.7µH tu też działa poprawnie, czy warto zmienić na 2.2µH.
- **R41=10kΩ między TS(37) a GND** zamiast bezpośredniego zwarcia — sekcja 4/8 zalecały **dosłowne
  zwarcie TS do GND** (0Ω) dla wyłączenia detekcji NTC (pack ma wbudowane PCM). Rezystor 10k może
  zostawić niezerowe napięcie z wewnętrznego źródła prądowego TS i nie wyłączyć wykrywania tak, jak
  zakładano — sprawdzić w datasheet, czy próg tolerancji na to pozwala, albo zamienić na zwarcie.
- **D6 (1N5819WS)** w układzie bocznikującym do masy (K→BATT, A→GND), nie szeregowym — potwierdzić,
  że to celowa ochrona przeciwprzepięciowa/odwrotnej polaryzacji na złączu baterii, nie pomyłka.
- **ACIN@1/ACIN@2 mają tylko C17=10µF**, bez widocznego dalszego połączenia — upewnić się, że
  łączy się to międzyarkuszowo z wyjściem TPS54360 (sekcja 3.1), a nie że ścieżka ACIN jest
  faktycznie martwa.
- **Nazewnictwo sieci "+3V3" vs "+3.3V"** — dwa różne symbole/nazwy dla DCDC3 (+3V3) i LDO4
  (+3.3V); prawdopodobnie zamierzone jako dwie osobne szyny 3.3V, ale warto ujednolicić nazwy dla
  czytelności i uniknięcia pomyłki przy podłączaniu czegoś do "złej" szyny 3.3V w przyszłości.
- Kondensatory 1nF (C9 na "+3V3", C10 na "+1V25") obok kondensatorów 10µF na tych samych szynach —
  prawdopodobnie kondensatory HF, ale warto sprawdzić, czy to nie miały być 1µF (literówka).
