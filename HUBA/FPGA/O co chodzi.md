---
tags: [fpga, verilog, podstawy]
---
# O co chodzi

No to jak każdy chyba już wie, FPGA to tak naprawdę zestaw bramek, których zachowanie definiujemy w Verilogu. Jedną z moim zdaniem ważniejszych konsekwencji tego jest fakt, że kod w Verilogu nie jest sekwencyjny, o czym łatwo zapomnieć w porównaniu z tradycyjnymi językami programowania.

> [!summary] Kolejność nauki
> 1. **Ta notatka**: czym jest FPGA i jak o nim myśleć
> 2. [[Podstawy składni Verilog]]: jak pisać kod, z ćwiczeniami na płytkę
> 3. [[W jaki sposób otworzyć i używać projektu verilog]]: jak uruchomić kod na DE10-Lite

---

## Procesor a FPGA

|                   | Procesor (PC, ESP32)                          | FPGA                                                 |
| ----------------- | --------------------------------------------- | ---------------------------------------------------- |
| Co robi           | wykonuje **program** instrukcja po instrukcji | staje się **układem elektronicznym**, który opiszesz |
| Kod to            | lista kroków                                  | opis połączeń: bramek, przewodów i pamięci           |
| Ile rzeczy naraz  | jedna (albo kilka na kilku rdzeniach)         | **wszystko naraz**, tysiące układów równolegle       |
| Język             | C, Rust, Python                               | Verilog, VHDL                                        |
| Co znaczy "wgrać" | skopiować program do pamięci                  | **połączyć** elementy w środku chipu                 |

> [!tip] Najlepsza analogia
> FPGA to **pusta płytka stykowa** z 50 000 elementów, na której nic nie jest jeszcze połączone.
> Kod w Verilogu to **schemat połączeń**. Quartus czyta ten schemat i "lutuje" elementy w środku chipu.

---

## Kod nie jest sekwencyjny

W C program robi jedno, potem drugie:

```c
a = 1;      // najpierw to
b = a + 1;  // potem to
```

W Verilogu każda linijka to **kawałek sprzętu, który istnieje cały czas**. Ten kod:

```verilog
assign b = ~a;       // b to zawsze odwrotność a
assign a = SW[0];    // a to zawsze przełącznik 0
assign LEDR[0] = b;  // dioda 0 pokazuje b
```

działa **tak samo w dowolnej kolejności linijek**. To nie są kroki, tylko trzy przewody i jedna bramka NOT, wszystkie podłączone jednocześnie:

```
SW[0] ──── a ──── [NOT] ──── b ──── LEDR[0]
```

> [!warning] Najczęstszy błąd początkujących
> Myślenie "najpierw to się wykona, potem tamto". W sprzęcie nic się nie "wykonuje", **wszystko działa naraz, cały czas**.
> Jedyny sposób na "kolejność" w czasie to **zegar** (patrz niżej).

---

## Co jest w środku FPGA

| Element                   | Co robi                                                                       |
| ------------------------- | ----------------------------------------------------------------------------- |
| **LUT** (Look-Up Table)   | malutka tabelka prawdy, która może udawać **dowolną** bramkę z kilkoma wejściami (AND, OR, XOR…) |
| **Przerzutnik** (flip-flop, rejestr) | pamięta **1 bit**, zmienia wartość tylko przy tyknięciu zegara    |
| **Połączenia** (routing)  | programowalne "kable" między elementami                                       |
| **Piny I/O**              | wyjścia na zewnątrz chipu: diody, przełączniki, złącza                        |
| Bloki specjalne           | pamięć RAM, mnożarki, PLL (generatory zegara), przetwornik ADC               |

LUT + przerzutnik razem tworzą **element logiczny** (LE). Twój układ **MAX 10 10M50** ma ich **49 760**.

> [!example] Ile to zajmuje w praktyce
> - Przełączniki → diody: **1** element (to głównie same przewody)
> - Licznik z mrugającymi diodami (`blink`): **56** elementów
> - Cały prosty procesor: kilka tysięcy
>
> Masz więc mnóstwo miejsca na eksperymenty.

---

## Dwa rodzaje układów

Każdy projekt cyfrowy składa się z dwóch rodzajów układów:

### 1. Kombinacyjny (bez pamięci)
Wyjście zależy **tylko od tego, co jest na wejściu teraz**. Same bramki i przewody.
- Przykład: dioda świeci, gdy oba przyciski są wciśnięte (bramka AND)
- Przykład: suma dwóch liczb z przełączników

### 2. Sekwencyjny (z pamięcią i zegarem)
Układ **pamięta** stan i zmienia go **przy każdym tyknięciu zegara**.
- Przykład: licznik, który co 1/4 sekundy dodaje 1
- Przykład: "światełko", które przesuwa się po diodach

> [!info] Zegar
> **Zegar** to sygnał, który zmienia się 0 → 1 → 0 → 1… bardzo szybko. Na DE10-Lite to **50 MHz**, czyli 50 milionów tyknięć na sekundę.
> Przy każdym tyknięciu **wszystkie** rejestry w układzie zapisują nowe wartości **jednocześnie**.
> To jedyny sposób, żeby coś działo się "po kolei": krok 1 przy tyknięciu 1, krok 2 przy tyknięciu 2 itd.

Szczegóły, jak pisać oba rodzaje, są w [[Podstawy składni Verilog]].

---

## Od kodu do świecącej diody

```
 blink.v  ──►  Synteza  ──►  Fitter  ──►  Assembler  ──►  Programmer  ──►  płytka
 (Verilog)    (kod →        (gdzie w     (plik .sof     (wysłanie
               bramki)       chipie?)     do wgrania)    przez USB)
```

| Etap           | Co robi                                                                | Polecenie                |
| -------------- | ---------------------------------------------------------------------- | ------------------------ |
| **Symulacja**  | testuje układ **na komputerze**, bez płytki                            | `make sim`               |
| **Synteza**    | zamienia Verilog na bramki i rejestry                                  | `make` (wszystkie 3 etapy) |
| **Fitter**     | rozmieszcza je w konkretnych miejscach chipu i łączy z pinami          | ↑                        |
| **Assembler**  | tworzy plik `.sof` gotowy do wgrania                                   | ↑                        |
| **Programmer** | wysyła plik przez USB do FPGA                                          | `make prog`              |

Jak tego używać krok po kroku: [[W jaki sposób otworzyć i używać projektu verilog]].

---

## Moja płytka: Terasic DE10-Lite

| Co                 | Nazwa w kodzie              | Wartość                                        |
| ------------------ | --------------------------- | ---------------------------------------------- |
| Zegar 50 MHz       | `MAX10_CLK1_50`             | tyka 50 000 000 razy/s                         |
| 10 przełączników   | `SW[0]` … `SW[9]`           | góra = `1`, dół = `0`                          |
| 2 przyciski        | `KEY[0]`, `KEY[1]`          | ⚠️ **wciśnięty = `0`**, puszczony = `1`        |
| 10 czerwonych diod | `LEDR[0]` … `LEDR[9]`       | `1` = świeci                                   |
| Układ FPGA         | (w `blink.qsf`)             | Intel MAX 10, `10M50DAF484C7G`                 |

Na płytce jest też więcej rzeczy na później: 6 wyświetlaczy 7-segmentowych, wyjście VGA, akcelerometr, pamięć SDRAM i złącze GPIO / Arduino.

> [!note] Skąd kod wie, która dioda to `LEDR[0]`?
> Z pliku `blink.qsf`, który przypisuje nazwy do fizycznych pinów chipu:
> `set_location_assignment PIN_A8 -to LEDR[0]`
> Nie trzeba go zmieniać, dopóki używasz tych samych nazw.

---

## Słowniczek

| Pojęcie             | Znaczenie                                                                  |
| ------------------- | -------------------------------------------------------------------------- |
| **FPGA**            | Field-Programmable Gate Array: chip, który można "przelutować" kodem       |
| **HDL**             | Hardware Description Language: język **opisu sprzętu** (Verilog, VHDL)     |
| **RTL**             | Register Transfer Level: "prawdziwy" kod układu (folder `rtl/`)            |
| **Testbench**       | kod, który testuje układ w symulacji (folder `sim/`), nigdy nie trafia na chip |
| **Moduł**           | "czarna skrzynka" z wejściami i wyjściami, podstawowy klocek w Verilogu    |
| **Synteza**         | zamiana kodu na bramki                                                     |
| **Bit / wektor**    | jeden przewód / kilka przewodów razem (np. `[9:0]` = 10 przewodów)         |
| **Rejestr**         | pamięć na bity, aktualizowana przy tyknięciu zegara                        |
| **Zbocze narastające** (`posedge`) | moment, gdy sygnał zmienia się z 0 na 1                     |
| **Active-low**      | sygnał "działa", gdy jest `0` (np. przyciski `KEY`)                        |
| **Timing / Fmax**   | jak szybki zegar wytrzyma układ (blink: ~233 MHz, płytka daje 50 MHz)      |
