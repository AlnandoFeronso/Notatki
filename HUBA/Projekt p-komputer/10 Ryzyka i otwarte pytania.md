---
tags: [p-komputer, ryzyka, decyzje]
---
# 10 Ryzyka i otwarte pytania

Wróć: [[00 Przegląd projektu]]

---

## Ryzyka (od najpoważniejszych)

| # | Ryzyko | Prawdop. | Wpływ | Jak ograniczyć |
| --- | --- | --- | --- | --- |
| 1 | **Brak przewagi nad klasycznymi metodami** (wyżarzanie na CPU/GPU, Gurobi) | wysokie | wysoki | benchmarki **od M3**, uczciwe metryki (czas, energia, jakość); wybrać argument, którego da się bronić (energia, on-premise), patrz [[06 Zastosowania - optymalizacja]] |
| 2 | **Słabe kodowanie problemów** (kary, kwantyzacja wag) daje złe wyniki mimo działającego sprzętu | wysokie | wysoki | model referencyjny z pełną precyzją kontra skwantowany; automatyczne strojenie kar ([[02 Kompilator (PC)]]) |
| 3 | **Za ambitny zakres** na 5 osób (4 warstwy + analog + zastosowania) | średnie | wysoki | kamienie milowe z jasnym "gotowe, gdy…"; **najpierw pionowy przekrój przez cały system** (M2) na małym N, potem skalowanie |
| 4 | **Niepoprawne próbkowanie w FPGA** (aktualizacja połączonych p-bitów, skorelowane RNG) wygląda jak działanie, ale daje zły rozkład | średnie | wysoki | testy rozkładu (χ²) vs model w CI; zasady z [[01 Teoria - p-bity, Ising, QUBO#7. Zasady, których nie wolno łamać (dla zespołu FPGA)]] |
| 5 | **Oczekiwanie, że TRNG napędza p-bity** | (już wykryte) | średni | architektura "ziarna + odświeżanie", patrz [[05 TRNG - generator szumu]] |
| 6 | **Wiarygodność przy "łamaniu kryptografii"** | średnie | wysoki | komunikować jako demonstrację, nie zagrożenie dla RSA ([[07 Zastosowania - cyberbezpieczeństwo]]) |
| 7 | **Integracja na końcu** ("wszystko działa osobno, razem nie") | wysokie | średni | kontrakty v0 od pierwszego tygodnia, wczesny M2, rotujący integrator ([[09 Zespół, interfejsy i plan]]) |
| 8 | **Krzywa nauki** (Verilog, STM32, analog: każdy zaczyna coś nowego) | wysokie | średni | proste pierwsze zadania, praca w parach, notatki w vaulcie |
| 9 | **Zakłócenia w TRNG** (zasilanie, EMI) dają przewidywalne bity | średnie | średni | testy zdrowia, osobne zasilanie, porównanie z RNG w STM32 |
| 10 | **Zablokowanie na jednym producencie FPGA** | niskie | średni | przenośny rdzeń, `boards/` osobno ([[08 Sprzęt - prototyp i płytka docelowa]]) |

---

## Otwarte pytania na spotkanie zespołu

### Cel i odbiorca
- [ ] **Co jest celem projektu?** Konkurs / praca dyplomowa / start-up / publikacja? To zmienia priorytety (demo kontra benchmarki kontra produkt).
- [ ] **Kto jest użytkownikiem "kompilatora"?** Analityk biznesowy, specjalista bezpieczeństwa, programista?
- [ ] **Jaki jest pierwszy problem flagowy?** Proponuję: rozmieszczenie sensorów IDS ([[07 Zastosowania - cyberbezpieczeństwo]]).

### Architektura
- [ ] Czy zostajemy przy osobnym MCU, czy docelowo SoC FPGA? ([[03 Mikrokontroler#Czy MCU w ogóle jest potrzebny? (analiza alternatyw)]])
- [ ] Gęsta macierz czy graf rzadki jako główny tryb? (zależy od problemu flagowego)
- [ ] Gdzie żyje biblioteka wag: SDRAM przy FPGA czy pamięć przy MCU?
- [ ] Który STM32 (F4 kontra H7, czy potrzebny Ethernet / USB HS)?

### TRNG
- [ ] "Układ scalony" oznacza PCB z elementów dyskretnych czy docelowo własny ASIC?
- [ ] Czy robimy eksperyment z **analogowym p-bitem** (3–4 p-bity)?
- [ ] Czy TRNG ma być osobnym produktem bezpieczeństwa (wtedy certyfikacja SP 800-90B / AIS 31)?

### Organizacja
- [ ] Podział ról: kto bierze którą z 5 ról ([[09 Zespół, interfejsy i plan#Proponowany podział ról]])?
- [ ] Gdzie repozytorium (GitHub / GitLab) i jak dzielimy notatki (wspólny vault? Obsidian Sync / git)?
- [ ] Budżet na sprzęt (płytka docelowa, STM32, części TRNG)?

---

## Kiedy uznamy, że projekt "działa"

> [!success] Minimalny sukces (prototyp)
> Problem opisany na PC → rozwiązany na p-bitach w FPGA → wynik z powrotem na PC, z **poprawnym rozkładem** (zgodnym z modelem) i z **ziarnami z własnego TRNG**.

> [!success] Pełny sukces
> Dla wybranej klasy problemów p-komputer jest **lepszy w udokumentowanej metryce** (energia, czas przy tej samej jakości, albo możliwość pracy on-premise przy porównywalnej jakości) niż klasyczna alternatywa, na przykładzie z cyberbezpieczeństwa.
