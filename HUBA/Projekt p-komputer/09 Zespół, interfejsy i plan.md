---
tags: [p-komputer, zespół, plan, interfejsy]
zespół: 5 osób
---
# 09 Zespół, interfejsy i plan

Wróć: [[00 Przegląd projektu]]

> [!abstract] Klucz do pracy w 5 osób
> **Kontrakty (interfejsy) najpierw.** Jeśli od początku ustalimy, *co* przechodzi między warstwami, każdy może pracować równolegle, zamiast czekać na innych. Kontrakty niżej to **szkice v0** do uzgodnienia na pierwszym spotkaniu.

---

## Proponowany podział ról

| # | Rola | Zakres | Główne notatki | Język / narzędzia |
| --- | --- | --- | --- | --- |
| 1 | **Kompilator i zastosowania** | modelowanie problemów → QUBO → J, h; szablony problemów; dekodowanie wyników | [[02 Kompilator (PC)]], [[06 Zastosowania - optymalizacja]] | Python |
| 2 | **Model referencyjny, weryfikacja, benchmarki** | model p-bitów w Pythonie, testy cocotb dla FPGA, porównania z CPU/Gurobi | [[01 Teoria - p-bity, Ising, QUBO]], [[06 Zastosowania - optymalizacja]] | Python, cocotb |
| 3 | **FPGA: rdzeń p-bitów** | jednostka p-bitu, PRNG, pamięć wag, energia, statystyki | [[04 FPGA - macierz p-bitów]] | Verilog |
| 4 | **MCU i komunikacja** | firmware STM32, protokół z PC, SPI slave + mapa rejestrów po stronie FPGA | [[03 Mikrokontroler]] | C/Rust, Verilog |
| 5 | **TRNG i bezpieczeństwo** | moduł szumu (PCB), testy entropii, zastosowania bezpieczeństwa | [[05 TRNG - generator szumu]], [[07 Zastosowania - cyberbezpieczeństwo]] | KiCad, C, Python |

> [!tip] Zasady
> - Każdy interfejs ma **jednego właściciela**: osoba 4 jest właścicielem kontraktów B i D, osoba 1 kontraktu A, osoba 5 kontraktu C.
> - Osoby 2 i 3 pracują **w parze**: testy (2) powstają razem z RTL (3).
> - Rola "integratora" (składa całość na koniec kamienia milowego) **rotuje**, żeby każdy znał cały system.

---

## Kontrakty między warstwami

### Kontrakt A - format problemu
PC → MCU (a w testach PC → model referencyjny). **v0: JSON** (czytelny, łatwy do debugowania), później wersja binarna.

```json
{
  "version": 0,
  "n": 3,
  "h": [1, 1, -2],
  "J": [[0, 1, -1], [0, 2, 2], [1, 2, 2]],
  "scale": 1.0,
  "colors": [[0], [1], [2]],
  "schedule": { "beta": [0.1, 0.5, 1.0, 2.0], "sweeps_per_step": 1000 },
  "restarts": 10,
  "seed": null
}
```
- `J`: lista rzadka `[i, j, waga]` (tylko i < j; FPGA uzupełnia symetrię)
- wagi **już skwantowane** do zakresu sprzętu (np. −128…127); `scale` mówi, jak wrócić do oryginalnych jednostek
- `colors`: grupy do równoległej aktualizacji (puste = sekwencyjnie)
- `seed: null` = użyj TRNG

### Kontrakt B - mapa rejestrów FPGA
MCU ↔ FPGA. **Ramka SPI v0:** `[1 bit R/W | 7 bitów adres] [32 bity dane]`.

| Adres | Nazwa | R/W | Opis |
| --- | --- | --- | --- |
| `0x00` | `ID` | R | stała, np. `0x50B17001`: test, czy SPI działa |
| `0x01` | `CTRL` | W | bit0 start, bit1 reset, bit2 zeruj statystyki, bit3 przeładuj ziarna |
| `0x02` | `STATUS` | R | bit0 busy, bit1 done, bit2 błąd |
| `0x03` | `N` | R/W | liczba aktywnych p-bitów |
| `0x04` | `BETA` | R/W | β w formacie stałoprzecinkowym (np. Q8.8) |
| `0x05` | `SWEEPS` | R/W | liczba przebiegów do wykonania |
| `0x08` | `W_ADDR` | W | adres w pamięci wag |
| `0x09` | `W_DATA` | W | dane wag (adres zwiększa się automatycznie) |
| `0x10`–`0x13` | `SEED0..3` | W | 128-bitowe ziarno dla PRNG |
| `0x20` | `BEST_E` | R | energia najlepszego stanu |
| `0x21`… | `BEST_STATE` | R | najlepszy stan, 32 p-bity na słowo |
| `0x40`… | `STATS` | R | liczniki ⟨mᵢ⟩ |

> [!note] Ta sama mapa w wariancie FMC i SoC
> Przy przejściu na FMC albo ARM w chipie ([[03 Mikrokontroler#Czy MCU w ogóle jest potrzebny? (analiza alternatyw)]]) zmienia się tylko **sposób dostępu**, nie adresy. Firmware i RTL nie muszą się zmieniać.

### Kontrakt C - losowość (TRNG ↔ MCU ↔ FPGA)
- **TRNG → MCU:** strumień bitów z komparatora (GPIO) albo próbki ADC
- **MCU:** testy zdrowia (Repetition Count, Adaptive Proportion wg SP 800-90B); przy błędzie flaga + ziarna z wbudowanego RNG STM32
- **MCU → FPGA:** ziarna przez `SEED0..3` + bit `CTRL.3`; wariant 2 (ciągłe odświeżanie) później
- szczegóły: [[05 TRNG - generator szumu]]

### Kontrakt D - protokół PC ↔ MCU
**v0:** USB CDC (wirtualny port szeregowy), ramka: `[0xA5] [typ] [długość 2 B] [dane] [CRC16]`

| Typ | Kierunek | Znaczenie |
| --- | --- | --- |
| `PING` / `PONG` | PC ↔ MCU | test połączenia + wersje firmware i bitstreamu |
| `LOAD_PROBLEM` | PC → MCU | problem (kontrakt A) |
| `RUN` | PC → MCU | start z harmonogramem |
| `STATUS` | MCU → PC | postęp, błędy |
| `RESULT` | MCU → PC | najlepszy stan + energia + czas |
| `STATS` | MCU → PC | statystyki |

---

## Struktura repozytorium (jedno repo dla wszystkich)

```
p-komputer/
├── model/          # model referencyjny (Python): WSPÓLNY dla kompilatora i testów FPGA
├── compiler/       # kompilator problemów (Python)
├── benchmarks/     # zbiory testowe, skrypty porównań, wyniki
├── firmware/       # STM32 (C/Rust)
├── fpga/
│   ├── rtl/        # przenośny Verilog (rdzeń)
│   ├── sim/        # testy cocotb
│   └── boards/
│       ├── de10lite/   # piny, PLL, Makefile dla prototypu
│       └── <docelowa>/
├── trng/           # schematy KiCad, skrypty testów entropii
└── docs/           # specyfikacje kontraktów A–D (źródło prawdy)
```

> [!tip] Praktyki
> - **Zmiany w kontraktach tylko przez pull request** z akceptacją właściciela i osób, których dotyczą
> - **CI**: przy każdym commicie testy modelu (pytest) i symulacje FPGA (cocotb + Icarus/Verilator)
> - Jeden **model referencyjny** to jedno źródło prawdy, z którym porównują się kompilator, FPGA i benchmarki
> - Cotygodniowe krótkie demo: każdy pokazuje coś, co **działa**

---

## Kamienie milowe

| # | Kamień milowy | Kto głównie | Gotowe, gdy… | Zależy od |
| --- | --- | --- | --- | --- |
| **M0** | Model referencyjny + kontrakty v0 | 1, 2 (+wszyscy: kontrakty) | bramka AND i Max-Cut N≤16 zgadzają się z rozkładem dokładnym; kontrakty A–D zatwierdzone | — |
| **M1** | p-bit i mała sieć na FPGA | 3, 2 | bramka AND na diodach DE10-Lite; histogram z symulacji zgodny z modelem (test χ²) | M0 |
| **M2** | Pełny łańcuch PC → MCU → FPGA | 4, 3, 1 | `LOAD_PROBLEM` + `RUN` z PC zwraca poprawny wynik dla AND i Max-Cut N=16 | M1 |
| **M3** | Benchmarki v1 | 2, 1 | Max-Cut N=16…256: jakość i czas vs wyżarzanie na CPU vs optimum; **pomiar LE na jednostkę p-bitu** | M2 |
| **M4** | TRNG podłączony | 5, 4 | ziarna z TRNG; testy zdrowia działają; raport z narzędzia NIST SP 800-90B | M2 (równolegle od M0) |
| **M5** | Zastosowanie end-to-end | 1, 5 | rozmieszczenie sensorów IDS ([[07 Zastosowania - cyberbezpieczeństwo]]) od opisu sieci do wizualizacji | M3 |
| **M6** | Decyzja o płytce docelowej | wszyscy | wybór na podstawie danych z M3 ([[08 Sprzęt - prototyp i płytka docelowa]]) | M3 |

```
M0 ──► M1 ──► M2 ──► M3 ──► M5
 │             │      └───► M6
 └── TRNG (5) ─┴──► M4
```

---

## Pierwsze kroki dla każdej roli (od razu, bez czekania)

| Rola | Pierwsze zadanie |
| --- | --- |
| 1 Kompilator | QUBO → Ising (wzory w [[01 Teoria - p-bity, Ising, QUBO]]), szablon Max-Cut, zapis do formatu A |
| 2 Weryfikacja | model z notatki 01 jako pakiet `model/` + testy pytest; pierwszy testbench cocotb |
| 3 FPGA | nauka z [[Podstawy składni Verilog]] → moduł PRNG (xorshift32) → jednostka p-bitu |
| 4 MCU | Nucleo: "hello" przez USB CDC, SPI master; po stronie FPGA SPI slave z rejestrem `ID` |
| 5 TRNG | przegląd OneRNG / Infinite Noise, schemat v0 w KiCad, zakup części; równolegle test RNG w STM32 |

---

## Dziennik decyzji

| Data | Decyzja | Uzasadnienie | Status |
| --- | --- | --- | --- |
| | Prototyp na DE10-Lite | mamy płytkę, darmowy Quartus | ✅ |
| | TRNG zasila ziarna PRNG, nie p-bity bezpośrednio | przepustowość ([[05 TRNG - generator szumu]]) | 🟡 do potwierdzenia |
| | Wagi liczone z pamięci wewnętrznej, RAM = biblioteka | przepustowość ([[04 FPGA - macierz p-bitów]]) | 🟡 do potwierdzenia |
| | MCU ↔ FPGA: SPI na prototyp | prostota | 🟡 |
| | Kompilator w Pythonie | ekosystem, wspólny język z testami | 🟡 |
| | Wybór MCU | | ⬜ otwarte |
| | Płytka docelowa | po M3 | ⬜ otwarte |
