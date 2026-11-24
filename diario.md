# Diário de Bordo — Operação Maré Alta (RA: 244902622)

## A1 — Painel do pátio
- **Exec 1 (Sem índice):** Estágios `SORT` → `COLLSCAN`. Varreu todos os 400.000 documentos sem usar chaves (`totalKeysExamined: 0`). Previsão confirmada.
- **Exec 2 (Índice na ordem do filtro `{ terminal: 1, classe: 1, ts: 1, peso_kg: -1 }`):** Estágios `LIMIT` → `FETCH` → `SORT` → `IXSCAN`. Examinou 10.021 chaves e 20 documentos.
  - *Nota/Ajuste de hipótese:* Esperávamos ler todos os documentos do intervalo, mas como `peso_kg` já fazia parte do índice, o MongoDB ordenou as 10.021 chaves na memória (`SORT`) e buscou (`FETCH`) apenas os 20 documentos finais.
- **Exec 3 (Índice ESR `{ terminal: 1, classe: 1, peso_kg: -1, ts: 1 }`):** Estágios `LIMIT` → `FETCH` → `IXSCAN` (sem `SORT` em memória). Leu apenas 20 documentos.

## A2 — Incidente no pátio
- **Sem índice:** Falhou conforme previsto (`MongoServerError[NoQueryExecutionPlans]: unable to find index for $geoNear query`).
- **Com índice `2dsphere` (`posicao_2dsphere`):**
  - **Estágios:** `LIMIT` → `GEO_NEAR_2DSPHERE` → `FETCH` → `IXSCAN` (sem `SORT` em memória).
  - **Métricas:** Retornou 10 registros (`nReturned: 10`), examinando 48 documentos e 72 chaves em `11 ms`.

## A3 — O TTL que não apaga
- **Diagnóstico:** O `getIndexes()` mostrou apenas o índice padrão `_id_` (o índice TTL não existia), e os 100.000 registros estavam com `registrado_em` salvo como `string` (nenhum como `date`).
- **Causa raiz:** Além da falta do índice, o monitor TTL do MongoDB roda a cada 60 segundos e só apaga documentos se o campo for do tipo `BSON Date` — textos ISO são ignorados silenciosamente.
- **Correção:** Criado índice `{ registrado_em: 1 }` com `expireAfterSeconds: 2592000` (30 dias) e documentada a necessidade de a aplicação gravar `new Date()` em vez de `.toISOString()`.

## A4 — Consulta coberta
- **Com `_id: 0` (Coberta):** Estágios `PROJECTION_COVERED` → `IXSCAN`. Retornou 15.908 registros lendo 15.908 chaves e **0 documentos** na coleção (`11 ms`).
- **Sem `_id: 0` (Não coberta):** Estágios `PROJECTION_SIMPLE` → `FETCH` → `IXSCAN`. Retornou os mesmos 15.908 registros, mas precisou buscar todos os **15.908 documentos** (`85 ms`).
- **Conclusão:** Como o `_id` vem por padrão e não está no índice `{ tipo: 1, terminal: 1, container: 1 }`, não incluir `_id: 0` obriga o banco a fazer `FETCH` para cada chave encontrada.
- **Limpeza:** Índices secundários removidos ao final da Parte A.

## B1 — Relatório gerencial
- **Resultado (Top 3 Terminais — Granel a partir de 2026-06-01):**
  1. `T2`: 9.966 mov. | Peso total: 160.431.503 kg | Médio: 16.097,88 kg | Maior: 30.998 kg
  2. `T5`: 10.044 mov. | Peso total: 160.113.588 kg | Médio: 15.941,22 kg | Maior: 30.996 kg
  3. `T4`: 10.021 mov. | Peso total: 159.856.073 kg | Médio: 15.952,11 kg | Maior: 30.996 kg
- **Evidência do `explain`:** O estágio `$match` usou `IXSCAN` no índice `classe_1_ts_1`, filtrando e enviando para o `$group` exatamente **49.855 documentos** (confirmado pelo `countDocuments`), reduzidos a 5 grupos de terminais em `91 ms`.

## B2 — Auditoria e correção de pipeline
- **Erros encontrados no pipeline original:**
  1. `$match` no final do pipeline, forçando varredura completa da coleção (`COLLSCAN`).
  2. `$project` inicial descartando o campo `peso_kg` antes do agrupamento.
  3. `$sum: "peso_kg"` sem o prefixo `$`, somando o texto fixo como `0`.
  4. `$group` agrupando por `{ terminal, classe }` em vez de apenas `$terminal`.
  5. `$limit: 3` aplicado antes do `$sort`, cortando documentos aleatórios antes de ordenar.
- **Correção:** Pipeline reordenado com `$match` inicial usando índice `{ classe: 1 }`, `$group` por `$terminal` com `$sum: "$peso_kg"`, `$sort: { kg_total: -1 }` e `$limit: 3`, validado com `IXSCAN` no `explain` e soma manual.
- **Limpeza:** Índices secundários removidos ao final da Parte B (`dropIndexes()`).

## C1 — Modelagem do grafo e sócios diretos da E22
- **Sócios diretos de `E22` (Empresa 22):**
  - Nenhum sócio direto encontrado (`0` registros retornados por `MATCH (x:Empresa)-[r:SOCIA_DE]->(s:Empresa {cnpj: 'E22'})`). Na estrutura gerada para o RA `244902622`, a `E22` atua apenas como investidora de outras empresas (`E4` e `E9`), sem sócios acima dela.
- **Diagrama de Estrutura (Mermaid):**
```mermaid
graph LR
    EmpresaA[Empresa] -->|SOCIA_DE {pct}| EmpresaB[Empresa]
    Container[Container] -->|PERTENCE_A| EmpresaB
    Container -->|PASSOU_POR {data}| Terminal[Terminal]
    ContainerTopo[Container] -->|EMPILHADO_SOBRE| ContainerBase[Container]
```
- **Legenda das direções:**
  - `(:Empresa)-[:SOCIA_DE {pct}]->(:Empresa)`: A empresa de origem detém participação societária (`pct %`) na empresa de destino.
  - `(:Container)-[:PERTENCE_A]->(:Empresa)`: O contêiner pertence à empresa proprietária.
  - `(:Container)-[:PASSOU_POR {data}]->(:Terminal)`: O contêiner passou pelo terminal na data registrada.
  - `(:Container)-[:EMPILHADO_SOBRE]->(:Container)`: O contêiner de origem está apoiado fisicamente em cima do contêiner de destino.

## C2 — Travessia societária e operacional (até 3 saltos de `E22`)
- **Total de ocorrências encontradas:** `0` registros (`No changes, no records`).
- **Análise:** Como a empresa sancionada `E22` não possui sócias na direção de entrada (`<-[:SOCIA_DE*1..3]-`), a busca não retorna sócios diretos ou indiretos com coincidência de terminal e data.

## C3 — Depuração da pilha de contêineres (`BASE_DA_PILHA`: `K28`)
- **Consulta com direção invertida (`(base)-[:EMPILHADO_SOBRE*1..]->(acima)`):** Retornou `0` registros (`No changes, no records`), porque `K28` é a base da pilha e a aresta `EMPILHADO_SOBRE` aponta sempre do contêiner de cima para o de baixo.
- **Consulta corrigida (`(acima)-[:EMPILHADO_SOBRE*1..]->(base)`):** Retornou os 3 contêineres apoiados acima de `K28` por ordem de altura:
  1. `K29` (nível 1)
  2. `K30` (nível 2)
  3. `K31` (nível 3)