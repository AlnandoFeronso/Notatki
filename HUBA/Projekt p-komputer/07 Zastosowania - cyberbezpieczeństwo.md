---
tags: [p-komputer, zastosowania, cyberbezpieczeństwo]
---
# 07 Zastosowania: cyberbezpieczeństwo

Wróć: [[00 Przegląd projektu]] · Powiązane: [[06 Zastosowania - optymalizacja]], [[05 TRNG - generator szumu]]

> [!abstract] Wniosek z analizy
> Najmocniejsze zastosowania to **optymalizacja *w* bezpieczeństwie** (obrona sieci, rozmieszczenie zasobów) i **TRNG jako źródło entropii**. **Łamanie kryptografii** (np. RSA) to tylko **demonstracja na zabawkowych liczbach** i tak trzeba je przedstawiać.

---

## Ocena kierunków

| Kierunek | Realność | Wartość | Uwagi |
| --- | --- | --- | --- |
| A. Optymalizacja w obronie sieci | ✅ wysoka | ✅ wysoka | naturalne problemy grafowe, a dane wrażliwe uzasadniają liczenie lokalnie |
| B. Wykrywanie anomalii (maszyny Boltzmanna) | 🟡 średnia | 🟡 | p-bity przyspieszają próbkowanie przy uczeniu. Kierunek badawczy |
| C. TRNG / źródło entropii | ✅ wysoka | ✅ | osobna linia produktu, ale wymaga certyfikacji, żeby ktoś zaufał |
| D. Kryptoanaliza (faktoryzacja, SAT) | ❌ dla prawdziwych szyfrów | 🟡 jako pokaz | tylko małe liczby / uproszczone szyfry |

---

## A. Optymalizacja w obronie sieci

### Rozmieszczenie monitoringu (sensory IDS, tapy, honeypoty)
**Problem:** wybrać jak najmniej hostów do monitorowania tak, żeby **każde połączenie** w sieci miało monitorowany co najmniej jeden koniec (pokrycie wierzchołkowe, *vertex cover*).

$$
\min \; \sum_i x_i \;+\; P \sum_{(i,j) \in E} (1 - x_i)(1 - x_j), \qquad P > 1
$$

- $x_i = 1$: monitorujemy host $i$
- pierwszy człon: koszt (liczba sensorów)
- drugi człon: kara za każde niepokryte połączenie

*(Sprawdzone: dla losowego grafu 8 hostów / 10 połączeń minimum QUBO = prawdziwe minimalne pokrycie.)*
Rozszerzenia: różne koszty sensorów ($c_i x_i$), wagi połączeń wg ryzyka, limit k sensorów.

### Utwardzanie sieci na grafie ataku
**Problem:** mając graf możliwych ruchów napastnika (podatności, uprawnienia), wybrać **zmiany o minimalnym koszcie** (łatki, reguły firewalla, odebranie uprawnień), które **odcinają wszystkie ścieżki** do kluczowych zasobów.

> [!warning] Nie wszystko jest NP-trudne
> Odcięcie **jednego** źródła od **jednego** celu (minimalne cięcie s-t) rozwiązuje się **szybko i dokładnie** klasycznym algorytmem max-flow. p-komputer nie ma tu sensu.
> Ma sens dla wersji **trudnych**: wiele celów (multicut), budżet, zależności między zmianami, koszty zależne od kombinacji. **Zawsze sprawdzić, czy problem nie ma szybkiego algorytmu klasycznego!**

### Priorytetyzacja łatek w budżecie
**Problem:** okno serwisowe ma X godzin. Które łatki wdrożyć, żeby maksymalnie zmniejszyć ryzyko, uwzględniając zależności (łatka B wymaga A) i to, że niektóre łatki chronią te same ścieżki ataku? To odmiana plecaka z interakcjami kwadratowymi.

### Segmentacja sieci
**Problem:** podzielić hosty na strefy (VLAN / mikrosegmentacja) tak, żeby **ruch między strefami był minimalny** przy ograniczeniach rozmiaru stref. To podział grafu, bliski kuzyn Max-Cut ⭐⭐⭐.

### Inne
- przydział analityków SOC do alertów / dyżurów (harmonogramy)
- rozmieszczenie honeypotów maksymalizujące szansę wykrycia ruchu bocznego

> [!success] Argument sprzedażowy: dane nie opuszczają firmy
> Topologia sieci, podatności i graf ataku to **bardzo wrażliwe dane**. Wiele firm **nie wyśle ich** do chmurowego solwera ani komputera kwantowego. Lokalne urządzenie (on-premise) to realny wyróżnik.

---

## B. Wykrywanie anomalii: maszyny Boltzmanna

- **Ograniczona maszyna Boltzmanna (RBM)** to sieć p-bitów. Uczy się rozkładu "normalnego" ruchu, a **wysoka energia** (mało prawdopodobny stan) oznacza anomalię.
- Uczenie RBM wymaga **dużo próbkowania**, a to dokładnie to, co p-komputer robi szybko.
- **Uwaga:** dziś detekcję anomalii robi się głównie sieciami neuronowymi na GPU. To kierunek **badawczy** na później, nie pierwsze demo.

---

## C. TRNG jako źródło entropii

- Klucze kryptograficzne, nonce, tokeny: **jakość losowości = bezpieczeństwo**.
- Żeby ktoś zaufał TRNG, potrzebne są: estymacja entropii i testy zdrowia wg **NIST SP 800-90B**, ewentualnie **BSI AIS 31**, odporność na wstrzykiwanie zakłóceń, kondycjonowanie. Szczegóły w [[05 TRNG - generator szumu]].
- Synergia: ten sam moduł TRNG dla p-komputera i jako osobny produkt.

---

## D. Kryptoanaliza: uczciwie

### Faktoryzacja przez "invertible logic"
- Budujemy z p-bitów **układ mnożący** (z bramek jak AND w [[01 Teoria - p-bity, Ising, QUBO]]) i **przypinamy wyjście** do liczby N. Sieć sama szuka wejść, czyli czynników N.
- Opublikowano faktoryzację **małych liczb** na sprzętowych p-bitach (Borders i in., Nature 2019).
- **Liczba p-bitów rośnie z kwadratem liczby bitów** (mnożarka n×n), a **nie ma dowodu przyspieszenia wykładniczego**. Czas szukania szybko rośnie.

> [!danger] p-komputer to nie komputer kwantowy
> p-bity **nie uruchamiają algorytmu Shora**. **Nie zagrażają RSA-2048** ani współczesnej kryptografii.
> Twierdzenie w prezentacji, że "łamiemy RSA", szybko zniszczy wiarygodność projektu u każdego eksperta. Prawidłowo: *"demonstracja odwracalnej logiki na przykładzie faktoryzacji małych liczb"*.

### Kryptoanaliza przez SAT
- Szyfry da się zapisać jako formuły logiczne (SAT), a p-komputer może szukać kluczy dla **uproszczonych** szyfrów (mało rund, małe klucze).
- Klasyczne solwery SAT są tu bardzo silne. Wartość badawcza, nie praktyczna.

---

## Etyka i zakres

> [!note] Zasady projektu
> - Kryptoanaliza tylko na **własnych danych**, publicznych zadaniach badawczych i CTF.
> - Zastosowania sieciowe na **danych testowych / własnej infrastrukturze** albo za zgodą właściciela sieci.
> - Projekt komunikujemy jako **narzędzie obronne** (optymalizacja ochrony, entropia), a nie ofensywne.

---

## Proponowany pierwszy pokaz "bezpieczeństwa"
**Rozmieszczenie sensorów IDS w syntetycznej sieci firmowej** (30–100 hostów):
1. generator sieci testowej (Python, `networkx`)
2. QUBO pokrycia wierzchołkowego z kosztami
3. rozwiązanie na p-komputerze i porównanie z dokładnym solverem
4. wizualizacja: sieć z zaznaczonymi sensorami

Pokazuje cały pipeline ([[00 Przegląd projektu]]) na zrozumiałym, obronnym przykładzie.
