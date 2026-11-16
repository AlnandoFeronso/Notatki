---
tags: [fpga, verilog, quartus, workflow]
---
# W jaki sposób otworzyć i używać projektu verilog

> [!summary] W skrócie
> ```bash
> cd ~/Documents/projects/fpga/blink
> nvim rtl/blink.v   # 1. edytuj
> make sim           # 2. przetestuj na komputerze
> make prog          # 3. skompiluj i wgraj na płytkę
> ```
> Teoria: [[O co chodzi]] · Składnia i ćwiczenia: [[Podstawy składni Verilog]]

---

## Co jest gdzie

Projekt: `~/Documents/projects/fpga/blink`

```
blink/
├── rtl/
│   └── blink.v        ← TWÓJ UKŁAD: tu piszesz kod, to trafia na chip
├── sim/
│   └── blink_tb.v     ← testbench: "sztuczna płytka" do symulacji
├── blink.qpf          ← plik projektu Quartusa (otwierasz go w GUI)
├── blink.qsf          ← ustawienia: model chipu, przypisanie pinów, lista plików .v
├── blink.sdc          ← informacja dla Quartusa: zegar ma 50 MHz
├── Makefile           ← skróty: make sim, make prog…
├── .gitignore         ← co git ma ignorować (pliki generowane)
│
├── output_files/      ┐
├── db/                ├ generowane przez Quartusa, można usunąć (make clean)
└── incremental_db/    ┘
```

> [!tip] Na co dzień edytujesz tylko `rtl/*.v`
> Reszta jest już ustawiona pod DE10-Lite. `sim/` ruszasz, gdy chcesz pisać własne testy.

### Narzędzia, które masz zainstalowane

| Narzędzie            | Do czego                                           | Gdzie                        |
| -------------------- | -------------------------------------------------- | ---------------------------- |
| **Quartus Prime Lite 25.1** | kompilacja i wgrywanie na MAX 10            | `~/altera_lite/25.1`         |
| **Icarus Verilog** (`iverilog`) | symulacja (`make sim`)                  | system                       |
| **GTKWave**          | oglądanie przebiegów sygnałów (`make wave`)        | system                       |
| **Verilator**        | szybkie sprawdzanie kodu (`make lint`)             | system                       |
| **verible**          | podpowiedzi i podkreślanie błędów w Neovimie       | Mason (Neovim)               |

---

## Przed pierwszym użyciem

1. **Otwórz nowy terminal.** Ścieżka do Quartusa jest ustawiona w `~/.bashrc_custom` i ładuje się przy starcie terminala.
2. **Podłącz płytkę** kablem USB do gniazda *USB Blaster* i włącz ją (świeci dioda zasilania).
3. Sprawdź, czy komputer ją widzi:
   ```bash
   lsusb | grep 09fb       # powinno pokazać: Altera Blaster
   jtagconfig              # powinno pokazać: USB-Blaster … 10M50DA
   ```

---

## Codzienna praca

| Polecenie    | Co robi                                                       | Czas      | Płytka potrzebna? |
| ------------ | ------------------------------------------------------------- | --------- | ----------------- |
| `make sim`   | symulacja testbencha, wypisuje `PASS` albo `FAIL`             | ~1 s      | ❌                |
| `make wave`  | symulacja + otwiera GTKWave z przebiegami                     | ~1 s      | ❌                |
| `make lint`  | szybkie sprawdzenie kodu Verilatorem (ostrzeżenia)            | ~1 s      | ❌                |
| `make`       | pełna kompilacja w Quartusie → `output_files/blink.sof`       | ~20 s     | ❌                |
| `make prog`  | kompiluje (jeśli trzeba) i **wgrywa do FPGA**                 | ~25 s     | ✅                |
| `make flash` | wgrywa **na stałe** do pamięci flash chipu                    | ~30 s     | ✅                |
| `make clean` | usuwa pliki generowane                                        | —         | ❌                |

> [!important] `make prog` vs `make flash`
> - **`make prog`**: projekt działa **do wyłączenia zasilania**. Szybkie, używaj do testów.
> - **`make flash`**: projekt zostaje w chipie **po wyłączeniu**. Płytka uruchomi go sama przy każdym włączeniu.
>
> ⚠️ `make flash` nie był jeszcze sprawdzony na mojej płytce. Przy pierwszym użyciu patrz, czy na końcu jest `0 errors`.

### Pętla pracy
```
  ┌──► edytuj rtl/blink.v ──► make sim ──► działa? ──nie──┐
  │                                         │ tak         │
  │                                         ▼             │
  │                                     make prog         │
  │                                         │             │
  └──────────── popraw ◄── na płytce źle? ◄─┘◄────────────┘
```

> [!tip] Jak czytać wynik `make`
> Quartus wypisuje **mnóstwo** tekstu. Interesuje cię tylko końcówka:
> ```
> Info: Quartus Prime Full Compilation was successful. 0 errors, 14 warnings
> ```
> - `0 errors` → jest dobrze. Ostrzeżenia typu *"input pin does not drive logic"* oznaczają tylko, że nie używasz wszystkich przełączników, i można je ignorować.
> - Przy błędzie szukaj linii zaczynającej się od `Error (`: jest tam nazwa pliku i numer linii.

---

## Praca w Neovimie

```bash
cd ~/Documents/projects/fpga/blink
nvim rtl/blink.v
```

Neovim jest skonfigurowany pod Veriloga (serwer **verible**): błędy są podkreślane **na bieżąco**, zanim w ogóle uruchomisz kompilację.

| Skrót                   | Co robi                                               |
| ----------------------- | ----------------------------------------------------- |
| `Ctrl-w d`              | pokaż pełny opis błędu pod kursorem                   |
| `]d` / `[d`             | skocz do następnego / poprzedniego błędu              |
| `Spacja q`              | lista wszystkich błędów w pliku                       |
| `Spacja f`              | sformatuj plik (ładne wcięcia)                        |
| `grd`                   | idź do definicji (np. modułu albo sygnału)            |
| `grn`                   | zmień nazwę sygnału we wszystkich miejscach           |
| `Spacja s f`            | szukaj pliku w projekcie                              |
| `Spacja s g`            | szukaj tekstu we wszystkich plikach                   |
| `\`                     | drzewo plików (Neo-tree)                              |
| `:!make sim`            | uruchom symulację bez wychodzenia z Neovima           |

> [!note] Inne pliki
> `.qsf` i `.sdc` mają kolorowanie składni jako Tcl. VHDL (`.vhd`) też jest obsługiwany, gdybyś kiedyś chciał spróbować.

---

## Symulacja i GTKWave

```bash
make sim     # wynik w terminalu: PASS: LEDR = 15
make wave    # otwiera okno GTKWave
```

**W GTKWave:**
1. Lewy górny panel (*SST*): rozwiń `blink_tb`, potem kliknij `dut` (to twój układ w środku testbencha)
2. Lewy dolny panel: pojawią się sygnały (`LEDR`, `tick_cnt`, `MAX10_CLK1_50`…)
3. Zaznacz je i kliknij **Append** (albo przeciągnij na prawą stronę)
4. **Zoom Fit** (ikona lupy z ramką albo `Ctrl+Alt+F`): pokaż całą symulację
5. Prawy klik na sygnale → **Data Format → Decimal**: liczby dziesiętnie zamiast hex

Zobaczysz, jak `tick_cnt` liczy 0…9 i wraca do 0, a `LEDR` rośnie o 1 przy każdym zawinięciu. To ten sam układ co na płytce, tylko testbench ustawia mniejsze liczby (`CLK_HZ(100)`), żeby symulacja była krótka.

> [!info] Testbench
> Testbench (`sim/blink_tb.v`) to "sztuczna płytka": generuje zegar, "naciska" przyciski i sprawdza wynik. Używa rzeczy działających **tylko w symulacji** (`#10`, `$display`, `initial`), dlatego trzymamy go osobno od `rtl/`. Więcej w [[Podstawy składni Verilog#13. Rzeczy tylko do symulacji]].

---

## Quartus GUI (opcjonalnie)

```bash
quartus ~/Documents/projects/fpga/blink/blink.qpf
```

Wszystko da się zrobić przez `make`, ale GUI ma kilka bardzo przydatnych rzeczy do nauki:

| Gdzie                                             | Co pokazuje                                          |
| ------------------------------------------------- | ---------------------------------------------------- |
| **Tools → Netlist Viewers → RTL Viewer**          | 🌟 twój kod narysowany jako **schemat układu**: bramki, rejestry, multipleksery |
| **Tools → Programmer**                            | graficzne wgrywanie (zamiast `make prog`)            |
| **Assignments → Pin Planner**                     | które sygnały są na których pinach                   |
| **Compilation Report → Flow Summary**             | ile elementów logicznych zużywa projekt              |
| **Compilation Report → Timing Analyzer → Fmax**   | jak szybki zegar wytrzyma twój układ                 |

> [!tip] RTL Viewer po każdym ćwiczeniu
> Najlepszy sposób, żeby zrozumieć, że Verilog = układ: napisz kod, `make`, otwórz RTL Viewer i zobacz, **co z niego powstało**. `if` zamienia się w multiplekser, `+` w sumator, a `always @(posedge…)` w rejestry.

Bez GUI też możesz sprawdzić zużycie:
```bash
cat output_files/blink.fit.summary
```

---

## Git, czyli jak nie zgubić działającej wersji

Projekt ma już repozytorium git (jeszcze bez żadnego commita). Zrób pierwszy:

```bash
cd ~/Documents/projects/fpga/blink
git add -A
git commit -m "Działający licznik blink"
```

| Polecenie                       | Co robi                                             |
| ------------------------------- | --------------------------------------------------- |
| `git status`                    | co się zmieniło od ostatniego commita               |
| `git diff`                      | pokaż dokładne zmiany                               |
| `git add -A && git commit -m "opis"` | zapisz obecny stan                             |
| `git restore rtl/blink.v`       | ⚠️ cofnij plik do ostatniego commita (zmiany przepadają!) |
| `git log --oneline`             | historia commitów                                   |

> [!tip] Przy ćwiczeniach
> Commit przed eksperymentem → bawisz się → `git restore rtl/blink.v` i masz z powrotem działający licznik.

---

## Nowy projekt z szablonu

Kopiujesz `blink` i zmieniasz nazwę wszędzie. Ustaw nazwę w pierwszej linijce, resztę wklej bez zmian:

```bash
NAZWA=swiatelko
cd ~/Documents/projects/fpga
cp -r blink $NAZWA && cd $NAZWA
make clean && rm -rf .git && git init
for f in blink.qpf blink.qsf blink.sdc rtl/blink.v sim/blink_tb.v; do mv "$f" "${f//blink/$NAZWA}"; done
sed -i "s/blink/$NAZWA/g" Makefile $NAZWA.qpf $NAZWA.qsf rtl/$NAZWA.v sim/${NAZWA}_tb.v
make sim
```

Powinno wypisać `PASS`. Teraz edytujesz `rtl/swiatelko.v`.

> [!warning] Nazwa projektu
> Tylko małe litery, cyfry i `_`. Bez spacji i polskich znaków (np. `swiatelko`, nie `światełko`).

---

## Dodanie nowego pliku .v

Gdy rozbijasz projekt na kilka modułów (np. `rtl/half_adder.v`):

1. Utwórz plik w `rtl/`, np. `nvim rtl/half_adder.v`
2. **Dopisz go do Quartusa**: otwórz `blink.qsf` i na samym końcu, w **nowej linii**, dodaj:
   ```tcl
   set_global_assignment -name VERILOG_FILE rtl/half_adder.v
   ```
3. `make sim` i `make lint` znajdą go same (Makefile bierze wszystkie `rtl/*.v`)

> [!warning] Nie dopisuj tej linii przez `echo >>`
> Quartus zapisuje `.qsf` bez znaku nowej linii na końcu, więc `echo … >> blink.qsf` sklei dwie linijki w jedną i kompilacja się wysypie. Dopisuj w Neovimie albo przez:
> `printf '\nset_global_assignment -name VERILOG_FILE rtl/half_adder.v\n' >> blink.qsf`

---

## Gdy coś nie działa

| Problem                                         | Rozwiązanie                                                                 |
| ----------------------------------------------- | --------------------------------------------------------------------------- |
| `quartus_sh: command not found`                 | otwórz **nowy** terminal (albo `source ~/.bashrc_custom`)                    |
| `No JTAG hardware available`                    | płytka włączona? Odłącz i podłącz USB. Dalej nic? → `pkill jtagd` i spróbuj ponownie |
| `lsusb` nie pokazuje `Altera Blaster`           | inny kabel: niektóre kable USB mają tylko zasilanie, bez danych              |
| Wgrało się, ale diody nic nie robią             | naciśnij **KEY0** (reset). Sprawdź, czy SW0 nie jest w górze (pauza w `blink`) |
| `Error (…)` przy `make`                         | przeczytaj linię `Error`: jest tam plik i numer linii. Neovim zwykle podkreśla to wcześniej |
| `Warning: inferring latch(es)`                  | brakujący `else` / `default`, patrz [[Podstawy składni Verilog]], punkt 7 |
| Po wyłączeniu zasilania projekt znika           | to normalne po `make prog`. Na stałe: `make flash`                           |
| Po `make flash` chcę "czystą" płytkę            | wgraj inny projekt przez `make flash`. Oryginalne demo Terasic jest na ich stronie (DE10-Lite CD) |

---

## Przydatne linki
- **Instrukcja DE10-Lite** (Terasic, *DE10-Lite User Manual*): pełna lista pinów (wyświetlacze, VGA, GPIO)
- **HDLBits**: ćwiczenia z Veriloga w przeglądarce
- **Nandland**: tutoriale dla początkujących
