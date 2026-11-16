---
tags: [p-komputer, teoria]
---
# Teoria: p-bity, model Isinga, QUBO

> [!info] Dla kogo
> Dla **całego zespołu**. To wspólny język: kompilator produkuje `J` i `h`, mikrokontroler je przesyła, a FPGA na nich liczy. Wróć: [[00 Przegląd projektu]]

---

## 1. p-bit

Zwykły bit ma wartość 0 albo 1. **p-bit (probabilistic bit)** ciągle i losowo przeskakuje między −1 a +1, a **wejście steruje tym, jak często** jest w którym stanie.

$$
m_i = \operatorname{sgn}\big(\tanh(\beta\, I_i) - r\big), \qquad r \sim U(-1, 1)
$$

| Symbol | Znaczenie |
| --- | --- |
| $m_i \in \{-1, +1\}$ | stan p-bitu |
| $I_i$ | wejście ("pole lokalne") |
| $\beta$ | odwrotność temperatury: duże β → mało losowo, małe β → bardzo losowo |
| $r$ | liczba losowa, **nowa przy każdej aktualizacji** |

Wynika z tego, że:

$$
P(m_i = +1) = \frac{1 + \tanh(\beta I_i)}{2}
$$

```
P(m=+1)
  1 ┤                ╭──────
    │              ╭─╯
0.5 ┤ ─ ─ ─ ─ ─ ─ ●   ← I = 0: rzut monetą
    │          ╭─╯
  0 ┤──────────╯
    └──────────┼──────────► I
               0
```

---

## 2. Sieć p-bitów = model Isinga

p-bity łączymy wagami. Wejście p-bitu to ważona suma stanów sąsiadów:

$$
I_i = h_i + \sum_j J_{ij}\, m_j
$$

- $J_{ij}$: **waga połączenia** (macierz symetryczna, $J_{ii} = 0$). $J > 0$: p-bity "chcą" być zgodne. $J < 0$: przeciwne.
- $h_i$: **bias**, czyli preferencja p-bitu do +1 albo −1.

Taka sieć ma **energię**:

$$
E(m) = -\sum_{i<j} J_{ij}\, m_i m_j - \sum_i h_i\, m_i
$$

**Kluczowy fakt:** gdy p-bity aktualizujemy **pojedynczo** (albo tylko niepołączone naraz), sieć **próbkuje rozkład Boltzmanna**:

$$
P(m) \propto e^{-\beta E(m)}
$$

Stany o **niskiej energii** pojawiają się **najczęściej**. Jeśli zakodujemy problem tak, że **najlepsze rozwiązanie = najniższa energia**, sieć sama będzie je najczęściej "pokazywać". To cała idea p-komputera.

> [!tip] Wyżarzanie (annealing)
> Zaczynamy od małego β (wysoka temperatura, dużo losowości, sieć swobodnie eksploruje), a potem **stopniowo zwiększamy β** (sieć "zamarza" w głębokim minimum energii). Tak działa optymalizacja na p-komputerze. Harmonogram β ustawia mikrokontroler.

---

## 3. QUBO, czyli jak zapisuje się problemy

Większość problemów optymalizacyjnych zapisuje się jako **QUBO** (Quadratic Unconstrained Binary Optimization):

$$
\min_{x \in \{0,1\}^n} \; f(x) = \sum_{i<j} Q_{ij}\, x_i x_j + \sum_i q_i\, x_i
$$

**Zamiana QUBO → Ising** przez podstawienie $x_i = \frac{1 + m_i}{2}$:

$$
J_{ij} = -\frac{Q_{ij}}{4}, \qquad h_i = -\left(\frac{q_i}{2} + \frac{1}{4}\sum_{j \ne i} Q_{ij}\right)
$$

Energia różni się od $f(x)$ tylko o stałą, więc minimum jest w tym samym miejscu. *(Sprawdzone numerycznie dla losowej macierzy 5×5.)*

> [!warning] "Unconstrained" = bez ograniczeń
> Ograniczenia typu "dokładnie jeden z tych x = 1" wpisuje się jako **kary**: $P\,(\sum x_i - 1)^2$. Dobór siły kary $P$ to jedno z najtrudniejszych zadań kompilatora. Za mała kara daje niepoprawne rozwiązania, za duża "spłaszcza" resztę problemu. Patrz [[02 Kompilator (PC)]].

---

## 4. Przykład: Max-Cut

Podziel wierzchołki grafu na dwie grupy tak, żeby **suma wag krawędzi między grupami** była jak największa.

$$
J_{ij} = -w_{ij}, \quad h_i = 0, \qquad \text{cięcie} = \frac{\sum w - E}{2}
$$

Najniższa energia = największe cięcie. To **benchmark nr 1** dla maszyn Isinga (zbiór G-set), patrz [[06 Zastosowania - optymalizacja]].

---

## 5. Przykład: bramka AND w "invertible logic"

Sieć 3 p-bitów (A, B, C), w której **najniższą energię mają dokładnie 4 poprawne wiersze tabeli prawdy** C = A AND B:

$$
h = (1,\ 1,\ -2), \qquad J = \begin{pmatrix} 0 & -1 & 2 \\ -1 & 0 & 2 \\ 2 & 2 & 0 \end{pmatrix}
$$

| A | B | C | E | |
| --- | --- | --- | --- | --- |
| −1 | −1 | −1 | **−3** | ✅ poprawny |
| −1 | +1 | −1 | **−3** | ✅ |
| +1 | −1 | −1 | **−3** | ✅ |
| +1 | +1 | +1 | **−3** | ✅ |
| −1 | +1 | +1 | +1 | ❌ |
| +1 | −1 | +1 | +1 | ❌ |
| +1 | +1 | −1 | +1 | ❌ |
| −1 | −1 | +1 | +9 | ❌ |

Przy β = 1 sieć jest w poprawnym stanie przez **98,6%** czasu, a przy β = 2 przez **99,97%**.

**Dlaczego "invertible" (odwracalna):** jeśli **przypniesz** C = +1 (bardzo duże $h_C$), sieć sama znajdzie A = B = +1. Logika działa **w obie strony**. Przypinając wyjście **układu mnożącego**, dostajesz czynniki liczby, czyli faktoryzację. Patrz [[07 Zastosowania - cyberbezpieczeństwo]].

---

## 6. Model referencyjny (Python, bez bibliotek)

Ten kod to **wzorzec dla całego projektu**: każdy wynik z FPGA porównujemy z nim.

```python
import math, random
from collections import Counter

def gibbs(J, h, beta, sweeps, seed=0):
    """Sekwencyjne próbkowanie Gibbsa siecią p-bitów. J symetryczna, przekątna = 0."""
    rnd = random.Random(seed)
    n = len(h)
    m = [rnd.choice((-1, 1)) for _ in range(n)]
    samples = []
    for _ in range(sweeps):
        for i in range(n):                                  # jeden p-bit naraz
            I = h[i] + sum(J[i][j] * m[j] for j in range(n))  # pole lokalne
            r = rnd.uniform(-1, 1)                          # liczba losowa
            m[i] = 1 if math.tanh(beta * I) > r else -1
        samples.append(tuple(m))
    return samples

# bramka AND: C = A AND B   (kolejność p-bitów: A, B, C)
J = [[ 0, -1, 2],
     [-1,  0, 2],
     [ 2,  2, 0]]
h = [1, 1, -2]

N = 100_000
counts = Counter(gibbs(J, h, beta=1.0, sweeps=N))
for state, c in sorted(counts.items(), key=lambda kv: -kv[1]):
    print(state, f"{c / N:.4f}")
```

**Wynik (sprawdzony):**
```
(-1, 1, -1) 0.2479     ┐
(1, -1, -1) 0.2470     │ 4 poprawne stany, dokładnie z teorii: 0.2466 każdy
(-1, -1, -1) 0.2455    │
(1, 1, 1) 0.2455       ┘
(-1, 1, 1) 0.0048      ┐ błędne stany o E = +1, dokładnie z teorii: 0.0045
(1, 1, -1) 0.0047      │
(1, -1, 1) 0.0047      ┘
```

---

## 7. Zasady, których nie wolno łamać (dla zespołu FPGA)

> [!danger] Poprawność próbkowania
> 1. **Połączonych p-bitów nie aktualizuj jednocześnie.** Rozkład przestanie być Boltzmannem, a sieć może oscylować. Równolegle tylko p-bity z różnych "kolorów" grafu, patrz [[04 FPGA - macierz p-bitów]].
> 2. **Każda aktualizacja potrzebuje świeżej, niezależnej liczby losowej** $r$. Te same lub skorelowane bity losowe w sąsiednich p-bitach psują wyniki.
> 3. **J musi być symetryczna** ($J_{ij} = J_{ji}$), z zerową przekątną.

---

## Literatura (punkt startowy)
- Camsari, Faria, Sutton, Datta, *Stochastic p-bits for invertible logic*, Physical Review X (2017): p-bity i invertible logic
- Borders i in., *Integer factorization using stochastic magnetic tunnel junctions*, Nature (2019): faktoryzacja małych liczb na sprzętowych p-bitach
- Aadit i in., *Massively parallel probabilistic computing with sparse Ising machines*, Nature Electronics (2022): p-komputer na FPGA, kolorowanie grafu
- Hasło do szukania: **"p-bits", "probabilistic computing", "Ising machine"**
