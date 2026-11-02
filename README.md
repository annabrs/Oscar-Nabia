Oscar com MongoDB: quase 100 anos de cinema em dados

Sobre o projeto:

O Oscar é o prêmio mais famoso do cinema e existe desde 1928. Neste projeto eu pego a história inteira da premiação, mais de 10 mil indicações, e uso MongoDB para fazer perguntas a esses dados: quem foi mais indicado, quais filmes mais ganharam, como a premiação mudou ao longo das décadas, quais categorias surgiram e quais deixaram de existir.

O trabalho nasceu de uma lista de exercícios de banco de dados NoSQL, mas acabou virando uma análise bem completa da história do Oscar. Cada pergunta virou uma consulta, e cada consulta tem a resposta ao lado.

A ideia:
Em vez de só olhar listas de vencedores, a proposta é entender a premiação pelos números:

- Quem são os artistas mais indicados, e quem foi indicado várias vezes sem nunca ganhar
- Como o número de indicações e de categorias evoluiu década por década
- A trajetória de filmes e franquias, como Toy Story e Crash
- O lugar do cinema brasileiro, com Central do Brasil (1999) e, mais recentemente, Ainda Estou Aqui (2025) e O Agente Secreto (2026)
- Marcos da história: primeiros vencedores, categorias extintas, empates e a estreia da categoria de Melhor Elenco em 2026

Alguns resultados:

- **Meryl Streep** é quem mais foi indicada em atuação: 21 vezes
- **Direção** é a categoria com mais indicações na história: 479
- **1943** foi o ano com mais indicações registradas: 182
- **Glenn Close** e **Peter O'Toole** têm 8 indicações cada e nenhuma vitória
- **Amy Adams** foi indicada 6 vezes e nunca levou o Oscar
- **Denzel Washington** tem 10 indicações e 2 vitórias (Glory e Training Day)
- **Toy Story** ganhou Oscars em 2011 e 2020, com 11 indicações no total
- **Crash** venceu Melhor Filme em 2006, com 6 indicações e 3 prêmios
- A década com mais indicações foram os **anos 1940**, com 1.459

O que a base tem:

Cada documento da coleção `oscar_indicados` é uma indicação:

```json
{
  "id_registro": 2,
  "ano_filmagem": 1927,
  "ano_cerimonia": 1928,
  "cerimonia": 1,
  "categoria": "ACTOR",
  "nome_do_indicado": "Emil Jannings",
  "nome_do_filme": "The Last Command",
  "vencedor": 1
}
```

A base original vem do repositório [gabaugusto/sample-databases](https://github.com/gabaugusto/sample-databases) e vai até a cerimônia de 2024. Eu acrescentei as cerimônias de **2025 e 2026**, que estão no arquivo `novos_2025_2026.json`.


O que está no repositório:

| Arquivo | O que é |
|---|---|
| `README.md` | esta apresentação |
| [`respostas.md`](respostas.md) | os exercícios dos níveis 1 a 8, com as queries e as respostas |
| [`novos_2025_2026.json`](novos_2025_2026.json) | as indicações de 2025 e 2026 |

Tecnologias:

MongoDB (consultas, agregações e atualizações em lote), mongosh e MongoDB Compass.

Depois é só seguir as queries do [`respostas.md`](respostas.md) na ordem. A limpeza dos dados está descrita no começo dele.

## Autor

[Seu Nome](https://github.com/SEU_USUARIO)
