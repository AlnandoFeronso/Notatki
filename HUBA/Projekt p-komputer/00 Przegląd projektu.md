---
tags: [p-komputer, przegląd, moc]
zespół: 5 osób
status: koncepcja / prototyp
platforma_prototypu: Terasic DE10-Lite (Intel MAX 10)
---
# Komputer probabilistyczny: przegląd projektu

> [!abstract] W jednym zdaniu
> Budujemy **komputer probabilistyczny (p-komputer)**: sprzęt, który rozwiązuje problemy **optymalizacyjne** i z obszaru **cyberbezpieczeństwa**, zamieniając je na sieć **p-bitów** (losowych bitów połączonych wagami) i pozwalając jej "spłynąć" do najlepszego rozwiązania.

**Zespół:** 5 osób · **Prototyp:** DE10-Lite (MAX 10). Docelowo możemy kupić mocniejszą płytkę, patrz [[08 Sprzęt - prototyp i płytka docelowa]].

---

## Architektura (pipeline)

```
┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
│  1. PC UŻYTKOWNIKA│     │ 2. MIKROKONTROLER │     │      3. FPGA      │     │  4. ŹRÓDŁO SZUMU  │
│                   │ USB │   (np. STM32)     │ SPI │                   │     │   (TRNG, dioda    │
│ "kompilator":     │────►│                   │────►│ macierz p-bitów   │◄────│    Zenera)        │
│ problem biznesowy │     │ - komunikacja     │ FMC │ - wagi J, h       │     │                   │
│   → QUBO / Ising  │◄────│ - zarządzanie     │◄────│ - próbkowanie     │     │ prawdziwa         │
│   → wagi J, h     │     │   wagami          │     │   Gibbsa          │     │ losowość          │
│ wyniki → odpowiedź│     │ - harmonogram β   │     │ - statystyki      │     │                   │
└───────────────────┘     └───────────────────┘     └─────────┬─────────┘     └───────────────────┘
                                                              │
                                                     ┌────────▼────────┐
                                                     │ zewnętrzny RAM  │
                                                     │ (biblioteka wag)│
                                                     └─────────────────┘
```

| Warstwa | Notatka | Technologia (propozycja) |
| --- | --- | --- |
| Teoria wspólna dla wszystkich | [[01 Teoria - p-bity, Ising, QUBO]] | — |
| 1. Kompilator na PC | [[02 Kompilator (PC)]] | Python |
| 2. Mikrokontroler | [[03 Mikrokontroler]] | STM32 (F4/H7), C albo Rust |
| 3. FPGA | [[04 FPGA - macierz p-bitów]] | Verilog / SystemVerilog |
| 4. Źródło szumu | [[05 TRNG - generator szumu]] | analog + PCB (KiCad) |
| Zastosowania | [[06 Zastosowania - optymalizacja]], [[07 Zastosowania - cyberbezpieczeństwo]] | — |
| Sprzęt | [[08 Sprzęt - prototyp i płytka docelowa]] | DE10-Lite → ? |
| Organizacja | [[09 Zespół, interfejsy i plan]] | git, cocotb |
| Krytyczna analiza | [[10 Ryzyka i otwarte pytania]] | — |

---

## Najważniejsze wnioski z analizy

> [!success] Co jest dobre w tym planie
> - **Podział na 4 warstwy jest rozsądny** i dobrze pasuje do 5 osób: każdy ma swoją warstwę, a granice między nimi da się opisać kontraktami ([[09 Zespół, interfejsy i plan]]).
> - **FPGA to sprawdzona platforma dla p-bitów.** Grupy badawcze (Purdue, UCSB: Camsari, Datta i in.) budowały p-komputery właśnie na FPGA.
> - **Optymalizacja on-premise** (bez chmury) to realna nisza, zwłaszcza dla danych wrażliwych w cyberbezpieczeństwie.

> [!warning] Co trzeba zmienić albo przemyśleć
> 1. **TRNG nie nakarmi p-bitów bezpośrednio.** Dioda Zenera daje rzędu **Mbit/s** losowości, a macierz p-bitów zużywa **setki Mbit/s do dziesiątek Gbit/s**. Rozwiązanie: TRNG **zasila ziarnem** szybkie generatory pseudolosowe w FPGA. Szczegóły w [[05 TRNG - generator szumu]].
> 2. **Wagi z zewnętrznego RAM-u przy każdym kroku byłyby za wolne.** RAM to biblioteka problemów, a do liczenia wagi trzeba skopiować do pamięci wewnętrznej FPGA. Szczegóły w [[04 FPGA - macierz p-bitów]].
> 3. **Mikrokontroler nie może przekazywać każdej próbki.** FPGA samo liczy statystyki i najlepsze znalezione rozwiązanie, a MCU odczytuje tylko wyniki.
> 4. **Najtrudniejszą częścią jest kompilator, nie sprzęt.** Zamiana "problemu biznesowego" na dobre QUBO (kary za ograniczenia, skalowanie, kwantyzacja wag) decyduje o jakości wyników.
> 5. **Przewaga nad klasycznymi metodami nie jest automatyczna.** Wyżarzanie symulowane na GPU i solwery typu Gurobi czy OR-Tools to silna konkurencja. Potrzebujemy **benchmarków od pierwszego miesiąca**.
> 6. **Cyberbezpieczeństwo: łamanie kryptografii (np. faktoryzacja RSA) to tylko demonstracja na małych liczbach.** Realne zastosowania to optymalizacja *w* bezpieczeństwie (rozmieszczenie sensorów, utwardzanie sieci, priorytety łatek) i TRNG jako osobny produkt. Szczegóły w [[07 Zastosowania - cyberbezpieczeństwo]].

---

## Kamienie milowe (skrót)

| # | Kamień milowy | Gotowe, gdy… |
| --- | --- | --- |
| M0 | Model referencyjny w Pythonie | bramka AND i mały Max-Cut zgadzają się z dokładnym rozkładem Boltzmanna |
| M1 | p-bit + mała sieć na FPGA | wyniki z płytki = wyniki z modelu |
| M2 | Pełny łańcuch PC → MCU → FPGA | problem z PC rozwiązany na płytce i odesłany z powrotem |
| M3 | Benchmarki | porównanie z wyżarzaniem na CPU dla zbioru G-set (Max-Cut) |
| M4 | TRNG podłączony + testy statystyczne | ziarna z TRNG, przechodzi testy zdrowia NIST SP 800-90B |
| M5 | Pierwsze zastosowanie end-to-end | konkretny problem (np. rozmieszczenie sensorów IDS) od opisu do odpowiedzi |

Pełny plan: [[09 Zespół, interfejsy i plan]].

---

## Powiązane
- Nauka Veriloga: [[O co chodzi]], [[Podstawy składni Verilog]], [[W jaki sposób otworzyć i używać projektu verilog]]
