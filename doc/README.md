# Dokumentacja Hardware ControlPanelV3S — co gdzie jest

Indeks katalogu `doc/`. Firmware ma swój analogiczny indeks w repo `CanSensorHub-Firmware`
(`firmware/doc/README.md`).

## Trwałe decyzje projektowe

Te dokumenty rosną i się nie starzeją — nowa praca dopisuje kolejne sekcje, a nie zastępuje starych.

| Dokument | Zakres |
|---|---|
| [TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md](TERMINAL_RECZNY_DECYZJE_PROJEKTOWE.md) | terminal ręczny V3S: hardware i firmware, decyzje i ich uzasadnienie. Kopia tego samego pliku jest też w repo firmware (`firmware/doc/`) — to tutaj jest źródło prawdy dla części hardware. |
| [DESIGN_REVIEW.md](DESIGN_REVIEW.md) | przegląd projektowy płyty |

## Rozpiski sprzętowe

Materiał referencyjny, zmieniany tylko przy zmianie sprzętu.

- [V3S_PINOUT_ROZPISKA.md](V3S_PINOUT_ROZPISKA.md) — 128 pinów LQFP128 i gałęzie zasilania
- [AXP209_PINOUT_ROZPISKA.md](AXP209_PINOUT_ROZPISKA.md) — PMIC: piny, drzewo zasilania, połączenia
- [STM32_AUDIO_BUTTONS_ROZPISKA.md](STM32_AUDIO_BUTTONS_ROZPISKA.md) — STM32L432KCU6, kontroler audio i przycisków headsetu

## Referencje

- [V3S_CDR_STD_V1_0_20150514.pdf](V3S_CDR_STD_V1_0_20150514.pdf) — datasheet Allwinner V3S
- [lichee_zero.pdf](lichee_zero.pdf) — dokumentacja LicheePi Zero

FunKey S jest przywoływany w kilku dokumentach (m.in. `V3S_PINOUT_ROZPISKA.md`,
`AXP209_PINOUT_ROZPISKA.md`) jako punkt odniesienia — ten sam SoC/PMIC co w tym projekcie — na
podstawie publicznie dostępnej dokumentacji projektu FunKey, bez kopiowania jego plików źródłowych
ani bibliotek do tego repo.

## TODO.md (poza `doc/`)

`../TODO.md` — otwarte punkty z ostatniego przeglądu projektowego (blokery MPN, DRC itd.), datowane
i nietrwałe, nie jest źródłem prawdy o decyzjach.
