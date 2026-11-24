# Operação Maré Alta — LogiPorto

- **Aluno:** Lucas Peterson Aoki Mukai
- **RA:** 244902622

## Parâmetros Exclusivos da Prova
- **CLASSE_ALVO:** Granel
- **TERMINAL_ALVO:** T4
- **DATA_CORTE:** 2026-06-01
- **INCIDENTE:** -25.4780, -48.5130
- **CÓDIGO DE AUTENTICIDADE:** LP-244902622-20049-42924
- **EMPRESA_SANCIONADA:** E22
- **BASE_DA_PILHA:** K28

## Estrutura dos Arquivos
- `previsoes.md`: Hipóteses registradas antes da execução de cada questão (Partes A, B e C).
- `diario.md`: Resultados reais, planos de execução (`explain`), diagnósticos, diagramas e correções aplicadas.
- `mongo/logiporto_gerador.js`: Script gerador determinístico das coleções do MongoDB com o RA configurado.
- `neo4j/grafo_gerador.js`: Script gerador determinístico do grafo do Neo4j com o RA configurado.