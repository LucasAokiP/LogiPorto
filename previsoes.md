# Previsões — Operação Maré Alta (RA: 244902622)

## A1
- **Exec 1 (Sem índice):** O banco vai varrer todos os 400.000 documentos (`COLLSCAN`) e ordenar tudo na memória (`SORT`) antes de pegar os 20 primeiros (`LIMIT`).
- **Exec 2 (Índice na ordem do filtro `{ terminal: 1, classe: 1, ts: 1, peso_kg: -1 }`):** Vai usar o índice (`IXSCAN`) para filtrar `terminal: "T4"` e `classe: "Granel"`. Porém, como o intervalo de data (`ts`) vem antes da ordenação por `peso_kg`, ainda vai precisar ordenar as chaves na memória (`SORT`) antes do `LIMIT`.
- **Exec 3 (Índice ESR `{ terminal: 1, classe: 1, peso_kg: -1, ts: 1 }`):** Seguindo a ordem Equality-Sort-Range, o índice já entrega os dados ordenados por `peso_kg: -1`. Isso elimina o `SORT` em memória e examina apenas os 20 documentos do `LIMIT`.

## A2
- **Sem índice:** A consulta `$geoNear` vai falhar na hora, acusando a falta de um índice geoespacial (`2dsphere`).
- **Com índice `2dsphere`:** O plano vai usar `GEO_NEAR_2DSPHERE` com `IXSCAN` a partir das coordenadas `-25.4780, -48.5130`, trazendo os 10 registros mais próximos sem precisar ordenar na memória.

## A3
- **Diagnóstico do TTL:** A coleção `alertas` não está apagando os registros antigos porque o índice TTL não foi criado ou o campo `registrado_em` foi salvo como texto (`string` ISO) em vez de `BSON Date`, que é o único formato aceito pelo monitor TTL para calcular a expiração.

## A4
- **Com `_id: 0` (Consulta coberta):** Como todos os campos filtrados e retornados (`tipo`, `terminal`, `container`) estão dentro do índice `{ tipo: 1, terminal: 1, container: 1 }`, o MongoDB resolve tudo direto no índice (`PROJECTION_COVERED` → `IXSCAN`), sem ler nenhum documento da coleção (`totalDocsExamined: 0`).
- **Sem `_id: 0` (Não coberta):** Como o `_id` vem por padrão e não faz parte do índice composto, o banco precisa buscar cada documento na coleção (`FETCH`), fazendo `totalDocsExamined` ser igual a `totalKeysExamined`.

## B1
- **Com índice `{ classe: 1, ts: 1 }`:** O `$match` inicial vai usar `IXSCAN` e enviar para o `$group` apenas os documentos da classe `"Granel"` a partir de `"2026-06-01"` — bem menos que os 400.000 documentos totais.

## B2
- **Pipeline original (com erro):** Vai varrer a coleção inteira (`COLLSCAN`), porque o `$match` ficou no final. Além disso, vai retornar `kg_total: 0` por três erros: o `$project` inicial descarta o campo `peso_kg`, o `$sum: "peso_kg"` tenta somar um texto fixo (faltou o `$`), e o `$limit: 3` corta os resultados antes do `$sort`.
- **Pipeline corrigido:** Com o `$match` no início usando o índice `{ classe: 1 }`, o `$group` apenas por `$terminal`, a soma correta (`$sum: "$peso_kg"`) e o `$sort` antes do `$limit: 3`, vai retornar os 3 maiores terminais de `"Granel"` usando `IXSCAN`.

## C2
- **Travessia societária e operacional:** Partindo de `E12` e seguindo as arestas `SOCIA_DE` no sentido de entrada (`<-[:SOCIA_DE*1..3]-`), encontramos todos os sócios diretos e indiretos em até 3 níveis. Depois, cruzamos os contêineres deles com os da `E12` que passaram pelo mesmo terminal na mesma data.

## C3
- **Depuração de travessias:** Em caminhos de tamanho variável (`*`), inverter a seta (`EMPILHADO_SOBRE` saindo de `K48` para frente) retorna 0 registros, porque `K48` está na base da pilha e as setas apontam sempre de cima para baixo.