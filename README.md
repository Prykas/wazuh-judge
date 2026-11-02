# wazuh-judge

Avaliacao da Qualidade de Regras de Deteccao SIEM com LLMs
PPGES - UNIPAMPA - Laboratorio de Inteligencia Artificial Aplicada (LIAA)

Discente: Priscila Schafhauzer
Orientador: Prof. Diego Kreutz

## Perguntas de pesquisa

- RQ1: Em que medida LLMs concordam com avaliadores humanos? (kappa ponderado)
- RQ2: Qual a consistencia entre modelos diferentes? (alfa de Krippendorff)
- RQ3: Os LLMs conseguem diagnosticar defeitos conhecidos? (recall + mutation score)

## Condicoes experimentais

| Condicao | Descricao | Prazo |
|:---:|:---|:---:|
| E0 | Sem perfil organizacional | Este semestre |
| E1 | Perfil + rubrica, julgamento unico | Este semestre |
| E2 | E1 + evidencia pre-computada sem rotulo | Este semestre |
| E3 | Fluxo agentico com ferramentas (sandbox Wazuh) | Trabalho futuro |
| E4 | E3 + verificador de outra familia de modelo | Trabalho futuro |

## Status

- [x] Dataset de 20 regras (A7)
- [x] Rubrica de avaliacao C1-C5 (A8)
- [x] Rubrica do avaliador separada (A9-A11)
- [x] 6 regras de calibracao (A9-A11)
- [x] Tabela de decisao C4 (A9-A11)
- [x] Plano de analise pre-registrado (A9-A11)
- [ ] Piloto nas regras de calibracao
- [ ] Congelamento com hash (pre-registro)
- [ ] Rotulacao humana independente
- [ ] Execucao E0 + E1 + E2
- [ ] Metricas e relatorio

## Estrutura do repositorio

```
wazuh-judge/
├── config/                    <- Configuracoes do experimento
├── data/
│   ├── calibration/           <- 6 regras de calibracao (fora do dataset)
│   ├── items/                 <- Dataset de 20 regras (apos congelamento)
│   └── corpus/                <- Eventos do replay sem rotulos
├── rubric/
│   └── rubric_evaluator.md   <- Rubrica do avaliador (sem exemplos do dataset)
├── prompts/                   <- Templates dos prompts por condicao
├── schemas/                   <- Schema JSON das avaliacoes
└── docs/                      <- Documentacao das atividades
```

## Referencias

- SCHAFHAUZER, P. et al. Context-Aware SIEM Rule Generation with LLMs. SBSeg, 2026.
- ZHENG, L. et al. Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. NeurIPS, 2023.
- GWET, K. L. Handbook of Inter-Rater Reliability. 4th ed. Advanced Analytics, 2014.
