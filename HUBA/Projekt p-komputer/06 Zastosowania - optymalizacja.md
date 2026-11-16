---
tags: [p-komputer, zastosowania, optymalizacja, benchmarki]
---
# 06 Zastosowania: optymalizacja

Wróć: [[00 Przegląd projektu]] · Jak kodować problemy: [[01 Teoria - p-bity, Ising, QUBO]], [[02 Kompilator (PC)]]

---

## Które problemy pasują do p-komputera

> [!tip] Zasada kciuka
> p-komputer jest **najlepszy**, gdy problem jest **naturalnie kwadratowy** (zależności parami), **bez ograniczeń albo z niewieloma**, i ma **rzadki** graf zależności.
> Jest **najsłabszy** przy wielu twardych ograniczeniach, bo kary robią "pofałdowany" krajobraz energii, a klasyczne solwery są tam bardzo dobre.

| Problem | Przykład biznesowy | Liczba p-bitów | Dopasowanie |
| --- | --- | --- | --- |
| **Max-Cut / podział grafu** | podział sieci / układu na części z minimalną komunikacją, klasteryzacja | N (wierzchołki) | ⭐⭐⭐ idealny |
| **Szkło spinowe / model Isinga** | benchmarki, fizyka | N | ⭐⭐⭐ |
| **Optymalizacja portfela** (Markowitz, wybór k aktywów) | finanse | N aktywów × bity precyzji | ⭐⭐ (naturalnie kwadratowy) |
| **Pokrycie zbioru / rozmieszczenie** | gdzie postawić sensory, kamery, serwery | N kandydatów + zmienne pomocnicze | ⭐⭐ |
| **Kolorowanie grafu / harmonogramy** | plan zajęć, przydział częstotliwości | N × K kolorów | ⭐⭐ (one-hot = kary) |
| **Plecak** (budżet) | wybór projektów / łatek w budżecie | N + log(budżet) | ⭐ (nierówność → slack) |
| **TSP / VRP (trasy)** | logistyka | **N²** (20 miast = 400 p-bitów) | ⭐ słaby: wiele ograniczeń, a klasyczne heurystyki (np. LKH) są świetne |
| **MaxSAT / SAT** | weryfikacja, konfiguracja | zmienne + klauzule | ⭐⭐ |

---

## Konkurencja (z kim się porównujemy)

| Kategoria | Przykłady | Uwagi |
| --- | --- | --- |
| Klasyczne solwery dokładne | Gurobi, CPLEX, OR-Tools (CP-SAT) | dla małych/średnich problemów często dają **optimum** |
| Heurystyki na CPU/GPU | wyżarzanie symulowane, parallel tempering, tabu search | **najważniejszy punkt odniesienia**: to dokładnie "p-komputer w oprogramowaniu" |
| Cyfrowe "annealery" | **Fujitsu Digital Annealer**, **Toshiba SQBM+** | komercyjne, dedykowany sprzęt / chmura: nasza najbliższa konkurencja |
| Kwantowe | D-Wave (wyżarzanie kwantowe, chmura) | |

> [!warning] Uczciwe pytanie, na które projekt musi odpowiedzieć
> **"Dlaczego nie uruchomić wyżarzania symulowanego na GPU?"**
> Możliwe odpowiedzi, które trzeba **zmierzyć**:
> - **energia** na rozwiązanie (FPGA zwykle zużywa dużo mniej niż GPU)
> - **opóźnienie / czas rzeczywisty** (przewidywalny czas odpowiedzi)
> - **on-premise / urządzenie brzegowe** (dane nie opuszczają firmy, patrz [[07 Zastosowania - cyberbezpieczeństwo]])
> - **koszt** sprzętu
>
> Sama "szybkość" na prototypie MAX 10 prawdopodobnie **nie** pokona nowoczesnego CPU/GPU. To normalne dla prototypu, ale trzeba wiedzieć, jaki argument budujemy.

---

## Metryki

| Metryka | Znaczenie |
| --- | --- |
| **Time-to-solution (TTS₉₉)** | czas potrzebny, żeby z 99% prawdopodobieństwem znaleźć rozwiązanie optymalne (albo w X% od najlepszego znanego) |
| **Energy-to-solution** | to samo w dżulach (pomiar poboru mocy płytki) |
| **Współczynnik przybliżenia** | wynik / najlepszy znany wynik |
| **Flips/s** (aktualizacje p-bitów na sekundę) | surowa przepustowość sprzętu (publikacje o p-komputerach podają flips/ns) |

> [!important] Porównania zawsze w tych samych warunkach
> Ten sam problem, to samo kryterium sukcesu, ten sam limit czasu, czas **całego** procesu (z kompilacją i transferem) **oraz** samego liczenia, raportowane osobno.

---

## Zbiory testowe (benchmarki)

| Zbiór | Problem | Uwagi |
| --- | --- | --- |
| **G-set** | Max-Cut | standard dla maszyn Isinga; znane najlepsze wyniki. Najmniejsze grafy mają **800 wierzchołków**, czyli raczej na płytkę docelową |
| losowe grafy (własne) | Max-Cut | na prototyp: 16–256 wierzchołków, optimum policzone Gurobi / brute force |
| Sherrington–Kirkpatrick (SK) | szkło spinowe, gęsty | test gęstej macierzy |
| SAT competition (małe instancje), MaxSAT | SAT | |
| QPLIB | QUBO / problemy kwadratowe | |
| TSPLIB (małe) | TSP | raczej jako pokaz ograniczeń |

---

## Proponowana ścieżka demonstracji

1. **Max-Cut na losowych grafach** (N=16 → 256). Porównanie: p-komputer vs wyżarzanie na CPU vs optimum (Gurobi). *Najprostszy, najbardziej "uczciwy" pokaz.*
2. **Problem z ograniczeniami:** np. rozmieszczenie k sensorów (pokrycie zbioru). Pokazuje cały kompilator: kary, dekodowanie, sprawdzanie ograniczeń.
3. **Zastosowanie "biznesowe" end-to-end** z [[07 Zastosowania - cyberbezpieczeństwo]] (np. utwardzanie sieci w budżecie).
4. **G-set** na płytce docelowej.

---

## Pozycjonowanie (do dyskusji w zespole)
- **Nie** "szybszy niż wszystko", tylko np. **"energooszczędny akcelerator optymalizacji on-premise"** albo **"optymalizacja dla bezpieczeństwa bez wysyłania danych do chmury"**.
- Sprzęt to tylko część wartości. **Biblioteka szablonów problemów + benchmarki** ([[02 Kompilator (PC)]]) to coś, co użytkownik faktycznie zobaczy.
