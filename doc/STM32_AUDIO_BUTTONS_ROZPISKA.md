# STM32L432KCU6 — kontroler audio/przycisków headsetu, pełna rozpiska pinów

Dokument uzupełniający do [`TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md`](TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md)
(sekcja 11) i [`V3S_PINOUT_ROZPISKA.md`](V3S_PINOUT_ROZPISKA.md) (piny 45/113-125) — pełna
rozpiska 32 pinów **STM32L432KCU6** (UFQFPN32/QFN-32), dodanego 2026-07-25 jako koprocesor I2C
obsługujący interfejs headsetu (przyciski/audio) i dodatkowe GPIO poza zakresem LRADC, w związku
z tym, że ta sama płyta ma posłużyć też do budowy innego urządzenia wymagającego obsługi dźwięku
(słuchawki + mikrofon).

Stan na 2026-07-25, **zaktualizowano 2026-07-29 wg faktycznego schematu** (`stm.kicad_sch` +
`audio.kicad_sch` + `ext_conn.kicad_sch`, zweryfikowane `kicad-happy:kicad`). Koncepcja
przesunęła się od czasu pierwszej wersji tego dokumentu — sekcje 3-4 poniżej opisują teraz **to,
co jest faktycznie narysowane**, nie pierwotny plan. Kluczowe różnice względem wersji z
2026-07-25:

- **Nie ma osobnych linii `BTN_PTT`/`BTN_VOL_UP`/`BTN_VOL_DOWN`.** Zamiast trzech dyskretnych
  przycisków, headset dostaje **dwie linie `DATA+`/`DATA-` w trybie dwufunkcyjnym**: albo GPIO
  (kodowanie 2-bitowe: spoczynek `00`, przyciski `01`/`10`/`11`, część zwiera do masy, część do
  plusa), albo pełny UART, jeśli podłączony akcesorium jest "mądrzejsze" (ma własny mikrokontroler
  po drugiej stronie). Wybór trybu jest w całości po stronie firmware STM32 (konfiguracja
  kierunku/pull na GPIO vs peryferium USART2) — płytka nie narzuca żadnego z góry.
- **Nie ma `JACK_DETECT`.** Ta koncepcja (mechaniczny styk wykrycia wtyku) została porzucona.
  Zamiast tego złącze `J14` ma dodatkowy pin zasilający (`+3.3V` filtrowane przez ferryt+TVS) —
  do zasilenia własnej elektroniki "mądrego" headsetu (spójne z opcją UART wyżej).
- **10 pinów (`PA0`, `PA4-PA9`, `PB0/PB1/PB4`) to teraz ogólny breakout GPIO** wyprowadzony na
  złącze `J15` (JST 12-pin) — do obsługi np. zewnętrznej klawiatury panelowej, ale też
  wykorzystywalny jako UART lub SPI, w zależności od potrzeb finalnego urządzenia. To nie jest
  już "rezerwa" w sensie niewykorzystanych pinów — są aktywnie wyprowadzone na złącze, tylko ich
  funkcja nie jest zdefiniowana na poziomie schematu.
- **Magistrala I2C1 wylądowała na `PB6`/`PB7`** (współdzielona z AXP209+TSC2007), nie na
  `PA9`/`PA10` jak pierwotnie zdecydowano — `PA9`/`PA10` są teraz częścią breakoutu GPIO /
  sterowania wzmacniaczem.
- **Dodano wzmacniacz słuchawkowy `U11` (TI TPA6132A2RTE)** między V3S `HPOUTR`/`HPOUTL` a
  złączem `J14` — sterowany z STM32 (`PA10`=`EN`, `PA11`=`G0`, `PA12`=`G1`, wybór wzmocnienia w
  runtime). To nie było przewidziane w pierwotnej wersji tego dokumentu.
- **`STM32_IRQ` jest na `PB5`, nie `PA8`**, i idzie do V3S **`PB7`**, nie `PB6` (`PB6` jest już
  zajęte przez `PWROK`, patrz `TODO.md`) — `PA8` należy teraz do breakoutu GPIO (`STM.IO8`).

## Źródła i metoda

- **Pełny pinout, typ pinów, funkcje alternatywne, kanały ADC, wymagania zasilania/decoupling,
  zachowanie NRST/BOOT0** — z oficjalnego datasheetu ST **DS11451 Rev 4** ("STM32L432KB/KC —
  Ultra-low-power Arm Cortex-M4 32-bit MCU+FPU"), Figura 5 (pinout UFQFPN32), Tabela 14 (pin
  definitions), Tabela 15 (alternate functions), Figura 9 (power supply scheme), §6.3.15
  (NRST), §3.7 (boot modes) — zweryfikowane bezpośrednio w PDF, nie z pamięci.
- **Przydział konkretnych pinów w tym projekcie** (który GPIO robi co) — decyzja podjęta tu, na
  bazie wolnych/zalecanych pinów z datasheetu (np. I2C1 na PA9/PA10 jako "primary", nie PB6/PB7
  jako "alt.").
- Jak w poprzednich dokumentach: pozycje niepewne oznaczone **(do potwierdzenia)**.

---

## 1. Obudowa i zasilanie

**UFQFPN32** (QFN-32-podobny, 5×5mm, rozstaw 0.5mm). **Tylko 1 domena I/O** (VDDIO1=VDD, brak
VDDIO2 — ten pakiet jest za mały na drugą domenę) — upraszcza zasilanie, wszystkie GPIO na tym
samym poziomie logicznym 3.3V co reszta płyty.

**Zasilanie**: VDD 1.71-3.6V — zasilić z **VCC-IO** (domena 3.3V V3S/AXP209 DCDC3, sekcja 11.2
dok. nadrzędnego), bez dodatkowej LDO. VDDA/VREF+ (pin 5, połączone w jeden pin w tym pakiecie)
— zewrzeć z VDD przez mały ferryt/RC, bo jedyna funkcja analogowa użyta tu to ADC1 (żadnego
DAC/OPAMP), co zgodnie z datasheet kwalifikuje się do prostego zasilenia z VDD.

**Dekupling (wprost z Figury 9 datasheetu)**:
- VDD/VSS (piny 1,17 / 16,32): **2× 100nF** (po jednym przy każdym pinie VDD) + **1× 4.7µF**
  bulk (jeden na cały chip).
- VDDA/VREF+ (pin 5): **10nF + 100nF + 1µF** (role VDDA-decoupling i VREF-filtering łączą się
  na tym samym fizycznym pinie w tym pakiecie).
- Razem: **~6 kondensatorów**, umieszczone jak najbliżej odpowiednich pinów (zalecenie
  datasheetu: najlepiej od spodu płytki, bezpośrednio pod pinem).

**Zegar: brak zewnętrznego kwarca** — ten pakiet **nie ma w ogóle dedykowanych pinów HSE
OSC_IN/OSC_OUT** (HSE możliwe tylko przez zewnętrzny sygnał na PA0/CK_IN, bypass, nie kwarc).
Wewnętrzny **HSI16** (16MHz RC) w pełni wystarcza dla I2C slave + ADC + debounce (żadna z tych
funkcji nie wymaga precyzji krystalicznej) — **zero-crystal design**, mniejszy BOM. PC14/PC15
(OSC32_IN/OUT, do opcjonalnego kwarca RTC 32.768kHz) **nie są potrzebne** — V3S ma już własny
RTC (`TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md`, pkt. 1.2) — zostają jako zwykłe GPIO w rezerwie
(z zastrzeżeniem: zasilane przez ograniczony prądowo przełącznik wewnętrzny, max 3mA, zalecana
prędkość wyjścia ≤2MHz/30pF — nie używać do sterowania czymś prądożernym).

---

## 2. NRST i BOOT0

- **NRST (pin 4)**: STM32 ma **wewnętrzny stały pull-up** (typ. 40kΩ) — w odróżnieniu od pinu
  RESET V3S (brak pull-up, wymaga zewnętrznego, patrz `V3S_PINOUT_ROZPISKA.md` pkt. 99),
  **STM32 nie wymaga zewnętrznego rezystora**. Zalecany (opcjonalny, dla odporności na zakłócenia)
  kondensator **100nF do GND** blisko pinu — warto dodać, biorąc pod uwagę sąsiedztwo z
  przełączanymi obwodami (CAN, DC/DC) na tej samej płycie.
- **BOOT0 (pin 31, współdzielony z PH3)**: zalecenie — **pull-down 10kΩ do GND** (gwarantuje
  boot z Flash przy domyślnych option bytes, `nSWBOOT0=1` fabrycznie) + **opcjonalna zworka/
  punkt testowy do VDD** równolegle, żeby serwisant mógł wymusić wejście w bootloader systemowy
  (UART/I2C/SPI/USB DFU) do odzyskiwania firmware w polu, bez utraty tej ścieżki na stałe.

---

## 3. Pełna tabela pinów (32) — wg faktycznego schematu (2026-07-29)

Legenda: **I2C1** = magistrala do V3S (TWI1, współdzielona z AXP209+TSC2007+STM32), **STM.IOn**
= ogólny GPIO wyprowadzony na złącze `J15` (JST 12-pin, `ext_conn.kicad_sch`), **audio** = pin
związany z interfejsem headsetu na `J14` (JST 8-pin, `audio.kicad_sch`) lub sterowaniem
wzmacniacza `U11` (TPA6132A2).

| Pin | Nazwa | Funkcje alt. (wybrane) | Sieć w schemacie | Przydział w tym projekcie |
|---|---|---|---|---|
| 1 | VDD | — | `+3.3V` | zasilanie 3.3V |
| 2 | PC14-OSC32_IN | EVENTOUT | *(no-connect)* | jawnie oznaczone no-connect — wolne (brak kwarca LSE — V3S ma własny RTC) |
| 3 | PC15-OSC32_OUT | EVENTOUT | *(no-connect)* | jawnie oznaczone no-connect — wolne |
| 4 | NRST | reset | `nRESET` | reset (wewn. pull-up; `C121` 100nF do GND obecny; też wyprowadzony na `J11` TC2030 SWD) |
| 5 | VDDA/VREF+ | analog supply | dedykowana sieć | zasilanie analogowe z `+3.3V` przez ferryt `FB3` (100R@100MHz) + `C119`(100n)/`C120`(1µF) |
| 6 | PA0/CK_IN | TIM2_CH1, USART2_CTS | `STM.IO3` | **breakout GPIO → J15** (klawiatura zewn. / UART / SPI, do ustalenia w firmware) |
| 7 | PA1 | TIM2_CH2, I2C1_SMBA | `AUDIO_ADC` | analogowy odczyt węzła mic-bias/drabinki na `J14.pin4` (przez `R48` 47k) |
| 8 | PA2 | TIM2_CH3, USART2_TX | `DATA+` | **headset, tryb dwufunkcyjny** — GPIO (bit kodowania przycisków) lub USART2_TX; przez `R49` (100Ω) do `J14.pin6` |
| 9 | PA3 | TIM2_CH4, USART2_RX | `DATA-/GPIO1` | **headset, tryb dwufunkcyjny** — GPIO (bit kodowania przycisków) lub USART2_RX; przez `R50` (100Ω) do `J14.pin7` |
| 10 | PA4 | SPI1_NSS, USART2_CK | `STM.IO4` | **breakout GPIO → J15** |
| 11 | PA5 | TIM2_CH1/ETR, SPI1_SCK | `STM.IO5` | **breakout GPIO → J15** |
| 12 | PA6 | TIM1_BKIN, SPI1_MISO | `STM.IO6` | **breakout GPIO → J15** (JACK_DETECT porzucone — patrz uwaga wyżej) |
| 13 | PA7 | I2C3_SCL, SPI1_MOSI | `STM.IO7` | **breakout GPIO → J15** |
| 14 | PB0 | SPI1_NSS, QUADSPI_BK1_IO1 | `STM.IO0` | **breakout GPIO → J15** |
| 15 | PB1 | USART3_RTS_DE | `STM.IO1` | **breakout GPIO → J15** |
| 16 | VSS | — | `GND` | masa |
| 17 | VDD | — | `+3.3V` | zasilanie 3.3V |
| 18 | PA8 | MCO, TIM1_CH1 | `STM.IO8` | **breakout GPIO → J15** (IRQ do V3S przeniesiony na PB5, patrz niżej) |
| 19 | PA9 | TIM1_CH2, I2C1_SCL (alt.) | `STM.IO9` | **breakout GPIO → J15** (I2C1 wylądowało na PB6/PB7, nie tutaj) |
| 20 | PA10 | TIM1_CH3, I2C1_SDA (alt.) | `EN` | **wzmacniacz `U11` TPA6132A2, pin 13 (EN)** — włącz/wyłącz |
| 21 | PA11 | CAN1_RX, USB_DM | `GPIO_CTRL1` | **wzmacniacz `U11`, pin 6 (G0)** — wybór wzmocnienia, bit 0 |
| 22 | PA12 | CAN1_TX, USB_DP | `G1` | **wzmacniacz `U11`, pin 7 (G1)** — wybór wzmocnienia, bit 1 |
| 23 | PA13 (SWDIO) | SWDIO | `SWDIO` | header programowania/debug (`J11` TC2030) |
| 24 | PA14 (SWCLK) | SWCLK | `SWCLK` | header programowania/debug (`J11` TC2030) |
| 25 | PA15 (JTDI) | JTDI, TIM2_CH1/ETR | *(single-pin, wolny)* | rezerwa GPIO — jedyny pin faktycznie jeszcze niewykorzystany |
| 26 | PB3 (JTDO) | JTDO-TRACESWO, TIM2_CH2 | `SWO` | **trace/SWO** — wyprowadzone na `J11`, nie jest już rezerwą |
| 27 | PB4 (NJTRST) | NJTRST, I2C3_SDA | `STM.IO2` | **breakout GPIO → J15** |
| 28 | PB5 | LPTIM1_IN1, I2C1_SMBA | `IRQ_V3S` | **STM32→V3S IRQ**, idzie do V3S **`U2.pin46` / PB7** (nie PA8/PB6 jak w pierwotnym planie — PB6 zajęte przez `PWROK`) |
| 29 | PB6 | LPTIM1_ETR, I2C1_SCL (alt.) | `APX_SCL` | **I2C1_SCL** → magistrala TWI1 współdzielona z AXP209+TSC2007 (pull-up `R11` 2.2kΩ do `+3V3`, już istniejący) |
| 30 | PB7 | LPTIM1_IN2, I2C1_SDA (alt.) | `APX_SDA` | **I2C1_SDA** → jw. (pull-up `R10` 2.2kΩ) |
| 31 | PH3/BOOT0 | BOOT0 | dedykowana sieć | **pull-down `R43` 10kΩ do GND obecny** — zgodnie z zaleceniem; brak opcjonalnej zworki/testpointu do VDD (nice-to-have, nieobecne) |
| 32 | VSS | — | `GND` | masa |
| 33 | EP/VSS | — | `GND` | pad termiczny/masa (numerowany jako osobny pin w symbolu KiCad) |

**Budżet rezerwy faktyczny**: tylko **`PA15`** jest naprawdę wolny. `PC14`/`PC15` są jawnie
no-connect (celowo, nie "do wykorzystania" bez przeróbki). Wszystkie pozostałe piny opisane
wcześniej jako "rezerwa" (`PA0`, `PA4-PA9`, `PB0/PB1/PB4`) są teraz aktywnie wyprowadzone jako
`STM.IO0`-`STM.IO9` na złącze `J15` — nie są to już piny "na przyszłość" w schemacie, tylko
realny, uzbrojony breakout czekający na decyzję firmware/akcesorium.

---

## 4. Obsługa headsetu — stan wg schematu (2026-07-29)

Fizyczny interfejs headsetu na `J14` (JST PH 8-pin) ma dziś **6 linii sygnałowych + 2×GND**:
`OUTR`/`OUTL` (wyjście wzmacniacza `U11`), mic/ADC (pin 4, przez `R48` 47k do `PA1`), `DATA+`/
`DATA-` (piny 6/7, przez `R49`/`R50` 100Ω do `PA2`/`PA3`), i pin zasilający (pin 8, `+3.3V`
filtrowane `FB6`+TVS `D18` — zasila ewentualną własną elektronikę "mądrego" headsetu, nie jest
to wejście detekcji). Wszystkie 6 linii + zasilanie mają TVS (`STN061050BL40`) na złączu.

### 4.1 Kanał analogowy — `PA1`/`AUDIO_ADC`

Czyta napięcie DC na węźle mic-bias (`J14.pin4`), zasilanym z V3S `HBIAS` (pin119) przez `R53`
(2k) — klasyczny obwód bias dla wkładki elektretowej, z `C136` (470n) jako bypass i `R48` (47k)
jako rezystor szeregowy/antyaliasingowy do wejścia ADC STM32. Może posłużyć do drabinki
rezystorowej (rozróżnianie poziomów napięcia) albo po prostu do monitorowania obecności/poziomu
sygnału na linii MIC — dokładna rola zależy od konkretnego akcesorium i jest do ustalenia w
firmware (progi nieznane z góry, brak jednego standardu CTIA/OMTP).

### 4.2 Kanał cyfrowy dwufunkcyjny — `PA2`/`PA3` (`DATA+`/`DATA-`)

Te dwie linie (przez `R49`/`R50` 100Ω + TVS do `J14.pin6`/`pin7`) obsługują **dwa niezależne
tryby pracy, wybierane w firmware**, nie na sztywno w sprzęcie:

- **Tryb GPIO (headset z przyciskami)**: `PA2`/`PA3` jako cyfrowe wejścia z **wewnętrznym
  pull-STM32** (kierunek pull do ustalenia w firmware — patrz uwaga o pull-up/down niżej).
  Kodowanie 2-bitowe: stan spoczynkowy `00`, przyciski dają `01`/`10`/`11` — część przycisków w
  headsecie zwiera linię do masy, część do plusa (`+3.3V`), w zależności od konkretnego
  akcesorium/konwencji producenta.
- **Tryb UART (headset "mądry")**: `PA2`=`USART2_TX`, `PA3`=`USART2_RX` — dla akcesorium z
  własnym mikrokontrolerem po drugiej stronie kabla. Pin zasilający na `J14.pin8` (patrz wyżej)
  istnieje właśnie po to, żeby zasilić taką elektronikę.
- Płytka **nie rozstrzyga, który tryb jest aktywny** — `R49`/`R50` (100Ω) to tylko
  zabezpieczenie prądowe/ESD-koordynacja z TVS, kompatybilne z obydwoma trybami. Wybór trybu i
  ewentualna autodetekcja to w całości zadanie firmware STM32.

**Uwaga o pull-up/pull-down (do zamknięcia w firmware, nie w sprzęcie):** na płytce **nie ma
rezystorów podciągających** na `DATA+`/`DATA-` — świadomie, bo stały pull-up/pull-down
przeszkadzałby w trybie UART. To oznacza, że **firmware musi jawnie skonfigurować wewnętrzny
pull STM32** (GPIO pull-up lub pull-down, zależnie od konwencji wybranego headsetu) zanim
zacznie odczytywać stan przycisków w trybie GPIO — inaczej piny są elektrycznie pływające
(floating) między resetem a inicjalizacją firmware, co przy dłuższym, nieekranowanym kablu do
headsetu może dawać losowe odczyty/fałszywe zdarzenia w tym krótkim oknie. Do rozważenia:
krótkie opóźnienie/maskowanie zdarzeń przycisków tuż po starcie, dopóki pull nie zostanie
skonfigurowany.

### 4.3 Sterowanie wzmacniaczem — `PA10`/`PA11`/`PA12`

`U11` (TPA6132A2) ma trzy cyfrowe wejścia sterujące z STM32: `EN` (`PA10`, pin13 — włącz/wyłącz
wzmacniacz), `G0`/`G1` (`PA11`/`PA12`, piny 6/7 — wybór wzmocnienia). Są to zwykłe wyjścia
push-pull STM32 — nie ma na nich rezystora podciągającego na płytce, co jest poprawne dla
aktywnie sterowanego wyjścia w normalnej pracy. **Jedyne ryzyko**: domyślny stan GPIO STM32 po
resecie to wejście bez pull (floating) — do czasu inicjalizacji firmware `EN` może być w stanie
nieokreślonym, co teoretycznie mogłoby dać krótki trzask/pop na starcie. Nieobowiązkowe, ale
tanie zabezpieczenie: zewnętrzny pull-down ~100kΩ na `EN` gwarantujący wzmacniacz wyłączony
(wyciszony) do czasu, aż firmware go świadomie włączy — do rozważenia przy kolejnej rewizji
płytki, nie blokuje obecnego etapu.

### 4.4 Breakout GPIO — `J15` / `STM.IO0`-`STM.IO9`

10 pinów (`PA0`, `PA4-PA9`, `PB0/PB1/PB4`) jest wyprowadzonych na złącze `J15` (JST 12-pin, w
`ext_conn.kicad_sch`, razem z filtrowanym zasilaniem `+3.3V` na pinie 1 i wspólną masą na pinie
12). Przeznaczenie: **ogólny interfejs do zewnętrznego modułu** — może to być klawiatura
panelowa (matryca), ale piny nadają się też pod UART lub SPI, w zależności od finalnej
architektury urządzenia. Żadna konkretna funkcja nie jest tu jeszcze przypisana na poziomie
schematu — to świadomie otwarty punkt, nie brak. Ewentualne rezystory podciągające (np. do
skanowania matrycy klawiszy) są sprawą modułu zewnętrznego, nie tej płytki.

### 4.5 Status rejestru I2C (koncepcja, do potwierdzenia przy pisaniu firmware)

Poprzednia wersja tego dokumentu proponowała rejestr `STATUS` z bitami PTT/Vol+/Vol− na sztywno
przypisanymi do konkretnych przycisków. Skoro `DATA+`/`DATA-` kodują teraz stan 2-bitowo (a nie
3 oddzielne, nazwane przyciski), mapa rejestrów wymaga przeprojektowania pod nowe kodowanie —
**do zrobienia przy starcie prac nad firmware STM32**, nie rozstrzygnięte w tym dokumencie.
Ogólny szkielet (adres I2C dowolny wolny 7-bit, np. `0x20`, niekolidujący z AXP209 `0x34` ani
TSC2007 `0x48/0x49`) pozostaje aktualny.

IRQ do V3S jest teraz na **`PB5`** (nie `PA8`), i dochodzi do V3S **`PB7`** (`U2.pin46`, nie
`PB6` — `PB6` zajęte przez `PWROK`).

---

## 5. Strona V3S/Linux

**Bez własnego kernel drivera** — zgodnie z już przyjętym wzorcem tego projektu (PWRON/dotyk
obsługiwane w userspace, nie przez dedykowany driver kernela):
- Odczyt rejestrów przez `/dev/i2c-X` (standardowy i2c-dev) z poziomu aplikacji/małego demona.
- Przerwanie (V3S **PB7**, nie PB6 — patrz sekcja 3/4) budzi proces przez linię GPIO
  skonfigurowaną jako edge-triggered w `/dev/gpiochipN` (`libgpiod`), zamiast pollingu I2C w
  pętli.
- Zdarzenia (kody przycisków zdekodowane z `DATA+`/`DATA-`, AUX z `J15`) mapowane na kody
  klawiszy (`KEY_*`/`BTN_*`) analogicznie do już istniejącej obsługi evdev dotyku/PWRON w
  `CanSensorHub-v3s`.

---

## 6. Otwarte pytania / do potwierdzenia

- **Kierunek pull na `DATA+`/`DATA-` w trybie GPIO** (pull-up czy pull-down, zależnie od
  konwencji konkretnego headseta) — nie ma na to odpowiedzi na poziomie sprzętu, płytka
  celowo nie narzuca kierunku (patrz sekcja 4.2); firmware musi to jawnie skonfigurować.
- **Tryb pracy `DATA+`/`DATA-` (GPIO vs UART) i ewentualna autodetekcja** — do ustalenia przy
  wyborze konkretnego akcesorium/BOM, podobnie jak poprzednio "wariant A/B".
- **Zewnętrzny pull-down na `EN` wzmacniacza `U11`** (sekcja 4.3) — nice-to-have przeciw
  potencjalnemu trzaskowi na starcie, nieobecny na obecnej rewizji, niski priorytet.
- **Dokładne progi ADC na `PA1`** (jeśli używane do rozróżniania poziomów) — zależą od
  konkretnego modelu headsetu, nie z góry znane.
- **Funkcja breakoutu `J15`** (klawiatura / UART / SPI) — świadomie otwarta, do decyzji przy
  projektowaniu konkretnego modułu zewnętrznego (sekcja 4.4).
- **Adres I2C STM32** — dowolny wolny, do ustalenia przy pisaniu firmware.
- **BOOT0 default (`nSWBOOT0`)** — założono fabryczny domyślny stan (`nSWBOOT0=1`, pin
  kontroluje boot) zgodnie ze standardową konwencją ST (np. płytki Nucleo) — nie zweryfikowane
  wprost w tym datasheet (szczegóły w RM0394, nie DS11451), oznaczone tam jako "likely", nie
  "confirmed" — potwierdzić przed poleganiem na tym przy BOM.
- **VBAT** — ten pakiet prawdopodobnie nie ma osobnego pinu VBAT (zbindowany wewnętrznie do
  VDD) — nieistotne funkcjonalnie tutaj (RTC STM32 nieużywany, V3S ma własny), ale odnotowane
  jako "likely", nie "confirmed" w źródle.
- **Firmware framework** — HAL/LL (STM32Cube) czy Zephyr — nie zdecydowane, osobna decyzja przy
  starcie prac nad firmware STM32.
