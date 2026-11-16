---
tags: [fpga, verilog, składnia]
---
# Podstawy składni Verilog

> [!info] Zanim zaczniesz
> Najpierw przeczytaj [[O co chodzi]]: **kod w Verilogu opisuje układ, a nie listę kroków.**
> Jak wgrać przykłady na płytkę: [[W jaki sposób otworzyć i używać projektu verilog]].
> Na końcu tej notatki są **ćwiczenia z rozwiązaniami**, sprawdzone na DE10-Lite.

---

## 1. Moduł, czyli czarna skrzynka

Każdy układ to **moduł**: skrzynka z przewodami wchodzącymi (`input`) i wychodzącymi (`output`).

```verilog
module blink (                        // nazwa modułu
    input  wire       MAX10_CLK1_50,  // 1 przewód wchodzący
    input  wire [1:0] KEY,            // 2 przewody: KEY[1], KEY[0]
    input  wire [9:0] SW,             // 10 przewodów: SW[9] … SW[0]
    output reg  [9:0] LEDR            // 10 przewodów wychodzących
);

    // ... tu opisujesz, co jest w środku ...

endmodule
```

```
             ┌─────────────────┐
 MAX10_CLK ─►│                 │
       KEY ─►│      blink      ├─► LEDR
        SW ─►│                 │
             └─────────────────┘
```

> [!note] Komentarze
> `// do końca linii` albo `/* wiele linii */`, tak jak w C.

---

## 2. `wire` i `reg`: rodzaje sygnałów

| Typ    | Co to                                      | Gdzie mu przypisujesz wartość           |
| ------ | ------------------------------------------ | --------------------------------------- |
| `wire` | **przewód**, sam niczego nie pamięta       | `assign` albo podłączenie do portu      |
| `reg`  | sygnał ustawiany w bloku `always`          | tylko wewnątrz `always`                 |

```verilog
wire       a;          // 1 bit
wire [3:0] liczba;     // 4 bity: liczba[3], liczba[2], liczba[1], liczba[0]
reg  [9:0] licznik;    // 10 bitów
wire       k0 = ~KEY[0];  // deklaracja + przypisanie naraz
```

> [!warning] `reg` ≠ rejestr!
> Mimo nazwy `reg` **nie zawsze** oznacza pamięć. W `always @(*)` to zwykłe bramki, a w `always @(posedge clk)` to prawdziwy rejestr (przerzutniki).
> Prosta zasada: **przypisujesz w `always` → `reg`. Przypisujesz przez `assign` → `wire`.**

---

## 3. Liczby

Format: **`rozmiar'podstawa wartość`**

| Zapis            | Znaczenie                         | Bity           |
| ---------------- | --------------------------------- | -------------- |
| `4'b1010`        | 4 bity, binarnie                  | `1010`         |
| `8'hFF`          | 8 bitów, szesnastkowo (hex)       | `11111111`     |
| `10'd5`          | 10 bitów, dziesiętnie             | `0000000101`   |
| `10'b0`          | 10 zer                            | `0000000000`   |
| `1'b1`           | jeden bit = 1                     | `1`            |
| `50_000_000`     | podkreślniki dla czytelności      | (ignorowane)   |
| `5`              | bez rozmiaru = liczba 32-bitowa   |                |

Podstawy: `b` binarna, `d` dziesiętna, `h` szesnastkowa, `o` ósemkowa.

---

## 4. Wektory: wycinanie, sklejanie, powielanie

```verilog
LEDR[0]          // jeden bit
LEDR[3:0]        // bity od 3 do 0 (4 najniższe diody)
SW[7:4]          // przełączniki 4–7 jako 4-bitowa liczba

{SW[1], SW[0]}   // SKLEJANIE: 2 bity w jeden 2-bitowy wektor
{4'b0, SW[5:0]}  // 4 zera + 6 przełączników = 10 bitów
{10{SW[0]}}      // POWIELANIE: SW[0] powtórzone 10 razy
{LEDR[8:0], LEDR[9]}  // ROTACJA w lewo (najstarszy bit wraca na koniec)
```

> [!tip] Indeksy liczy się od prawej
> W `[9:0]` bit **0 jest najmłodszy** (po prawej, jak jedności w liczbie), a bit 9 najstarszy.
> `10'b0000000001` → świeci tylko `LEDR[0]`.

---

## 5. Operatory

### Bitowe: działają na każdym bicie osobno
| Operator  | Nazwa | Przykład (`a=1100`, `b=1010`) |
| --------- | ----- | ----------------------------- |
| `~a`      | NOT   | `0011`                        |
| `a & b`   | AND   | `1000`                        |
| `a \| b`  | OR    | `1110`                        |
| `a ^ b`   | XOR   | `0110`                        |

### Logiczne: wynik to 1 bit (prawda/fałsz), do warunków w `if`
`!a` (nie), `a && b` (i), `a || b` (lub)

### Porównania
`==`, `!=`, `<`, `>`, `<=`, `>=`

> [!warning] `<=` ma dwa znaczenia
> W `if (a <= b)` to **"mniejsze lub równe"**. Jako `x <= y;` (instrukcja) to **przypisanie nieblokujące** (patrz pkt 8). Verilog rozpoznaje to po kontekście.

### Arytmetyka
`+`, `-`, `*`, `/`, `%`

> [!warning] Nie wszystko jest "za darmo"
> `+` i `-` są tanie. `*` korzysta ze sprzętowych mnożarek w chipie. **Dzielenie `/` i `%` przez zmienną** tworzy ogromny, wolny układ, więc unikaj go.
> Dzielenie przez **stałą** (np. `CLK_HZ / STEP_HZ` w `localparam`) jest liczone podczas kompilacji i nic nie kosztuje. Dzielenie przez potęgę dwójki to po prostu przesunięcie: `x >> 3` = `x / 8`.

### Przesunięcia
`a << 1` (w lewo = ×2), `a >> 1` (w prawo = ÷2). Bity, które "wypadną", przepadają, a z drugiej strony wchodzą zera.

### Redukcja: cały wektor do 1 bitu
```verilog
&SW[7:2]   // 1, jeśli WSZYSTKIE te przełączniki są w górze
|SW[7:2]   // 1, jeśli KTÓRYKOLWIEK jest w górze
^SW        // 1, jeśli nieparzysta liczba jedynek (parzystość)
```

### Warunek w jednej linii (jak w C)
```verilog
assign y = SW[0] ? a : b;   // jeśli SW[0] to a, w przeciwnym razie b
```
W sprzęcie to **multiplekser** (przełącznik wybierający jedno z dwóch wejść).

---

## 6. `assign`: stałe połączenie

```verilog
wire jest_ok;
assign jest_ok  = SW[0] & SW[1];    // bramka AND, działa cały czas
assign LEDR[0]  = jest_ok;          // przewód do diody
```

- tylko dla `wire`
- każdy `assign` to osobny kawałek sprzętu, **kolejność linii nie ma znaczenia**
- jeden sygnał może mieć **tylko jeden** `assign` (dwa źródła na jednym przewodzie to zwarcie)

---

## 7. `always @(*)`: układ kombinacyjny

Kiedy logika jest bardziej złożona, z warunkami, wygodniej użyć bloku:

```verilog
always @(*) begin           // (*) = "przelicz, gdy KTÓREKOLWIEK wejście się zmieni"
    LEDR = 10'b0;           // domyślnie wszystko zgaszone
    if (SW[9])
        LEDR[3:0] = SW[3:0];
    else
        LEDR[3:0] = ~SW[3:0];
end
```

- używaj **`=`** (przypisanie blokujące)
- sygnały ustawiane tutaj deklarujesz jako `reg`
- wewnątrz bloku kolejność **ma** znaczenie: późniejsze przypisanie nadpisuje wcześniejsze (stąd sztuczka "najpierw wszystko na 0, potem ustaw wybrane")

> [!danger] Pułapka: zatrzask (latch)
> ```verilog
> always @(*) begin
>     if (SW[0])
>         LEDR[0] = SW[1];
>     // brak else! co ma być, gdy SW[0] = 0?
> end
> ```
> Jeśli w **jakimś** przypadku nie ustawisz sygnału, Verilog uzna, że ma **zapamiętać** starą wartość, i zbuduje niechciany "latch".
> Quartus ostrzeże: `Warning: inferring latch(es)`.
> **Rozwiązanie:** zawsze dawaj `else` albo ustaw wartość domyślną na początku bloku.

---

## 8. `always @(posedge clk)`: układ sekwencyjny (z pamięcią)

```verilog
reg [9:0] licznik;

always @(posedge MAX10_CLK1_50) begin   // "przy każdym tyknięciu zegara"
    licznik <= licznik + 1;              // dodaj 1 (50 milionów razy na sekundę!)
end
```

- `posedge` to zbocze narastające: moment, gdy zegar zmienia się z 0 na 1
- używaj **`<=`** (przypisanie nieblokujące)
- tu powstają prawdziwe **rejestry**, które pamiętają wartość aż do następnego tyknięcia

### Reset
```verilog
always @(posedge MAX10_CLK1_50 or negedge KEY[0]) begin
    if (!KEY[0])              // przycisk wciśnięty (= 0) → reset
        licznik <= 0;
    else
        licznik <= licznik + 1;
end
```
`negedge KEY[0]` oznacza, że reset działa **natychmiast** po wciśnięciu, nie czekając na zegar.

---

## 9. `=` kontra `<=`, czyli najważniejsza różnica

Załóżmy, że przed tyknięciem zegara `a = 1`, `b = 2`:

```verilog
// NIEBLOKUJĄCE <=  (poprawne w układach z zegarem)
always @(posedge clk) begin
    a <= b;
    b <= a;
end
// po tyknięciu: a = 2, b = 1  ✅ zamiana!
```

```verilog
// BLOKUJĄCE =  (źle w układach z zegarem)
always @(posedge clk) begin
    a = b;      // a = 2 od razu
    b = a;      // b = 2 (bo a już jest 2!)
end
// po tyknięciu: a = 2, b = 2  ❌
```

`<=` oznacza: **"wszystkie prawe strony odczytaj PRZED tyknięciem, a zapisz WSZYSTKIE naraz"**. Dokładnie tak działają prawdziwe przerzutniki.

> [!tip] Zasada, której się trzymaj
> | Blok                         | Przypisanie |
> | ---------------------------- | ----------- |
> | `always @(*)`                | `=`         |
> | `always @(posedge clk)`      | `<=`        |
>
> Nigdy nie mieszaj ich w jednym bloku.

---

## 10. `if` / `else` i `case`

```verilog
if (warunek) begin
    ...
end else if (inny_warunek) begin
    ...
end else begin
    ...
end
```
`begin` … `end` działa jak `{` … `}` w C. Przy jednej instrukcji można je pominąć.

```verilog
always @(*) begin
    case (SW[9:8])                    // sprawdź 2 przełączniki
        2'b00:   LEDR[9:6] = 4'b0001;
        2'b01:   LEDR[9:6] = 4'b0011;
        2'b10:   LEDR[9:6] = 4'b0111;
        default: LEDR[9:6] = 4'b1111; // wszystkie pozostałe przypadki
    endcase
end
```
> [!tip] Zawsze dodawaj `default`
> Bez niego łatwo o zatrzask (patrz pkt 7).

W sprzęcie `if` i `case` to **multipleksery**: wszystkie gałęzie są zbudowane jednocześnie, a warunek wybiera, której wynik przejdzie dalej. Nic tu nie "skacze" jak w programie.

---

## 11. `parameter` i `localparam`: stałe

```verilog
module blink #(
    parameter integer CLK_HZ  = 50_000_000,   // można zmienić z zewnątrz
    parameter integer STEP_HZ = 4
) ( ...porty... );

    localparam integer DIV = CLK_HZ / STEP_HZ; // liczona wewnątrz, nie da się zmienić z zewnątrz
```
- `parameter` to stała, którą można nadpisać przy użyciu modułu (np. testbench ustawia `CLK_HZ(100)`, żeby symulacja trwała krótko)
- `localparam` to stała prywatna, często wyliczana z parametrów
- liczone **podczas kompilacji**, nie zajmują miejsca w układzie

---

## 12. Używanie modułu w innym module

Moduły to klocki, z których składa się większe układy:

```verilog
// pół-sumator: dodaje dwa bity
module half_adder (
    input  wire a,
    input  wire b,
    output wire sum,
    output wire carry
);
    assign sum   = a ^ b;
    assign carry = a & b;
endmodule
```

```verilog
// użycie w module blink:
wire s, c;

half_adder ha1 (          // typ modułu, potem nazwa tego egzemplarza
    .a    (SW[0]),        // .port_modułu (sygnał_u_mnie)
    .b    (SW[1]),
    .sum  (s),
    .carry(c)
);

assign LEDR[0] = s;  // ⚠️ jeśli LEDR to `output reg`, przypisz w always @(*)
```

Możesz wstawić **wiele egzemplarzy** tego samego modułu (`ha1`, `ha2`…). Każdy to osobny, fizyczny kawałek układu.

> [!note] Gdzie trzymać moduły
> Każdy moduł najlepiej w osobnym pliku w `rtl/`, np. `rtl/half_adder.v`. Makefile sam łapie wszystkie `rtl/*.v`, ale Quartus potrzebuje dopisku w `.qsf`, patrz [[W jaki sposób otworzyć i używać projektu verilog#Dodanie nowego pliku .v]].

---

## 13. Rzeczy tylko do symulacji

Tych **nie używaj w `rtl/`**, bo prawdziwy chip ich nie potrafi:

| Konstrukcja  | Znaczenie                        |
| ------------ | -------------------------------- |
| `#10`        | czekaj 10 ns                     |
| `initial`    | wykonaj raz na początku symulacji |
| `$display()` | wypisz tekst (jak `printf`)      |
| `$finish`    | zakończ symulację                |
| `$dumpfile` / `$dumpvars` | zapisz przebiegi do GTKWave |

Należą do **testbencha** (`sim/blink_tb.v`), czyli "sztucznej płytki", która podaje sygnały i sprawdza wyniki.

---

## 14. Najczęstsze błędy

| Objaw                                       | Przyczyna                                                        |
| ------------------------------------------- | ---------------------------------------------------------------- |
| `must have a variable data type` (Quartus) / `not a valid l-value` (symulator) | przypisujesz w `always` do `wire` → zmień na `reg` |
| `inferring latch(es)`                       | brak `else` / `default` w `always @(*)`                          |
| `multiple drivers`                          | ten sam sygnał ustawiany w dwóch miejscach                       |
| Przycisk działa "odwrotnie"                 | `KEY` = 0, gdy wciśnięty → użyj `~KEY[0]`                         |
| Dioda świeci cały czas słabo / miga za szybko | licznik bez dzielnika: 50 MHz to za szybko dla oka              |
| Zapomniany `;` / `end` / `endmodule`        | Neovim podkreśli to od razu (verible)                            |
| Wynik dodawania ucięty                      | za wąski wynik: `4 bity + 4 bity` potrzebuje **5 bitów**         |

---

## 15. Ściągawka

```verilog
module nazwa (
    input  wire       clk,
    input  wire [3:0] wej,
    output reg  [3:0] wyj
);
    wire [3:0] tmp;
    assign tmp = wej ^ 4'b1111;           // kombinacyjnie: assign

    always @(*) begin                     // kombinacyjnie: blok, używa =
        wyj = 4'b0;
        if (wej[0]) wyj = tmp;
    end

    reg [7:0] cnt;
    always @(posedge clk) begin           // sekwencyjnie: blok, używa <=
        cnt <= cnt + 1;
    end
endmodule
```

---

## 16. Ćwiczenia na płytkę

> [!info] Jak robić ćwiczenia
> 1. Najpierw zapisz oryginał: `git add -A && git commit -m "oryginalny blink"` (szczegóły w [[W jaki sposób otworzyć i używać projektu verilog#Git, czyli jak nie zgubić działającej wersji]])
> 2. Zamień **całą zawartość** `rtl/blink.v` na kod z ćwiczenia (nazwa modułu i porty zostają te same: `blink`)
> 3. `make prog` i sprawdź na płytce
> 4. Powrót do oryginału: `git restore rtl/blink.v`
>
> ⚠️ `make sim` testuje tylko oryginalny licznik. Ćwiczenia sprawdzaj na płytce.
>
> Najpierw spróbuj sam, rozwiązanie rozwiń dopiero potem.

### Ćwiczenie 1: przełączniki → diody
Każdy przełącznik zapala diodę nad sobą.

> [!example]- Rozwiązanie
> ```verilog
> module blink (
>     input  wire       MAX10_CLK1_50,
>     input  wire [1:0] KEY,
>     input  wire [9:0] SW,
>     output reg  [9:0] LEDR
> );
>   always @(*) begin
>     LEDR = SW;
>   end
> endmodule
> ```
> **Co tu się dzieje:** 10 przewodów, zero logiki. Quartus używa na to praktycznie **0 elementów logicznych**.
> **Pobaw się:** `LEDR = ~SW;` (odwrotnie), `LEDR = {10{SW[0]}};` (jeden przełącznik steruje wszystkimi), `LEDR = {SW[0], SW[1], SW[2], SW[3], SW[4], SW[5], SW[6], SW[7], SW[8], SW[9]};` (lustrzane odbicie).

### Ćwiczenie 2: bramki logiczne na przyciskach
Dioda 0 = AND, 1 = OR, 2 = XOR, 3 = NAND dwóch przycisków. Pamiętaj: wciśnięty = `0`!

> [!example]- Rozwiązanie
> ```verilog
> module blink (
>     input  wire       MAX10_CLK1_50,
>     input  wire [1:0] KEY,
>     input  wire [9:0] SW,
>     output reg  [9:0] LEDR
> );
>   wire k0 = ~KEY[0];  // 1 = przycisk 0 wciśnięty
>   wire k1 = ~KEY[1];  // 1 = przycisk 1 wciśnięty
>
>   always @(*) begin
>     LEDR    = 10'b0;
>     LEDR[0] = k0 & k1;   // AND
>     LEDR[1] = k0 | k1;   // OR
>     LEDR[2] = k0 ^ k1;   // XOR
>     LEDR[3] = ~(k0 & k1); // NAND
>   end
> endmodule
> ```
> **Wypełnij tabelkę, naciskając przyciski:**
>
> | KEY0 wciśn. | KEY1 wciśn. | AND | OR | XOR | NAND |
> | ----------- | ----------- | --- | -- | --- | ---- |
> | nie         | nie         | 0   | 0  | 0   | 1    |
> | tak         | nie         | 0   | 1  | 1   | 1    |
> | nie         | tak         | 0   | 1  | 1   | 1    |
> | tak         | tak         | 1   | 1  | 0   | 0    |

### Ćwiczenie 3: kalkulator (sumator 4-bitowy)
`SW[3:0]` to liczba A, `SW[7:4]` to liczba B. Diody pokazują A + B binarnie.

> [!example]- Rozwiązanie
> ```verilog
> module blink (
>     input  wire       MAX10_CLK1_50,
>     input  wire [1:0] KEY,
>     input  wire [9:0] SW,
>     output reg  [9:0] LEDR
> );
>   always @(*) begin
>     LEDR = SW[3:0] + SW[7:4];  // wynik ma 5 bitów, reszta LED-ów = 0
>   end
> endmodule
> ```
> **Test:** A = `1001` (9), B = `1100` (12) → diody `10101` = 21 ✅. Maksimum: 15 + 15 = 30 = `11110`.
>
> **Ciekawostka:** Verilog liczy dodawanie w szerokości **lewej strony** (`LEDR` = 10 bitów), więc przeniesienie na 5. bit się nie gubi. Gdyby wynik szedł do 4-bitowego sygnału, 15 + 15 dałoby 14 (przeniesienie by przepadło)!
> `make lint` ostrzeże tu `WIDTHEXPAND` ("rozszerzam 4 bity do 10"). W tym miejscu to zamierzone.

### Ćwiczenie 4: licznik w dół
W oryginalnym `blink.v` spraw, żeby diody liczyły **w dół**.

> [!example]- Rozwiązanie
> W oryginalnym pliku zmień jedną linijkę:
> ```verilog
> LEDR <= LEDR + 1'b1;
> ```
> na
> ```verilog
> LEDR <= LEDR - 1'b1;
> ```
> Po resecie (0) pierwszy krok da `1111111111` (1023), bo liczba "zawija się" jak licznik kilometrów.
> **Dodatkowo:** zmień `STEP_HZ = 4` na `1` albo `20` i zobacz różnicę prędkości.

### Ćwiczenie 5: światełko (Knight Rider)
Jedna zapalona dioda przesuwa się w kółko. `SW[0]` zmienia kierunek, `KEY[0]` resetuje.

> [!example]- Rozwiązanie
> ```verilog
> module blink #(
>     parameter integer CLK_HZ  = 50_000_000,  // zegar płytki
>     parameter integer STEP_HZ = 8            // ile kroków światełka na sekundę
> ) (
>     input  wire       MAX10_CLK1_50,
>     input  wire [1:0] KEY,
>     input  wire [9:0] SW,
>     output reg  [9:0] LEDR
> );
>   localparam integer DIV = CLK_HZ / STEP_HZ;   // ile tyknięć zegara na jeden krok
>
>   reg  [31:0] tick_cnt;                        // licznik tyknięć zegara
>   wire        reset_n = KEY[0];
>   wire        step    = (tick_cnt == DIV - 1); // 1 przez jedno tyknięcie, STEP_HZ razy na sekundę
>
>   always @(posedge MAX10_CLK1_50 or negedge reset_n) begin
>     if (!reset_n) begin
>       tick_cnt <= 0;
>       LEDR     <= 10'b0000000001;              // zapalona tylko jedna dioda
>     end else begin
>       if (step) tick_cnt <= 0;
>       else      tick_cnt <= tick_cnt + 1;
>
>       if (step) begin
>         if (SW[0]) LEDR <= {LEDR[0], LEDR[9:1]};  // przesuń w prawo
>         else       LEDR <= {LEDR[8:0], LEDR[9]};  // przesuń w lewo
>       end
>     end
>   end
> endmodule
> ```
> **Nowe rzeczy:**
> - `{LEDR[8:0], LEDR[9]}` to **rotacja**: bity przesuwają się o 1 w lewo, a ten, który wypada, wraca z drugiej strony
> - `step` to sygnał, który jest `1` **tylko przez jedno tyknięcie** co 1/8 sekundy. To typowy sposób na "zwolnienie" czegoś względem 50 MHz
>
> ⚠️ Po wgraniu **naciśnij raz KEY0**, żeby zapalić pierwszą diodę.
>
> **Wyzwanie:** zrób, żeby światełko odbijało się od krańców zamiast krążyć w kółko (podpowiedź: potrzebujesz dodatkowego `reg` pamiętającego kierunek).

### Ćwiczenie 6: licznik wciśnięć przycisku
Każde wciśnięcie `KEY[1]` dodaje 1 do liczby na diodach. `KEY[0]` zeruje.

> [!example]- Rozwiązanie
> ```verilog
> module blink (
>     input  wire       MAX10_CLK1_50,
>     input  wire [1:0] KEY,
>     input  wire [9:0] SW,
>     output reg  [9:0] LEDR
> );
>   wire       reset_n = KEY[0];
>   reg  [1:0] key_sync;               // 2 przerzutniki: synchronizacja przycisku z zegarem
>   reg        key_prev;               // stan przycisku w poprzednim takcie
>   wire       key_now = key_sync[1];
>   wire       pressed = key_prev & ~key_now;  // było 1, jest 0 → właśnie wciśnięty
>
>   always @(posedge MAX10_CLK1_50 or negedge reset_n) begin
>     if (!reset_n) begin
>       key_sync <= 2'b11;
>       key_prev <= 1'b1;
>       LEDR     <= 10'b0;
>     end else begin
>       key_sync <= {key_sync[0], KEY[1]};
>       key_prev <= key_now;
>       if (pressed) LEDR <= LEDR + 1'b1;
>     end
>   end
> endmodule
> ```
> **Dlaczego nie po prostu `if (!KEY[1]) LEDR <= LEDR + 1;`?**
> Bo przy 50 MHz przycisk trzymany przez 0,1 s to **5 milionów** tyknięć, czyli licznik dodałby 5 milionów, a nie 1!
> Trzeba wykryć **moment** wciśnięcia (zbocze): "w poprzednim tyknięciu było 1, teraz jest 0".
>
> **Po co `key_sync`?** Przycisk zmienia się w dowolnej chwili, niezależnie od zegara. Dwa przerzutniki pod rząd "wyrównują" go do zegara. To standardowa technika: **każde wejście z zewnątrz przepuszczaj przez 2 rejestry**.
>
> Na DE10-Lite przyciski mają sprzętowy filtr drgań (układ Schmitta), więc jedno wciśnięcie = jedno zliczenie. Na zwykłych przyciskach trzeba by jeszcze dodać filtr drgań styków (*debouncing*).

---

## Co dalej
- **Wyświetlacz 7-segmentowy** (`HEX0`…`HEX5`): pokaż cyfry zamiast binarnych diod (nowe: `case` jako dekoder)
- **Stoper**: liczy sekundy i dziesiąte części na wyświetlaczach
- **PWM**: płynne rozjaśnianie diody
- **Maszyna stanów (FSM)**: np. sygnalizacja świetlna, zamek na kod z przełączników
- Do ćwiczeń online: **HDLBits** (zadania sprawdzane w przeglądarce)
