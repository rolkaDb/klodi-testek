# Mentes vonalkód-app — tervezési döntések

## Az ítéletek rangsora

Négy ítélet van: `safe`, `caution`, `unknown`, `unsafe`.

A rangsorban az **`unknown` rosszabb, mint a `caution`**. Ez szándékos: érzékeny
étrendnél a „nem tudjuk" veszélyesebb, mint a jelölt nyomnyi mennyiség.

A szigorú mód a `caution`-ből `unsafe`-et csinál, de az `unknown`-t **nem** bántja:
az adathiány továbbra is adathiány, nem tény.

## Forrásjelölés

Minden ítélet visz magával egy `source` mezőt:

- `data` — az OpenFoodFacts rekordjából
- `note` — a felhasználó saját jegyzetéből
- `translation` — gépi fordításon alapul

Ez azért kell, hogy adat és feltételezés között egy pillantásból látszódjon a
különbség. Enélkül hónapokkal később a felhasználó a saját tippjét nézné tényadatnak.

A gyártói címke (`labelTags`) akkor is `data` marad, ha közben fordítottunk —
az OFF-címke nem a fordításból jön.

## Miért nem mondunk „mentes"-t, ha nem értjük a nyelvet

Ha az összetevő-szöveg olyan nyelven van, amit a szótár nem ismer, és nincs
allergén-címke sem, az ítélet `unknown`. A hallgatásunk ilyenkor nem bizonyíték,
csak értetlenség.

A szótár által lefedett nyelvek: hu, en, de, fr, pl, ro, it, cs, sk, hr, sr, sl, bs, es.
Nem latin írás (cirill, görög, héber, arab) esetén a tokenizáló semmit nem lát.

## A gépi fordítás szerepe

Az AI **csak fordít**, az ítéletet továbbra is a determinisztikus kulcsszó-motor
hozza a lefordított szövegen. Így egy prompt injection nem tud hamis „mentes"
ítéletet előállítani: a támadó mondat lefordítva is csak szöveg marad, a döntést
a kód hozza.

Amit a modell rontani tud, az egy félrefordítás — az pedig a `caution`/`unsafe`
irányba visz, nem hamis biztonságba.

**Amit nem csinálunk:** ha egyáltalán nincs összetevő-adat, nem tippelünk a
terméknévből. Az pontosan az a hallucináció lenne, ami ellen az app fel van építve.

## Tartalmaz vs. tartalmazhat

A fordítás külön mezőben adja vissza a „nyomokban tartalmazhat" részt.
Ha egyben hagynánk, a kulcsszó-motor minden nyomnyi allergént tiltásnak venne,
mert csak a szót látja, a „tartalmazhat" kontextust nem.

## Tagadás-felismerés

A „gluténmentes", „sans gluten", „glutenfrei" alakok kioltják a kulcsszót.
Enélkül épp az ellenkezőjét állítanánk annak, ami a csomagoláson van.

Ismert korlát: a „non-dairy creamer" mégis riaszt, mert a „non" csak a közvetlenül
utána álló szót oltja ki. Ez tudatos: fölösleges riasztás vállalható,
elmulasztott riasztás nem.
