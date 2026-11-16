---
tags: [p-komputer, trng, analog, losowość, bezpieczeństwo]
warstwa: 4
---
# 05 TRNG: generator prawdziwie losowego szumu

Wróć: [[00 Przegląd projektu]]

> [!abstract] Rola
> Dostarcza **prawdziwej (fizycznej) losowości** z szumu lawinowego diody Zenera. W p-komputerze jako **źródło ziaren** dla generatorów w FPGA. W cyberbezpieczeństwie jako **źródło entropii** (klucze kryptograficzne).

---

## Najważniejsza poprawka planu

> [!warning] TRNG nie może bezpośrednio sterować p-bitami
> | | Przepustowość |
> | --- | --- |
> | TRNG na diodzie Zenera (realnie, po obróbce) | **rzędu Mbit/s** |
> | Macierz p-bitów, sekwencyjnie, 50 MHz | **~500 Mbit/s** |
> | Macierz p-bitów, 256 jednostek równolegle | **~100 Gbit/s** |
>
> Brakuje **2–5 rzędów wielkości**. Gdyby p-bity czekały na TRNG, p-komputer byłby wolniejszy niż zwykły procesor.

### Jak połączyć TRNG z p-bitami

| Wariant | Opis | Ocena |
| --- | --- | --- |
| **1. Ziarna** | TRNG ustawia stan początkowy generatorów PRNG (xoshiro) w FPGA | ✅ minimum, proste |
| **2. Ziarna + ciągłe odświeżanie** | bity z TRNG stale "domieszane" (XOR) do stanu PRNG | ✅✅ **rekomendacja**: prędkość PRNG + stały dopływ prawdziwej entropii |
| 3. **Analogowy p-bit** | szum bezpośrednio w komparatorze = fizyczny p-bit | 🔬 ciekawy **eksperyment** dla kilku p-bitów, nie skaluje się (niżej) |
| 4. TRNG bezpośrednio do każdego p-bitu | | ❌ za wolne |

> [!question] Czy do optymalizacji w ogóle potrzeba prawdziwej losowości?
> Szczerze: **nie**. Dobry PRNG (xoshiro) wystarcza i tak robią to publikowane p-komputery na FPGA. Prawdziwa losowość ma znaczenie:
> - w **cyberbezpieczeństwie** (klucze, nieprzewidywalność): [[07 Zastosowania - cyberbezpieczeństwo]]
> - jako **wyróżnik** projektu i fundament przyszłego analogowego/scalonego p-bitu
>
> Warto to uczciwie mówić w prezentacjach projektu.

---

## Jak działa TRNG na diodzie Zenera

Dioda spolaryzowana zaporowo **powyżej ~6 V** pracuje w **przebiciu lawinowym**, które jest naturalnie zaszumione (losowe "lawiny" nośników). Ten szum ma pochodzenie kwantowo-termiczne i jest nieprzewidywalny.

```
 zasilanie     ┌──────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌───────────┐
 12–24 V ─────►│ dioda w  │──►│wzmacniacz├─►│ komparator│─►│ próbko-  │──►│ testy     │──►│ kondycjo- │──► wyjście
 (filtrowane)  │ przebiciu│   │ + sprzęż.│   │ (albo ADC)│  │ wanie    │   │ zdrowia   │   │ nowanie   │
               │ lawinowym│   │ AC       │   └──────────┘   │ (2 rej.) │   │ SP 800-90B│   │ (np. SHA) │
               └──────────┘   └─────────┘                   └──────────┘   └───────────┘   └───────────┘
                  analog (PCB)                               cyfrowo (FPGA albo MCU)
```

| Etap | Po co |
| --- | --- |
| Wzmacniacz | szum to miliwolty, a potrzeba poziomów logicznych |
| Sprzężenie AC | usuwa składową stałą, która dryfuje z temperaturą |
| Komparator | zamiana na bity 0/1 |
| Próbkowanie | zegar z FPGA/MCU. **Za szybkie próbkowanie = skorelowane bity** (ograniczone pasmo wzmacniacza) |
| Usuwanie nierównowagi | np. **von Neumann**: pary 01 → 0, 10 → 1, 00/11 odrzucić. Usuwa przewagę jedynek/zer kosztem ≥ 75% bitów |
| **Testy zdrowia** | ciągłe wykrywanie awarii (np. zablokowany komparator daje same jedynki) |
| Kondycjonowanie | "zagęszczenie" entropii funkcją skrótu (SHA-256) dla zastosowań kryptograficznych |

---

## Typowe problemy (i ataki)

| Problem | Skutek | Środek zaradczy |
| --- | --- | --- |
| Nierównowaga 0/1 | bity nie są 50/50 | von Neumann, kondycjonowanie |
| Autokorelacja | kolejne bity zależne | wolniejsze próbkowanie, XOR wielu źródeł |
| Dryf z temperaturą | zmienia się próg i amplituda | sprzężenie AC, automatyczny próg |
| Zakłócenia z zasilania / EMI | **przewidywalny** sygnał miesza się z szumem | filtrowanie zasilania, ekranowanie, dobry layout PCB |
| **Atak przez wstrzyknięcie sygnału** (zasilanie, pole EM) | napastnik "narzuca" bity | testy zdrowia, monitorowanie, ekran (ważne dla zastosowań bezpieczeństwa) |
| Starzenie diody | spadek amplitudy szumu | testy zdrowia wykrywają |

---

## Jak sprawdzić, czy TRNG jest dobry

> [!danger] Przejście testów statystycznych ≠ prawdziwa losowość
> Zwykły PRNG też przechodzi testy statystyczne! Testy wykrywają **błędy**, ale nie **dowodzą** entropii. Do tego potrzeba **modelu fizycznego źródła** i **estymacji min-entropii**.

| Narzędzie / standard | Do czego |
| --- | --- |
| **NIST SP 800-90B** | standard dla **źródeł entropii**: testy zdrowia (Repetition Count Test, Adaptive Proportion Test) i estymacja min-entropii. Narzędzie NIST `SP800-90B_EntropyAssessment` (GitHub, usnistgov) |
| NIST SP 800-22 | zestaw testów statystycznych (STS) |
| **dieharder**, **PractRand**, TestU01 | testy statystyczne (szybkie wykrywanie błędów) |
| BSI AIS 31 | niemiecki standard oceny TRNG (klasy PTG.1–3), istotny przy certyfikacji w UE |
| NIST SP 800-90A | jak z entropii zrobić bezpieczny generator (DRBG) |

**Punkt odniesienia:** sprzętowy RNG w STM32 ([[03 Mikrokontroler#Który mikrokontroler?]]) i generatory na oscylatorach pierścieniowych w samym FPGA.

**Warto przejrzeć otwarte projekty TRNG:** *OneRNG* (szum diody lawinowej) i *Infinite Noise TRNG*. Mają otwarte schematy, ich problemy są dobrze opisane, i pomogą uniknąć typowych błędów.

---

## Eksperyment: analogowy p-bit

Komparator, do którego wchodzi **sygnał sterujący $I$ (z przetwornika DAC) + szum**, sam jest p-bitem:

$$
P(\text{wyjście}=1) = \Phi\!\left(\frac{I}{\sigma}\right)
$$

gdzie Φ to dystrybuanta rozkładu normalnego, kształtem bardzo podobna do $(1+\tanh)/2$. **Amplituda szumu σ działa jak temperatura** (β ~ 1/σ).

| Za | Przeciw |
| --- | --- |
| prawdziwy, fizyczny p-bit, mocny wyróżnik projektu | każdy p-bit potrzebuje **własnego, niezależnego** źródła szumu + DAC + komparatora |
| pierwszy krok w stronę układu scalonego | skaluje się do **kilku–kilkunastu** p-bitów, a nie tysięcy |
| | wspólne zakłócenia (zasilanie) **korelują** kanały, co psuje próbkowanie |

**Propozycja:** demonstracja **hybrydowa**: 3–4 analogowe p-bity (np. bramka AND z [[01 Teoria - p-bity, Ising, QUBO]], punkt 5) sterowane przez FPGA, z porównaniem do wersji cyfrowej.

---

## "Układ scalony": PCB czy ASIC?

> [!question] Do ustalenia w zespole
> - **Prototyp: płytka PCB z elementów dyskretnych** (KiCad). Tania, szybka do poprawiania, w zasięgu zespołu.
> - **Własny układ scalony (ASIC)** to ogromny krok (projekt, weryfikacja, produkcja, miesiące czekania). Istnieją tańsze programy "współdzielonych płytek krzemowych" (MPW / shuttle, np. TinyTapeout). **Sprawdzić aktualną dostępność i koszty**, a traktować jako cel długoterminowy, nie część prototypu.
> - Badawczy "prawdziwy" p-bit w krzemie to np. **stochastyczne złącza MTJ** (patrz literatura w [[01 Teoria - p-bity, Ising, QUBO]]).

---

## Integracja z systemem (prototyp)

**Rekomendacja:** komparator TRNG → **wejście MCU** (GPIO albo ADC) → testy zdrowia w firmware (C) → ziarna przez SPI do FPGA.
- Ziarna są potrzebne rzadko, więc przepustowość SPI nie gra roli.
- Testy zdrowia łatwiej napisać i debugować w C niż w Verilogu.
- Później (wariant 2, ciągłe odświeżanie): komparator bezpośrednio do pinu FPGA, przez 2 rejestry synchronizujące.

Interfejs: [[09 Zespół, interfejsy i plan#Kontrakt C - losowość (TRNG ↔ MCU ↔ FPGA)]]

> [!warning] Bezpieczeństwo elektryczne
> Napięcia 12–24 V przy małych prądach są bezpieczne, ale **osobna, filtrowana linia zasilania** dla części analogowej, wspólna masa i nie podawać 12 V na piny 3,3 V FPGA/MCU (zawsze przez komparator z wyjściem 3,3 V).
