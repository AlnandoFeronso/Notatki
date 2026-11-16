---
tags: [p-komputer, mikrokontroler, firmware, stm32]
warstwa: 2
---
# 03 Mikrokontroler

Wróć: [[00 Przegląd projektu]]

> [!abstract] Rola
> "Kierownik" sprzętu: rozmawia z PC, ładuje wagi do FPGA, steruje przebiegiem obliczeń (start/stop, temperatura β w czasie, powtórzenia), zbiera wyniki i odsyła je do PC.

---

## Zadania

| Zadanie | Szczegóły |
| --- | --- |
| **Komunikacja z PC** | USB (CDC / bulk) albo Ethernet. Ramki z sumą kontrolną (CRC) i potwierdzeniami |
| **Zarządzanie wagami** | odbiór problemu, zapis do biblioteki (zewnętrzny RAM / flash / karta SD), załadowanie do FPGA |
| **Sterowanie obliczeniami** | harmonogram β(t), liczba przebiegów, wielokrotne restarty z różnych stanów początkowych, limit czasu |
| **Odczyt wyników** | najlepszy znaleziony stan + jego energia, statystyki ⟨mᵢ⟩ / ⟨mᵢmⱼ⟩ policzone przez FPGA |
| **Losowość** | (propozycja) odbiór z TRNG, testy zdrowia, rozsyłanie ziaren do FPGA, patrz [[05 TRNG - generator szumu]] |
| **Diagnostyka** | temperatura, status FPGA, logi |

> [!danger] Czego MCU **nie** robi
> **Nie przekazuje każdej próbki p-bitów.** FPGA generuje setki milionów aktualizacji na sekundę, a żadne łącze MCU tego nie przeniesie.
> FPGA samo liczy statystyki i pamięta najlepszy stan, a MCU czyta tylko te podsumowania (kilobajty, nie gigabajty).

---

## Bilans przepustowości (czy łącza wystarczą?)

| Dane | Rozmiar (przykład) | Kiedy |
| --- | --- | --- |
| Macierz gęsta N=256, wagi 8-bit | 64 KB | raz na problem |
| Graf rzadki N=1024, 16 sąsiadów (waga + indeks) | ~48 KB | raz na problem |
| Harmonogram β | ~100 B – 1 KB | raz na przebieg |
| Wynik: najlepszy stan + energia | ~N/8 B + 4 B | na koniec przebiegu |
| Statystyki ⟨mᵢ⟩ (32 bity na p-bit) | 1–4 KB | co jakiś czas |

**Wniosek:** nawet USB Full Speed (12 Mbit/s, realnie ~1 MB/s) i SPI wystarczą. Załadowanie problemu zajmie ułamek sekundy. Komunikacja **nie jest** wąskim gardłem, **jeśli** FPGA liczy statystyki u siebie.

---

## Połączenie MCU ↔ FPGA

| Interfejs | Przepustowość (rząd wielkości) | Trudność | Kiedy |
| --- | --- | --- | --- |
| **SPI** | kilka MB/s | ⭐ łatwy | **prototyp** |
| **FMC** (równoległa magistrala pamięci w STM32 F4/F7/H7) | dziesiątki MB/s | ⭐⭐ | wersja docelowa: FPGA widziane jak pamięć SRAM pod adresem |
| QSPI / OctoSPI | dziesiątki MB/s | ⭐⭐⭐ | alternatywa dla FMC |

> [!warning] SPI a zegar FPGA (praktyczna pułapka)
> Najprostszy SPI slave w FPGA próbkuje sygnały SPI zegarem 50 MHz przez 2 rejestry synchronizujące (technika z [[Podstawy składni Verilog]], ćwiczenie 6). Wtedy **zegar SPI musi być kilka razy wolniejszy niż zegar FPGA**, w praktyce do ~10 MHz.
> Szybciej: podnieść zegar FPGA (PLL) albo zbudować SPI taktowany samym SCK, z poprawnym przejściem między domenami zegara.

**Mapa rejestrów** (co pod jakim adresem w FPGA) to wspólny kontrakt zespołu MCU i FPGA, patrz [[09 Zespół, interfejsy i plan#Kontrakt B - mapa rejestrów FPGA]].

---

## Który mikrokontroler?

| Rodzina | Za | Przeciw |
| --- | --- | --- |
| **STM32F4** (np. F407/F429) | tani, ogromnie dużo przykładów, ma FMC, Ethernet (niektóre), sprzętowy RNG | USB High Speed wymaga zewnętrznego PHY |
| **STM32H7** | bardzo szybki (do kilkuset MHz), FMC, duży RAM, sprzętowe AES/HASH (niektóre modele) | bardziej złożony, droższy |
| Płytki **Nucleo / Discovery** | gotowe do pracy od razu, programator na pokładzie | — |

**Rekomendacja na prototyp:** płytka Nucleo z STM32F4 albo H7 z wyprowadzonym SPI (i najlepiej FMC na później). Konkretny model wybrać po ustaleniu, czy chcecie Ethernet / USB HS. Sprawdzić w karcie katalogowej, bo możliwości różnią się w obrębie rodziny.

> [!tip] Wbudowany RNG w STM32
> Wiele STM32 ma **sprzętowy generator liczb losowych** (peryferium RNG). To świetny **punkt odniesienia** dla własnego TRNG na diodzie Zenera ([[05 TRNG - generator szumu]]), a do czasu jego zbudowania także źródło ziaren.

### Język firmware
- **C + STM32Cube HAL**: najwięcej dokumentacji i przykładów, najłatwiej znaleźć pomoc.
- **Rust (embassy / stm32-rs)**: bezpieczniejszy, nowoczesny. Ma sens, jeśli osoba od firmware zna Rusta.

---

## Czy MCU w ogóle jest potrzebny? (analiza alternatyw)

| Wariant | Opis | Za | Przeciw |
| --- | --- | --- | --- |
| **A. PC → MCU → FPGA** (obecny plan) | jak w przeglądzie | jasny podział pracy w zespole; MCU robi sterowanie w czasie rzeczywistym, testy TRNG, kryptografię | dodatkowa płytka, firmware i protokół; jeden "przeskok" więcej |
| B. PC → mostek USB (np. FTDI FT232H) → FPGA | PC rozmawia z FPGA prawie bezpośrednio | prostsze, szybkie łącze | całe sterowanie po stronie PC (bez czasu rzeczywistego) albo w FPGA |
| C. **FPGA z procesorem w środku (SoC)** | np. Cyclone V SoC (DE10-Nano) albo Zynq: rdzenie ARM + FPGA w jednym chipie | brak osobnego MCU, ogromna przepustowość CPU ↔ FPGA, Linux na ARM | trudniejsze narzędzia, droższa płytka |
| D. Procesor programowy w FPGA (Nios V / RISC-V) | MCU "zbudowany" z logiki FPGA | jeden chip | zjada miejsce na p-bity, wolny |
| E. Karta PCIe z FPGA | FPGA w komputerze | najszybsze | drogie, skomplikowane |

> [!success] Rekomendacja
> - **Prototyp: wariant A**, bo pozwala 5 osobom pracować równolegle (firmware i FPGA niezależnie), a STM32 przyda się do TRNG.
> - **Wersja docelowa: rozważyć wariant C (SoC FPGA)**. Rolę MCU przejmuje wtedy ARM w tym samym chipie. Decyzja przy wyborze płytki docelowej, patrz [[08 Sprzęt - prototyp i płytka docelowa]].
> - Żeby przejście A → C było łatwe: **protokół PC ↔ "kontroler" i mapa rejestrów FPGA niezależne od tego, czym jest kontroler.**

---

## Gdzie trzymać "bibliotekę wag" (zewnętrzny RAM)

| Miejsce | Za | Przeciw |
| --- | --- | --- |
| **SDRAM przy FPGA** (DE10-Lite ma 64 MB) | blisko obliczeń, szybkie przełączanie problemów | trzeba zbudować kontroler SDRAM w FPGA |
| RAM / flash / SD przy MCU | prostsze (biblioteki do SD/QSPI istnieją) | każdy problem idzie przez SPI (to wciąż ułamki sekundy) |

**Na prototyp:** biblioteka przy MCU (albo po prostu na PC), a SDRAM przy FPGA jako etap późniejszy. Szczegóły w [[04 FPGA - macierz p-bitów#Wagi i pamięć]].
