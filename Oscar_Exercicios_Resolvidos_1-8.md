# Oscar - Exercícios MongoDB (Níveis 1 a 8)

Coleção usada: `oscar_indicados`. Campos: `id_registro`, `ano_filmagem`, `ano_cerimonia`, `cerimonia`, `categoria`, `nome_do_indicado`, `nome_do_filme`, `vencedor`.

As respostas seguem a ordem de execução dos exercícios (o 1.6 acrescenta dados antes do nível 2, o 7 altera a base antes do nível 8).

## Preparação: limpeza de registros com problema

Antes de começar, olhando a base achei 5 documentos fora do padrão. Corrigi para não bagunçar as contagens e as ordenações:

- `id_registro` 5, 5313 e 7509 têm um campo extra (`fa` / `diretor`) que não existe nos outros
- `id_registro` 6 tem `ano_cerimonia: "Mil Novecentos e vinte 8"` (string). Em ordem decrescente o Mongo coloca string antes de número, então isso quebraria o 1.4
- `id_registro` 7507 tem o campo escrito `"ven cedor"` (com espaço)

```js
db.oscar_indicados.updateMany({}, { $unset: { fa: "", diretor: "" } })
db.oscar_indicados.updateOne({ id_registro: 6 }, { $set: { ano_cerimonia: 1928 } })
db.oscar_indicados.updateOne({ id_registro: 7507 }, { $rename: { "ven cedor": "vencedor" } })
```

---

## Nível 1: Primeiros Passos

**1.1 Quantos registros existem na coleção?**

```js
db.oscar_indicados.countDocuments()
```

R: 10889 registros

**1.2 Categorias únicas**

```js
db.oscar_indicados.distinct("categoria")
```

R: 115 categorias (os nomes mudaram bastante ao longo dos anos, por exemplo `ACTOR` virou `ACTOR IN A LEADING ROLE`, e cada nome conta como uma categoria)

**1.3 Primeiro ano de cerimônia**

```js
db.oscar_indicados.find({}, { _id: 0, ano_cerimonia: 1 }).sort({ ano_cerimonia: 1 }).limit(1)
```

R: 1928

**1.4 Último ano de cerimônia**

```js
db.oscar_indicados.find({}, { _id: 0, ano_cerimonia: 1 }).sort({ ano_cerimonia: -1 }).limit(1)
```

R: 2024 (96ª cerimônia). Depois do 1.6 passa a ser 2026.

**1.5 Quantas cerimônias estão registradas?**

```js
db.oscar_indicados.distinct("cerimonia").length
```

R: 96 cerimônias (98 depois do 1.6). Usei o campo `cerimonia` porque `distinct("ano_cerimonia")` daria um valor a mais por causa do registro com o ano escrito por extenso.

**1.6 Atualizar com os dados do Oscar 2025 e 2026**

Montei os registros da 97ª cerimônia (2025, filmes de 2024) e da 98ª (2026, filmes de 2025) no mesmo formato da base, com `id_registro` continuando de 10890. Estão no arquivo `novos_2025_2026.json`.

```bash
mongoimport --db oscar --collection oscar_indicados --file novos_2025_2026.json --jsonArray
```

Conferência:

```js
db.oscar_indicados.aggregate([
  { $match: { ano_cerimonia: { $gte: 2025 } } },
  { $group: { _id: "$ano_cerimonia", indicacoes: { $sum: 1 }, vencedores: { $sum: "$vencedor" } } },
  { $sort: { _id: 1 } }
])
```

R: 245 registros inseridos

| ano_cerimonia | indicações | vencedores |
|---|---|---|
| 2025 | 120 | 23 |
| 2026 | 125 | 25 |

Total na coleção: 11134 registros e 98 cerimônias. Em 2026 aparece a categoria nova `CASTING` (estreou na 98ª) e houve empate em `SHORT FILM (Live Action)` (`The Singers` e `Two People Exchanging Saliva`), por isso 25 vencedores em 24 categorias. Vencedores principais: Anora (2025) e One Battle After Another (2026).

---

## Nível 2: Explorando Categorias

**2.1 Indicações por categoria**

```js
db.oscar_indicados.aggregate([
  { $group: { _id: "$categoria", total: { $sum: 1 } } },
  { $sort: { total: -1 } }
])
```

R: 116 linhas. As 10 primeiras:

| categoria | total |
|---|---|
| DIRECTING | 479 |
| FILM EDITING | 460 |
| ACTOR IN A SUPPORTING ROLE | 450 |
| ACTRESS IN A SUPPORTING ROLE | 450 |
| BEST PICTURE | 391 |
| DOCUMENTARY (Short Subject) | 378 |
| CINEMATOGRAPHY | 348 |
| DOCUMENTARY (Feature) | 345 |
| FOREIGN LANGUAGE FILM | 315 |
| ART DIRECTION | 307 |

**2.2 Categoria com mais indicações**

```js
db.oscar_indicados.aggregate([
  { $group: { _id: "$categoria", total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 1 }
])
```

R: DIRECTING, com 479 indicações

**2.3 Categoria com menos indicações**

```js
db.oscar_indicados.aggregate([
  { $group: { _id: "$categoria", total: { $sum: 1 } } },
  { $match: { total: 1 } }
])
```

R: o mínimo é 1 indicação e quatro categorias empatam: `SPECIAL ACHIEVEMENT AWARD (Sound Effects)`, `SPECIAL ACHIEVEMENT AWARD (Sound Editing)`, `GORDON E. SAWYER AWARD` e `AWARD OF COMMENDATION`

**2.4 A partir de que ano "ACTRESS" deixou de existir?**

```js
db.oscar_indicados.find(
  { categoria: "ACTRESS" },
  { _id: 0, ano_cerimonia: 1, cerimonia: 1 }
).sort({ ano_cerimonia: -1 }).limit(1)
```

R: a última cerimônia com `ACTRESS` foi 1976 (48ª). A categoria deixou de existir a partir de 1977, quando passou a se chamar `ACTRESS IN A LEADING ROLE`.

**2.5 Categorias da primeira cerimônia que não existem mais**

Considerei que "existe hoje" quem aparece em alguma cerimônia de 2020 em diante.

```js
const primeiro = db.oscar_indicados.find().sort({ ano_cerimonia: 1 }).limit(1).next().ano_cerimonia
const antigas = db.oscar_indicados.distinct("categoria", { ano_cerimonia: primeiro })
const atuais = db.oscar_indicados.distinct("categoria", { ano_cerimonia: { $gte: 2020 } })
antigas.filter(c => !atuais.includes(c))
```

R: 12 categorias: `ACTOR`, `ACTRESS`, `ART DIRECTION`, `DIRECTING (Comedy Picture)`, `DIRECTING (Dramatic Picture)`, `ENGINEERING EFFECTS`, `OUTSTANDING PICTURE`, `SPECIAL AWARD`, `UNIQUE AND ARTISTIC PICTURE`, `WRITING (Adaptation)`, `WRITING (Original Story)` e `WRITING (Title Writing)`. Várias só mudaram de nome (ACTOR, ACTRESS, ART DIRECTION, OUTSTANDING PICTURE).

**2.6 Categorias com "DIRECTING"**

```js
db.oscar_indicados.distinct("categoria", { categoria: /DIRECTING/ })
```

R: 3 categorias: `DIRECTING`, `DIRECTING (Comedy Picture)` e `DIRECTING (Dramatic Picture)`

---

## Nível 3: Atores e Atrizes Famosos

### Natalie Portman

Como `nome_do_indicado` às vezes traz mais de um nome (produtores), busquei com regex em vez de igualdade.

**3.1 Quantas vezes foi indicada?**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Natalie Portman/ })
```

R: 3

**3.2 Quantos Oscars ganhou?**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Natalie Portman/, vencedor: 1 })
```

R: 1

**3.3 Anos e filmes**

```js
db.oscar_indicados.find(
  { nome_do_indicado: /Natalie Portman/ },
  { _id: 0, ano_cerimonia: 1, nome_do_filme: 1 }
).sort({ ano_cerimonia: 1 })
```

R: 2005 (Closer), 2011 (Black Swan) e 2017 (Jackie). Anos são os da cerimônia.

**3.4 Todas as indicações**

```js
db.oscar_indicados.find(
  { nome_do_indicado: /Natalie Portman/ },
  { _id: 0, ano_cerimonia: 1, categoria: 1, nome_do_filme: 1, vencedor: 1 }
).sort({ ano_cerimonia: 1 })
```

R:

| ano | categoria | filme | venceu |
|---|---|---|---|
| 2005 | ACTRESS IN A SUPPORTING ROLE | Closer | não |
| 2011 | ACTRESS IN A LEADING ROLE | Black Swan | sim |
| 2017 | ACTRESS IN A LEADING ROLE | Jackie | não |

### Viola Davis

**3.5 Indicações**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Viola Davis/ })
```

R: 4

**3.6 Oscars ganhos**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Viola Davis/, vencedor: 1 })
```

R: 1 (Fences, 2017)

**3.7 Filmes**

```js
db.oscar_indicados.distinct("nome_do_filme", { nome_do_indicado: /Viola Davis/ })
```

R: Doubt, Fences, Ma Rainey's Black Bottom e The Help

### Amy Adams

**3.8 Já ganhou algum Oscar?**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Amy Adams/, vencedor: 1 })
```

R: 0, nunca ganhou

**3.9 Indicações sem ganhar**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Amy Adams/, vencedor: 0 })
```

R: 6 (Junebug, Doubt, The Fighter, The Master, American Hustle e Vice)

### Denzel Washington

**3.10 Já ganhou algum Oscar?**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Denzel Washington/, vencedor: 1 }) > 0
```

R: true

**3.11 Quantas indicações?**

```js
db.oscar_indicados.countDocuments({ nome_do_indicado: /Denzel Washington/ })
```

R: 10. São 9 como ator e 1 como produtor de Fences (Melhor Filme, 2017).

**3.12 Oscars que ganhou**

```js
db.oscar_indicados.find(
  { nome_do_indicado: /Denzel Washington/, vencedor: 1 },
  { _id: 0, ano_cerimonia: 1, categoria: 1, nome_do_filme: 1 }
)
```

R:

| ano | categoria | filme |
|---|---|---|
| 1990 | ACTOR IN A SUPPORTING ROLE | Glory |
| 2002 | ACTOR IN A LEADING ROLE | Training Day |

---

## Nível 4: Vencedores Históricos

**4.1 Primeira vencedora de Melhor Atriz**

```js
db.oscar_indicados.find(
  { categoria: { $in: ["ACTRESS", "ACTRESS IN A LEADING ROLE"] }, vencedor: 1 },
  { _id: 0, nome_do_indicado: 1, nome_do_filme: 1, ano_cerimonia: 1 }
).sort({ ano_cerimonia: 1 }).limit(1)
```

R: Janet Gaynor, por 7th Heaven, em 1928

**4.2 Primeiro vencedor de Melhor Ator**

```js
db.oscar_indicados.find(
  { categoria: { $in: ["ACTOR", "ACTOR IN A LEADING ROLE"] }, vencedor: 1 },
  { _id: 0, nome_do_indicado: 1, nome_do_filme: 1, ano_cerimonia: 1 }
).sort({ ano_cerimonia: 1 }).limit(1)
```

R: Emil Jannings, por The Last Command, em 1928

**4.3 Quantos vencedores existem?**

```js
db.oscar_indicados.countDocuments({ vencedor: 1 })
```

R: 2513 (já com os 48 vencedores de 2025 e 2026)

**4.4 Filmes vencedores de Melhor Filme**

O prêmio teve vários nomes ao longo dos anos, então usei todos eles. Deixei `UNIQUE AND ARTISTIC PICTURE` de fora porque é outro prêmio, dado só em 1928.

```js
db.oscar_indicados.find(
  {
    categoria: { $in: [
      "OUTSTANDING PICTURE", "OUTSTANDING PRODUCTION", "OUTSTANDING MOTION PICTURE",
      "BEST MOTION PICTURE", "BEST PICTURE"
    ] },
    vencedor: 1
  },
  { _id: 0, ano_cerimonia: 1, nome_do_filme: 1 }
).sort({ ano_cerimonia: 1 })
```

R: 98 filmes, um por cerimônia:

1928 Wings; 1929 The Broadway Melody; 1930 All Quiet on the Western Front; 1931 Cimarron; 1932 Grand Hotel; 1933 Cavalcade; 1935 It Happened One Night; 1936 Mutiny on the Bounty; 1937 The Great Ziegfeld; 1938 The Life of Emile Zola; 1939 You Can't Take It with You; 1940 Gone with the Wind; 1941 Rebecca; 1942 How Green Was My Valley; 1943 Mrs. Miniver; 1944 Casablanca; 1945 Going My Way; 1946 The Lost Weekend; 1947 The Best Years of Our Lives; 1948 Gentleman's Agreement; 1949 Hamlet; 1950 All the King's Men; 1951 All about Eve; 1952 An American in Paris; 1953 The Greatest Show on Earth; 1954 From Here to Eternity; 1955 On the Waterfront; 1956 Marty; 1957 Around the World in 80 Days; 1958 The Bridge on the River Kwai; 1959 Gigi; 1960 Ben-Hur; 1961 The Apartment; 1962 West Side Story; 1963 Lawrence of Arabia; 1964 Tom Jones; 1965 My Fair Lady; 1966 The Sound of Music; 1967 A Man for All Seasons; 1968 In the Heat of the Night; 1969 Oliver!; 1970 Midnight Cowboy; 1971 Patton; 1972 The French Connection; 1973 The Godfather; 1974 The Sting; 1975 The Godfather Part II; 1976 One Flew over the Cuckoo's Nest; 1977 Rocky; 1978 Annie Hall; 1979 The Deer Hunter; 1980 Kramer vs. Kramer; 1981 Ordinary People; 1982 Chariots of Fire; 1983 Gandhi; 1984 Terms of Endearment; 1985 Amadeus; 1986 Out of Africa; 1987 Platoon; 1988 The Last Emperor; 1989 Rain Man; 1990 Driving Miss Daisy; 1991 Dances With Wolves; 1992 The Silence of the Lambs; 1993 Unforgiven; 1994 Schindler's List; 1995 Forrest Gump; 1996 Braveheart; 1997 The English Patient; 1998 Titanic; 1999 Shakespeare in Love; 2000 American Beauty; 2001 Gladiator; 2002 A Beautiful Mind; 2003 Chicago; 2004 The Lord of the Rings: The Return of the King; 2005 Million Dollar Baby; 2006 Crash; 2007 The Departed; 2008 No Country for Old Men; 2009 Slumdog Millionaire; 2010 The Hurt Locker; 2011 The King's Speech; 2012 The Artist; 2013 Argo; 2014 12 Years a Slave; 2015 Birdman or (The Unexpected Virtue of Ignorance); 2016 Spotlight; 2017 Moonlight; 2018 The Shape of Water; 2019 Green Book; 2020 Parasite; 2021 Nomadland; 2022 CODA; 2023 Everything Everywhere All at Once; 2024 Oppenheimer; 2025 Anora; 2026 One Battle After Another.

**4.5 Quantos filmes diferentes já ganharam o Oscar?**

```js
db.oscar_indicados.distinct("nome_do_filme", { vencedor: 1, nome_do_filme: { $ne: "NULL" } }).length
```

R: 1356 filmes (tirei o valor `"NULL"`, que não é filme)

---

## Nível 5: Análise de Indicações

Para "ator/atriz" usei todas as categorias que começam com `ACTOR` ou `ACTRESS` (inclui os nomes antigos).

**5.1 Atores/atrizes indicados mais de uma vez**

```js
db.oscar_indicados.aggregate([
  { $match: { categoria: /^(ACTOR|ACTRESS)/ } },
  { $group: { _id: "$nome_do_indicado", indicacoes: { $sum: 1 } } },
  { $match: { indicacoes: { $gt: 1 } } },
  { $sort: { indicacoes: -1 } }
])
```

R: 370 atores/atrizes. Os que mais aparecem: Meryl Streep (21), Katharine Hepburn (12), Jack Nicholson (12), Bette Davis (11), Spencer Tracy (9), Laurence Olivier (9), Paul Newman (9), Al Pacino (9), Denzel Washington (9).

**5.2 Maior número de indicações**

```js
db.oscar_indicados.aggregate([
  { $match: { categoria: /^(ACTOR|ACTRESS)/ } },
  { $group: { _id: "$nome_do_indicado", indicacoes: { $sum: 1 } } },
  { $sort: { indicacoes: -1 } },
  { $limit: 1 }
])
```

R: Meryl Streep, com 21 indicações

**5.3 Mais de 3 indicações e nunca ganharam**

```js
db.oscar_indicados.aggregate([
  { $match: { categoria: /^(ACTOR|ACTRESS)/ } },
  { $group: { _id: "$nome_do_indicado", indicacoes: { $sum: 1 }, vitorias: { $sum: "$vencedor" } } },
  { $match: { indicacoes: { $gt: 3 }, vitorias: 0 } },
  { $sort: { indicacoes: -1 } }
])
```

R: 27 nomes: Peter O'Toole (8), Glenn Close (8), Richard Burton (7), Deborah Kerr (6), Thelma Ritter (6), Amy Adams (6), Irene Dunne (5), Arthur Kennedy (5), Albert Finney (5), Annette Bening (5), Michelle Williams (5), Bradley Cooper (5), Charles Boyer (4), Barbara Stanwyck (4), Mickey Rooney (4), Claude Rains (4), Rosalind Russell (4), Agnes Moorehead (4), Montgomery Clift (4), Warren Beatty (4), Jane Alexander (4), Marsha Mason (4), Willem Dafoe (4), Ed Harris (4), Edward Norton (4), Saoirse Ronan (4) e Mark Ruffalo (4)

**5.4 Artistas indicados em categorias diferentes (ator e diretor)**

Se agrupar todas as categorias aparecem estúdios, compositores e filmes (por exemplo "Metro-Goldwyn-Mayer"), então limitei a atuação e direção. Atuação conta como um tipo só, para `ACTOR` e `ACTOR IN A LEADING ROLE` não parecerem categorias diferentes.

```js
db.oscar_indicados.aggregate([
  { $match: { categoria: /^(ACTOR|ACTRESS|DIRECTING)/ } },
  { $addFields: {
      tipo: { $cond: [{ $regexMatch: { input: "$categoria", regex: /^DIRECTING/ } }, "DIRETOR", "ATUACAO"] }
  } },
  { $group: { _id: "$nome_do_indicado", tipos: { $addToSet: "$tipo" } } },
  { $match: { "tipos.1": { $exists: true } } },
  { $sort: { _id: 1 } }
])
```

R: 15 artistas: Clint Eastwood, George Clooney, John Cassavetes, John Huston, Kenneth Branagh, Kevin Costner, Laurence Olivier, Lionel Barrymore, Orson Welles, Robert Redford, Roberto Benigni, Steve McQueen, Tim Robbins, Warren Beatty e Woody Allen

**5.5 Indicados com exatamente 1 indicação**

```js
db.oscar_indicados.aggregate([
  { $group: { _id: "$nome_do_indicado", n: { $sum: 1 } } },
  { $match: { n: 1 } },
  { $count: "total" }
])
```

R: 5718. Obs.: o campo mistura pessoas e textos de crédito (ex.: "Producers: ..."), então é o total de valores distintos do campo.

**5.6 Maior número de indicados em um único ano**

```js
db.oscar_indicados.aggregate([
  { $group: { _id: "$ano_cerimonia", total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 1 }
])
```

R: 1943, com 186 registros (1942 e 1944 vêm logo atrás, com 183 e 182)

---

## Nível 6: Análise de Filmes

### Toy Story

**6.1 Anos em que a série ganhou Oscars**

```js
db.oscar_indicados.find(
  { nome_do_filme: /^Toy Story/, vencedor: 1 },
  { _id: 0, ano_cerimonia: 1, nome_do_filme: 1, categoria: 1 }
).sort({ ano_cerimonia: 1 })
```

R: 2011 (Toy Story 3: Animated Feature Film e Music (Original Song)) e 2020 (Toy Story 4: Animated Feature Film)

**6.2 Total de indicações**

```js
db.oscar_indicados.countDocuments({ nome_do_filme: /^Toy Story/ })
```

R: 11

**6.3 Categorias**

```js
db.oscar_indicados.aggregate([
  { $match: { nome_do_filme: /^Toy Story/ } },
  { $group: { _id: { filme: "$nome_do_filme", categoria: "$categoria" } } },
  { $sort: { "_id.filme": 1 } }
])
```

R:

| filme | categorias |
|---|---|
| Toy Story | Music (Original Musical or Comedy Score), Music (Original Song), Writing (Screenplay Written Directly for the Screen) |
| Toy Story 2 | Music (Original Song) |
| Toy Story 3 | Animated Feature Film, Best Picture, Music (Original Song), Sound Editing, Writing (Adapted Screenplay) |
| Toy Story 4 | Animated Feature Film, Music (Original Song) |

### Crash

Filtrei por `nome_do_filme: "Crash"` exato, para não pegar "Crash Dive" nem "Crashing the Water Barrier".

**6.4 Edição em que concorreu**

```js
db.oscar_indicados.distinct("ano_cerimonia", { nome_do_filme: "Crash" })
```

R: 2006 (78ª cerimônia, filmes de 2005)

**6.5 Quantas indicações?**

```js
db.oscar_indicados.countDocuments({ nome_do_filme: "Crash" })
```

R: 6

**6.6 Ganhou Melhor Filme?**

```js
db.oscar_indicados.find(
  { nome_do_filme: "Crash", categoria: "BEST PICTURE" },
  { _id: 0, ano_cerimonia: 1, categoria: 1, vencedor: 1 }
)
```

R: Sim (`vencedor: 1`). Também ganhou Film Editing e Writing (Original Screenplay), 3 Oscars no total.

### Central do Brasil

**6.7 O filme aparece na base?**

Os títulos estão em inglês, então procurei pelos dois nomes.

```js
db.oscar_indicados.find({ nome_do_filme: /Central (Station|do Brasil)/ })
```

R: Sim, como "Central Station" (1999, 71ª cerimônia)

**6.8 Quantas indicações?**

```js
db.oscar_indicados.countDocuments({ nome_do_filme: /Central (Station|do Brasil)/ })
```

R: 2: Fernanda Montenegro (Actress in a Leading Role) e Brasil em Foreign Language Film. Nenhuma venceu.

---

## Nível 7: Operações de Atualização

**7.1 Converter `vencedor` para booleano**

Na base o campo já não é string, e sim número 0/1, então a conversão é de número para booleano.

```js
db.oscar_indicados.updateMany(
  { vencedor: { $in: [0, 1] } },
  [{ $set: { vencedor: { $toBool: "$vencedor" } } }]
)

// conferindo os tipos
db.oscar_indicados.aggregate([{ $group: { _id: { $type: "$vencedor" }, total: { $sum: 1 } } }])
```

R: 11134 documentos modificados. Depois da conversão todos ficam do tipo `bool`.

**7.2 Três filmes que nunca foram indicados**

Primeiro conferi que nenhum deles existe na base:

```js
db.oscar_indicados.countDocuments({ nome_do_filme: { $in: ["Heat", "Zodiac", "Paris, Texas"] } })   // 0
```

```js
db.oscar_indicados.insertMany([
  { id_registro: 11135, ano_filmagem: 1995, ano_cerimonia: 1996, cerimonia: 68, categoria: "DIRECTING", nome_do_indicado: "Michael Mann", nome_do_filme: "Heat", vencedor: false, ficticio: true },
  { id_registro: 11136, ano_filmagem: 2007, ano_cerimonia: 2008, cerimonia: 80, categoria: "DIRECTING", nome_do_indicado: "David Fincher", nome_do_filme: "Zodiac", vencedor: false, ficticio: true },
  { id_registro: 11137, ano_filmagem: 1984, ano_cerimonia: 1985, cerimonia: 57, categoria: "CINEMATOGRAPHY", nome_do_indicado: "Robby Müller", nome_do_filme: "Paris, Texas", vencedor: false, ficticio: true }
])
```

R: 3 registros inseridos. Usei o campo extra `ficticio: true` para separar dos dados reais.

**7.3 Nova categoria "BEST INTERNATIONAL FEATURE FILM"**

Vencedores das cerimônias de 2020 a 2024, no padrão da base (país em `nome_do_indicado`). A base já tem a categoria `INTERNATIONAL FEATURE FILM`, esta é uma categoria nova, separada.

```js
db.oscar_indicados.insertMany([
  { id_registro: 11138, ano_filmagem: 2019, ano_cerimonia: 2020, cerimonia: 92, categoria: "BEST INTERNATIONAL FEATURE FILM", nome_do_indicado: "South Korea", nome_do_filme: "Parasite", vencedor: true },
  { id_registro: 11139, ano_filmagem: 2020, ano_cerimonia: 2021, cerimonia: 93, categoria: "BEST INTERNATIONAL FEATURE FILM", nome_do_indicado: "Denmark", nome_do_filme: "Another Round", vencedor: true },
  { id_registro: 11140, ano_filmagem: 2021, ano_cerimonia: 2022, cerimonia: 94, categoria: "BEST INTERNATIONAL FEATURE FILM", nome_do_indicado: "Japan", nome_do_filme: "Drive My Car", vencedor: true },
  { id_registro: 11141, ano_filmagem: 2022, ano_cerimonia: 2023, cerimonia: 95, categoria: "BEST INTERNATIONAL FEATURE FILM", nome_do_indicado: "Germany", nome_do_filme: "All Quiet on the Western Front", vencedor: true },
  { id_registro: 11142, ano_filmagem: 2023, ano_cerimonia: 2024, cerimonia: 96, categoria: "BEST INTERNATIONAL FEATURE FILM", nome_do_indicado: "United Kingdom", nome_do_filme: "The Zone of Interest", vencedor: true }
])
```

R: 5 registros inseridos

**7.4 Corrigir erros nos nomes dos filmes**

Primeiro procurei espaços sobrando no começo, no fim ou duplicados:

```js
db.oscar_indicados.find({ nome_do_filme: /^\s|\s$|\s{2,}/ }, { _id: 0, id_registro: 1, nome_do_filme: 1 })
```

R: 3 documentos, todos com espaço duplo: "Paramount News Issue #37 (Twentieth Anniversary Issue!  1927.....1947)", "Volcano:  An Inquiry into the Life and Death of Malcolm Lowry" e "You Are Free  (Ihr Zent Frei)"

```js
db.oscar_indicados.updateMany(
  { nome_do_filme: /^\s|\s$|\s{2,}/ },
  [{ $set: { nome_do_filme: {
      $trim: { input: { $replaceAll: { input: "$nome_do_filme", find: "  ", replacement: " " } } }
  } } }]
)
```

R: 3 documentos modificados

**7.5 Remover registros com `nome_do_filme` NULL**

Na base o NULL aparece como texto `"NULL"`, então cobri os três casos.

```js
db.oscar_indicados.deleteMany({ nome_do_filme: { $in: [null, "NULL", ""] } })
```

R: 319 documentos removidos (prêmios sem filme, como Honorary Award e Jean Hersholt). Sobram 10823 registros.

---

## Nível 8: Análise Temporal

Depois do nível 7 a base tem 10823 registros.

**8.1 Indicações por década**

```js
db.oscar_indicados.aggregate([
  { $group: {
      _id: { $subtract: ["$ano_cerimonia", { $mod: ["$ano_cerimonia", 10] }] },
      total: { $sum: 1 }
  } },
  { $sort: { _id: 1 } }
])
```

R:

| década | total |
|---|---|
| 1920s | 67 |
| 1930s | 662 |
| 1940s | 1459 |
| 1950s | 1146 |
| 1960s | 1167 |
| 1970s | 1026 |
| 1980s | 1037 |
| 1990s | 1091 |
| 2000s | 1104 |
| 2010s | 1211 |
| 2020s | 853 |

**8.2 Década com mais indicações**

```js
db.oscar_indicados.aggregate([
  { $group: {
      _id: { $subtract: ["$ano_cerimonia", { $mod: ["$ano_cerimonia", 10] }] },
      total: { $sum: 1 }
  } },
  { $sort: { total: -1 } },
  { $limit: 1 }
])
```

R: anos 1940, com 1459 indicações

**8.3 Categorias únicas por década**

```js
db.oscar_indicados.aggregate([
  { $group: {
      _id: { $subtract: ["$ano_cerimonia", { $mod: ["$ano_cerimonia", 10] }] },
      categorias: { $addToSet: "$categoria" }
  } },
  { $project: { _id: 1, qtd_categorias: { $size: "$categorias" } } },
  { $sort: { _id: 1 } }
])
```

R:

| década | categorias |
|---|---|
| 1920s | 14 |
| 1930s | 25 |
| 1940s | 35 |
| 1950s | 37 |
| 1960s | 37 |
| 1970s | 46 |
| 1980s | 30 |
| 1990s | 27 |
| 2000s | 28 |
| 2010s | 26 |
| 2020s | 30 |

O número cresce até os anos 70 (46) e depois cai, em parte porque categorias com nomes variáveis foram sendo unificadas.

**8.4 Ano com mais indicações**

```js
db.oscar_indicados.aggregate([
  { $group: { _id: "$ano_cerimonia", total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 1 }
])
```

R: 1943, com 182 indicações (depois vêm 1944 com 180 e 1942 com 178)

**8.5 Taxa de crescimento: primeira década x última**

```js
db.oscar_indicados.aggregate([
  { $group: {
      _id: { $subtract: ["$ano_cerimonia", { $mod: ["$ano_cerimonia", 10] }] },
      total: { $sum: 1 }
  } },
  { $sort: { _id: 1 } },
  { $group: { _id: null, primeira: { $first: "$total" }, ultima: { $last: "$total" } } },
  { $project: {
      _id: 0, primeira: 1, ultima: 1,
      crescimento_pct: { $round: [{ $multiply: [{ $divide: [{ $subtract: ["$ultima", "$primeira"] }, "$primeira"] }, 100] }, 1] }
  } }
])
```

R: 1920s = 67 e 2020s = 853, crescimento de 1173,1%. Esse número exagera, porque a década de 1920 só tem 2 cerimônias e a de 2020 tem 7. Por cerimônia, são 33,5 contra 121,9 indicações (cerca de +264%).
