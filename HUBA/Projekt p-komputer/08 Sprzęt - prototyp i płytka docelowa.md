---
tags: [p-komputer, sprzęt, fpga, decyzje]
---
# 08 Sprzęt: prototyp i płytka docelowa

Wróć: [[00 Przegląd projektu]]

> [!abstract] Założenie
> **Prototyp powstaje na DE10-Lite**, którą już mamy. **Docelowo możemy kupić inną płytkę.** Wybór dopiero wtedy, gdy prototyp pokaże, ile zasobów zużywa jedna jednostka p-bitu, czyli **po pomiarach, nie przed**.

---

## Prototyp: Terasic DE10-Lite

| Co | Wartość | Znaczenie dla projektu |
| --- | --- | --- |
| FPGA | Intel **MAX 10** `10M50DAF484C7G` | ~50 tys. elementów logicznych |
| Pamięć wewnętrzna | ~1,6 Mbit (bloki M9K) | wagi **N ≈ 256 gęstej** albo tysiące p-bitów w grafie rzadkim |
| Mnożarki | 144 (18×18) | mnożenie ×β |
| **SDRAM** | 64 MB (16-bit) | biblioteka wag ("zewnętrzny RAM") |
| Złącza | GPIO 2×20, złącze Arduino | podłączenie STM32 (SPI) i modułu TRNG |
| Do testów | 10 diod, 10 przełączników, 2 przyciski, 6 wyświetlaczy 7-seg., VGA, akcelerometr | debugowanie bez MCU (np. bramka AND na diodach) |
| Łącze z PC | tylko USB-Blaster (programowanie / JTAG) | **brak szybkiego łącza do PC**, więc komunikacja przez MCU (zgodnie z planem) |
| Narzędzia | Quartus Prime Lite (darmowy) | już zainstalowany, patrz [[W jaki sposób otworzyć i używać projektu verilog]] |

> [!success] Do czego DE10-Lite wystarczy
> Kamienie milowe **M0–M4** z [[09 Zespół, interfejsy i plan]]: pojedynczy p-bit, małe sieci, cały pipeline PC → MCU → FPGA, pierwsze benchmarki na małych grafach, podłączenie TRNG.

> [!warning] Ograniczenia
> - Za mały na benchmarki typu G-set (800+ wierzchołków o wielu połączeniach) w szybkiej, równoległej wersji
> - Brak procesora w chipie, Ethernetu i szybkiego USB

---

## Kryteria wyboru płytki docelowej

| Kryterium | Dlaczego ważne |
| --- | --- |
| **Elementy logiczne / LUT-y** | ile jednostek p-bitów zmieści się **równolegle** |
| **Pamięć blokowa (BRAM)** | jak duże macierze wag liczymy z pamięci wewnętrznej |
| **Bloki DSP** | mnożenie ×β, ewentualnie bardziej złożone jednostki |
| **Zewnętrzna pamięć** (DDR) | biblioteka problemów, duże instancje |
| **Procesor w chipie (SoC)** | może **zastąpić STM32**, patrz [[03 Mikrokontroler#Czy MCU w ogóle jest potrzebny? (analiza alternatyw)]] |
| Łącza (Ethernet, USB 3, PCIe) | szybkość komunikacji z PC |
| **Darmowe narzędzia** | wersje Lite/Standard/open source pokrywają tylko część układów |
| Dokumentacja i społeczność | 5 osób uczących się: przykłady są kluczowe |
| Producent (Intel vs AMD/Xilinx vs Lattice) | przesiadka = nowe narzędzia (Quartus → Vivado), nowe IP |
| Cena | sprawdzić aktualne ceny i **ceny akademickie** (często dużo niższe) |

---

## Kandydaci (do sprawdzenia przed zakupem)

| Płytka | Układ | Procesor w chipie | Narzędzia | Komentarz |
| --- | --- | --- | --- | --- |
| **Terasic DE10-Nano** | Intel Cyclone V SoC (~110 tys. LE) | ✅ 2× ARM Cortex-A9 + DDR3 | Quartus Lite ✅ (ten sam co teraz) | **naturalny następca**: ta sama rodzina narzędzi, ARM zastępuje MCU, popularna i tania |
| Terasic DE1-SoC | Cyclone V SoC (~85 tys. LE) | ✅ ARM | Quartus Lite ✅ | dużo peryferiów edukacyjnych |
| Digilent Arty A7-100T | AMD Artix-7 100T | ❌ | Vivado (darmowa wersja) | popularna, DDR3, Ethernet; zmiana narzędzi |
| TUL PYNQ-Z2 | AMD Zynq-7020 | ✅ ARM + **Python (PYNQ)** | Vivado | sterowanie FPGA z Pythona, pasuje do zespołu |
| AMD Kria KV260 | Zynq UltraScale+ | ✅ 4× ARM A53, Linux | Vivado | dużo mocniejsza, droższa; na "wersję 2" |
| ULX3S | Lattice ECP5 | ❌ | **w pełni open source** (Yosys + nextpnr) | lekkie narzędzia na Linuksie, mniejsza moc |

> [!tip] Wstępna rekomendacja
> **DE10-Nano**: zostajemy przy Quartusie (wiedza z prototypu przechodzi 1:1), ~2× więcej logiki niż MAX 10, a **ARM w chipie** pozwala uprościć architekturę (wariant C w [[03 Mikrokontroler]]). Decyzję podjąć **po M3**, z pomiarem zasobów na jednostkę p-bitu w ręku.

---

## Moduł TRNG (osobna płytka)
- **PCB z elementów dyskretnych** (KiCad), podłączany do MCU lub FPGA przez złącze
- osobne, filtrowane zasilanie części analogowej
- szczegóły: [[05 TRNG - generator szumu]]

---

## Jak pisać, żeby przesiadka była łatwa
Zasady w [[04 FPGA - macierz p-bitów#Przenośność (bo płytka docelowa może być inna)]]. W skrócie: przenośny rdzeń, rzeczy zależne od płytki w `boards/<płytka>/`, wszystko sparametryzowane.
