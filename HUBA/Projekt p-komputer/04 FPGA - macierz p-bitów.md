---
tags: [p-komputer, fpga, verilog, architektura]
warstwa: 3
---
# 04 FPGA: macierz p-bitów

Wróć: [[00 Przegląd projektu]] · Teoria: [[01 Teoria - p-bity, Ising, QUBO]] · Verilog od zera: [[Podstawy składni Verilog]]

> [!abstract] Rola
> Serce systemu: trzyma wagi `J`, `h` i stany p-bitów, aktualizuje je miliony razy na sekundę, śledzi energię i najlepsze znalezione rozwiązanie, liczy statystyki.

---

## Architektura wewnątrz FPGA

```
          z MCU (SPI / FMC)
                │
        ┌───────▼────────┐
        │ interfejs +    │  CTRL, STATUS, BETA, adresy/dane wag, wyniki
        │ mapa rejestrów │
        └───┬───────┬────┘
            │       │
   ┌────────▼──┐  ┌─▼──────────────────────────────────────────┐
   │ pamięć wag│  │             SILNIK p-BITÓW                  │
   │ J, h, lista──►  ┌───────────┐  ┌──────┐  ┌─────┐  ┌──────┐ │
   │ sąsiadów  │  │  │ suma      │─►│ ×β   │─►│ LUT │─►│ porów│─┼──► nowy mᵢ
   │ (M9K)     │  │  │ Σ ±Jᵢⱼ+hᵢ │  │      │  │ tanh│  │ nanie│ │
   └─────▲─────┘  │  └───────────┘  └──────┘  └─────┘  └──▲───┘ │
         │        │                                       │     │
   ┌─────┴─────┐  │  ┌──────────────┐               ┌────┴───┐ │
   │ SDRAM     │  │  │ stany m (rej.)│               │ PRNG   │◄┼── ziarna z TRNG
   │ (później) │  │  └──────────────┘               │xoshiro │ │
   └───────────┘  └──────────────┬─────────────────────────────┘
                                 │
                  ┌──────────────▼──────────────┐
                  │ energia E, najlepszy stan,  │──► do MCU
                  │ liczniki ⟨mᵢ⟩, ⟨mᵢmⱼ⟩       │
                  └─────────────────────────────┘
```

---

## Jednostka p-bitu (krok po kroku)

| Krok | Co | Jak w sprzęcie |
| --- | --- | --- |
| 1 | $I_i = h_i + \sum_j J_{ij} m_j$ | **bez mnożenia!** $m_j = \pm1$, więc $J_{ij} m_j$ to $+J_{ij}$ albo $-J_{ij}$: sumator/odejmator |
| 2 | $\beta \cdot I_i$ | **jedno** mnożenie (MAX 10 ma sprzętowe mnożarki) |
| 3 | $\tanh(\cdot)$ | **tablica (LUT/ROM)**: obcięte i przesunięte $\beta I$ jako adres, próg jako wartość |
| 4 | porównanie z $r$ | $r$ z generatora PRNG, nowe przy **każdej** aktualizacji |
| 5 | nowy $m_i$ | zapis do rejestru stanów |

**Proponowana precyzja na start:** wagi 8 bitów ze znakiem, suma 16 bitów, tablica tanh 256–1024 wpisów. Wszystko jako `parameter`, żeby dało się zmieniać.

> [!tip] Energia i najlepsze rozwiązanie liczone "za darmo"
> Gdy $m_i$ zmienia się z $s$ na $-s$, energia zmienia się o:
> $$\Delta E = 2\, s\, I_i$$
> a $I_i$ i tak już mamy policzone! *(Sprawdzone numerycznie dla wszystkich stanów sieci 6 p-bitów.)*
> FPGA może więc śledzić **energię na bieżąco** i **zapamiętać stan o najniższej energii**. Do optymalizacji to wystarczy: MCU odczytuje tylko najlepszy stan.

---

## Kolejność aktualizacji

| Schemat | Jak | Za | Przeciw |
| --- | --- | --- | --- |
| **Sekwencyjny** | jeden p-bit na krok, po kolei | zawsze poprawny, najprostszy | wolny dla dużych N |
| **Kolorowanie grafu** | p-bity podzielone na "kolory" bez połączeń wewnątrz koloru, a cały kolor aktualizowany naraz | ogromna równoległość dla grafów rzadkich | wymaga kolorowania (robi kompilator) |
| Wszystkie naraz (synchronicznie) | ❌ | — | **błędne próbkowanie**, oscylacje |

> [!danger] Zasada
> **Dwa połączone p-bity nigdy nie są aktualizowane w tym samym takcie.** Patrz [[01 Teoria - p-bity, Ising, QUBO#7. Zasady, których nie wolno łamać (dla zespołu FPGA)]].

---

## Wagi i pamięć

### Ile miejsca zajmuje macierz

| Rodzaj | Wzór | N=256 | N=512 | N=1024 |
| --- | --- | --- | --- | --- |
| gęsta, 8 bitów | N² B | 64 KB ✅ | 256 KB ❌ | 1 MB ❌ |
| rzadka, d=16 sąsiadów (waga + indeks ~3 B) | ~3·N·d B | 12 KB | 24 KB | 48 KB ✅ |

**MAX 10 (10M50)** ma około **200 KB** pamięci wewnętrznej (bloki M9K, łącznie ~1,6 Mbit).

### Dlaczego nie liczyć bezpośrednio z zewnętrznego RAM-u

| | Wewnętrzna M9K | Zewnętrzny SDRAM (DE10-Lite) |
| --- | --- | --- |
| Pojemność | ~200 KB | 64 MB |
| Odczyt | wiele bloków **równolegle**, co takt | 16 bitów na takt, opóźnienia przy zmianie wiersza |
| Do czego | **liczenie** | **biblioteka problemów** |

Aktualizacja jednego p-bitu w gęstej macierzy wymaga całego wiersza `J`. Z SDRAM byłoby to rzędu **tysiąca pełnych przebiegów na sekundę** dla N=512, czyli **100–1000× wolniej** niż z M9K.

> [!success] Rekomendacja
> SDRAM (zewnętrzny RAM) = **magazyn wag**. Przed obliczeniami: **kopia do M9K**, liczenie wyłącznie z M9K. Bezpośrednie liczenie z SDRAM tylko, gdy problem nie mieści się w środku (świadomy kompromis szybkości).

> [!tip] Sztuczka na później: przyrostowe pola lokalne
> Zamiast liczyć $I_i$ od zera, trzymaj wszystkie $I_i$ w rejestrach. Gdy $m_j$ się zmienia, dodaj $\pm 2J_{ij}$ do każdego $I_i$. Przy niskiej temperaturze zmiany są rzadkie, więc to bardzo przyspiesza gęste macierze.

---

## Generatory losowe w FPGA

| Generator | Jakość | Koszt | Uwagi |
| --- | --- | --- | --- |
| LFSR (rejestr przesuwny) | słaba | najtańszy | sąsiednie bity silnie skorelowane, więc **nie dawać** kilku p-bitom bitów z jednego LFSR |
| **xorshift32 / xoshiro128+** | dobra | kilkadziesiąt LE | ✅ **rekomendacja**: osobny generator dla każdej równoległej jednostki |
| TRNG bezpośrednio | prawdziwa | — | **za wolny** (Mbit/s), patrz niżej |

**Ile losowości potrzeba:** 1 aktualizacja = ~8–10 bitów losowych.
- sekwencyjnie, 1 aktualizacja na takt przy 50 MHz: **~500 Mbit/s**
- 256 jednostek równolegle: **~100 Gbit/s**

TRNG na diodzie Zenera daje **rzędu Mbit/s**, czyli 100–100 000× za mało. Dlatego **TRNG zasila ziarna i okresowo "odświeża" (reseed) generatory PRNG**, ale nie karmi p-bitów bezpośrednio. Pełna analiza w [[05 TRNG - generator szumu]].

---

## Budżet zasobów MAX 10 (szacunek do weryfikacji)

| Zasób | Ma 10M50 | Zużycie |
| --- | --- | --- |
| Elementy logiczne | 49 760 | jednostka p-bitu ~100–300 LE (suma + LUT + porównanie + PRNG) |
| Pamięć M9K | ~1,6 Mbit | wagi, tablica tanh, statystyki |
| Mnożarki 18×18 | 144 | 1 na jednostkę (×β) |
| Zegar | 50 MHz (PLL do wyższych) | |

**Rząd wielkości:** gęsta N≈256 z sekwencyjną aktualizacją albo **kilkaset p-bitów równolegle** dla grafu rzadkiego. Konkretne liczby wyjdą po pierwszej syntezie (raport `fit.summary`).

---

## Weryfikacja (bez tego nie wiadomo, czy działa)

> [!important] Wyniki p-komputera są losowe, więc test "czy wynik = X" nie działa
> Testujemy **rozkład**: z FPGA (i z symulacji) zbieramy histogram stanów i porównujemy z modelem referencyjnym ([[01 Teoria - p-bity, Ising, QUBO#6. Model referencyjny (Python, bez bibliotek)]]) testem statystycznym (np. χ², odległość KL).

**Narzędzia:**
- **cocotb**: testbenche pisane w **Pythonie**, symulacja przez Icarus/Verilator. Ten sam model referencyjny co w kompilatorze, więc jeden język dla całego zespołu.
- Testy jednostkowe: pojedynczy p-bit (krzywa P(m=+1) od I), PRNG (porównanie z referencją bit po bicie), interfejs SPI.
- CI: testy uruchamiane automatycznie przy każdym commicie (GitHub Actions, iverilog/verilator).

---

## Przenośność (bo płytka docelowa może być inna)

> [!tip] Zasady, żeby przesiadka na inną płytkę była łatwa
> - Rdzeń (p-bity, PRNG, statystyki) w **czystym, przenośnym Verilogu**, bez IP specyficznych dla Intela
> - Pamięć pisana tak, żeby narzędzie samo **wywnioskowało** bloki RAM (standardowy wzorzec `reg [..] mem [..]`)
> - Rzeczy zależne od płytki (PLL, kontroler SDRAM, piny) w osobnym katalogu `boards/<płytka>/`
> - Wszystkie rozmiary (N, szerokość wag, liczba jednostek) jako `parameter`

Patrz [[08 Sprzęt - prototyp i płytka docelowa]].

---

## Etapy budowy
1. p-bit + PRNG + LUT tanh, testbench krzywej P(m=+1)
2. Sieć N=3–16, wagi w rejestrach, sekwencyjnie, bramka AND na diodach DE10-Lite
3. Interfejs SPI + mapa rejestrów ([[09 Zespół, interfejsy i plan#Kontrakt B - mapa rejestrów FPGA]])
4. Wagi w M9K, śledzenie energii i najlepszego stanu, liczniki
5. Równoległość (kolorowanie), wiele jednostek
6. SDRAM jako biblioteka wag
