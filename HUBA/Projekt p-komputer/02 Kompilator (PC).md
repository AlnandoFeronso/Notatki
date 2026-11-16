---
tags: [p-komputer, kompilator, software]
warstwa: 1
---
# 02 Kompilator (PC)

Wróć: [[00 Przegląd projektu]] · Teoria: [[01 Teoria - p-bity, Ising, QUBO]]

> [!abstract] Rola
> Zamienia **problem biznesowy** w liczby, które zrozumie FPGA (`J`, `h`, harmonogram β), wysyła je przez mikrokontroler, a wynik (ciąg bitów) **tłumaczy z powrotem** na odpowiedź dla użytkownika.

> [!important] To najważniejsza warstwa projektu
> Sprzęt tylko próbkuje rozkład. **Jakość wyniku decyduje się tutaj**: w tym, jak dobrze problem jest zakodowany. Najczęstsza przyczyna porażek maszyn Isinga to słabe kodowanie, a nie słaby sprzęt.

---

## Etapy kompilacji

```
problem (np. "rozmieść 5 sensorów w sieci 200 hostów")
   │
   ▼ 1. Modelowanie        zmienne binarne x, funkcja celu, ograniczenia
   ▼ 2. QUBO               ograniczenia → kary P·(…)²
   ▼ 3. Ising              x = (1+m)/2  →  J, h
   ▼ 4. Skalowanie         dopasowanie do zakresu sprzętu (np. wagi 8-bitowe)
   ▼ 5. Odwzorowanie       na topologię i rozmiar FPGA (+ kolorowanie grafu)
   ▼ 6. Harmonogram        β(t), liczba przebiegów, liczba powtórzeń
   │
   ▼  ── wysłanie do MCU ──►  FPGA liczy  ──►  wyniki
   │
   ▼ 7. Dekodowanie        bity → odpowiedź; sprawdzenie ograniczeń
   ▼ 8. Poprawa (opcja)    szybkie lokalne przeszukiwanie na CPU
   ▼
odpowiedź + jakość + czas
```

### 1–2. Modelowanie i QUBO
- Gotowe "przepisy" na typowe problemy: Max-Cut, pokrycie zbioru, plecak, kolorowanie grafu, przydział zadań, TSP. Większość jest opisana w przeglądzie Lucas, *Ising formulations of many NP problems* (2014).
- **Kary za ograniczenia:** $P \cdot (\text{naruszenie})^2$. Dobór $P$: zacznij od wartości porównywalnej z maksymalną zmianą funkcji celu, potem stroić automatycznie (np. zwiększać, dopóki rozwiązania łamią ograniczenia).
- **Ograniczenia nierównościowe** (np. "koszt ≤ budżet") wymagają **zmiennych pomocniczych** (slack), co zwiększa liczbę p-bitów. Trzeba to uwzględnić w budżecie sprzętu.

### 4. Skalowanie i kwantyzacja

> [!warning] Precyzja wag
> FPGA będzie liczyć na liczbach całkowitych (np. 8 bitów: −128…127). Jeśli wagi w problemie mają rozpiętość 1 : 10 000, po zaokrągleniu **małe wagi znikną**, a kara może "zjeść" funkcję celu.
> - Mierzyć: **zakres dynamiki** = max|J| / min|J≠0|
> - Raportować użytkownikowi, ile informacji traci kwantyzacja
> - Testować: rozwiązanie modelu z pełną precyzją kontra po kwantyzacji (w modelu referencyjnym)

### 5. Odwzorowanie na sprzęt
- **Gęsta macierz** (każdy z każdym): wystarczy, że N ≤ pojemność FPGA.
- **Stała rzadka topologia** (np. kratka, każdy ma 8 sąsiadów): problem trzeba **osadzić** (*minor embedding*), czyli jeden logiczny bit zamienić na łańcuch kilku fizycznych p-bitów. To kosztowne i trudne. **Unikać, jeśli się da**: lepiej, żeby FPGA przyjmowało dowolny rzadki graf (lista sąsiadów w pamięci).
- **Kolorowanie grafu** (dla równoległych aktualizacji): liczone tutaj, na PC, i wysyłane jako część konfiguracji.

### 7–8. Wyniki
- Zawsze **sprawdzić ograniczenia** w dekodowanym wyniku, bo sprzęt probabilistyczny może zwrócić rozwiązanie niepoprawne.
- Szybkie **lokalne poprawki na CPU** (greedy / 1-flip) często tanio podnoszą jakość. Uczciwie raportować, ile zrobił sprzęt, a ile CPU.

---

## Język i narzędzia

| Wybór | Dlaczego |
| --- | --- |
| **Python** (rekomendacja) | najszybszy rozwój, gotowe biblioteki, ten sam język co model referencyjny i testy FPGA (cocotb) |
| Rust (później, opcjonalnie) | gdy wąskim gardłem stanie się wydajność kompilatora (duże grafy) |

**Przydatne biblioteki** (sprawdzić licencje przed użyciem w produkcie):
- `dimod` (D-Wave Ocean, open source): standardowe struktury BQM/QUBO, konwersje, gotowe solwery referencyjne (simulated annealing)
- `PyQUBO`: zapis wyrażeń z karami → QUBO
- `networkx`: grafy, kolorowanie
- `numpy` / `scipy`: macierze
- **OR-Tools** / **Gurobi** (licencja akademicka): **klasyczny punkt odniesienia** do benchmarków

> [!tip] "Język dla użytkownika"
> Pomysł "kompilatora zadań biznesowych" warto zacząć **od biblioteki Pythona z gotowymi szablonami problemów** (np. `solve_maxcut(graph)`, `place_sensors(network, k)`), a nie od własnego języka (DSL). Własny język ma sens dopiero, gdy wiadomo, jakich problemów użytkownicy naprawdę potrzebują.

---

## Interfejs z resztą systemu
Kompilator produkuje **plik/pakiet problemu** wg wspólnej specyfikacji, patrz [[09 Zespół, interfejsy i plan#Kontrakt A - format problemu]].

Ważne: kompilator powinien umieć **wysłać problem do trzech "backendów"** przez to samo API:
1. **model referencyjny w Pythonie** (bez sprzętu)
2. **klasyczny solver** (benchmark)
3. **prawdziwy sprzęt** (MCU → FPGA)

Dzięki temu zespół kompilatora nie czeka na sprzęt, a porównania są uczciwe.

---

## Ryzyka
- Kodowanie z karami daje "trudny krajobraz energii", więc sprzęt może dawać gorsze wyniki niż prosty algorytm zachłanny. Rozwiązanie: benchmarki od początku.
- Liczba p-bitów rośnie przez zmienne pomocnicze. Rozwiązanie: liczyć budżet p-bitów dla każdego szablonu problemu.
- Więcej w [[10 Ryzyka i otwarte pytania]].
