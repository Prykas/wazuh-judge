# Pré-registro do Experimento — wazuh-judge
# Projeto: Avaliação da Qualidade de Regras de Detecção SIEM com LLMs
# PPGES · UNIPAMPA · LIAA
# Data de congelamento: 2026-11-08

---

## 1. Identificação

| Campo | Valor |
|---|---|
| Título | Avaliação da Qualidade de Regras de Detecção SIEM com LLMs |
| Autora | Priscila Schafhauzer |
| Orientador | Prof. Diego Kreutz |
| Programa | PPGES — UNIPAMPA |
| Laboratório | LIAA — Laboratório de Inteligência Artificial Aplicada |
| Repositório | https://github.com/Prykas/wazuh-judge |
| Data de congelamento | 2026-11-08 |

---

## 2. Perguntas de pesquisa

| RQ | Pergunta | Métrica principal |
|---|---|---|
| RQ1 | O quanto o julgamento do LLM concorda com o julgamento humano? | κ ponderado (Cohen) |
| RQ2 | O quanto os modelos concordam entre si? | α de Krippendorff ordinal |
| RQ3 | O modelo consegue identificar e localizar defeitos nas regras? | Recall, mutation score, precisão |

---

## 3. Artefatos congelados — hashes de commit (GitHub)

Estes hashes identificam o estado exato de cada artefato no momento do
pré-registro. Qualquer alteração posterior produz hash diferente e fica
registrada no histórico do repositório.

| Artefato | Caminho | Commit Hash |
|---|---|---|
| Rubrica do avaliador | rubric/rubric_evaluator.md | `ccf11dff5129eab7308142c70f528e85307584ed` |
| Prompt E1 | prompts/E1_single_judge.md | `609a6d7a3722fdcef881a7ff6529664502205bea` |
| Catálogo de defeitos | data/catalog.csv | `59d19c328e656579e6b1299e0c53640a6f19a0d3` |
| Manifesto do dataset | data/items/items.yaml | `b23900c52bc7a218dbca11f82f948cf7b68fe773` |
| Tabela de decisão C4 | config/severity_table.yaml | `7c1dcd74085b45583b95813d9be9026413d09e1d` |

---

## 4. Dataset

| Campo | Valor |
|---|---|
| Total de itens | 20 regras Wazuh |
| Nativas Wazuh (N1–N6) | 6 |
| Geradas SBSeg (S1–S6) | 6 |
| Mutantes (M1–M5) | 5 |
| Controles (C1–C3) | 3 |
| Localização | data/items/IT-01.xml a IT-20.xml |
| Manifesto | data/items/items.yaml (hash acima) |

---

## 5. Modelos avaliados

| Modelo | Interface | Condições |
|---|---|---|
| Gemini 2.5 Flash | Google Gemini (web) | E0, E1, E2 |
| GPT-6 | ChatGPT (web) | E0, E1, E2 |
| Claude Sonnet 5.5 | Claude.ai (web) | E0, E1, E2 |

**Nota:** execução via interface web — sem controle de temperatura ou versão
exata de modelo. Declarado como limitação explícita no artigo.

---

## 6. Condições experimentais

| Condição | Descrição | Contexto entregue ao modelo |
|---|---|---|
| E0 | Sem perfil organizacional | XML + rubrica |
| E1 | Com perfil organizacional (linha de base) | XML + rubrica + perfil ACME-LAB |
| E2 | Com evidência pré-computada sem rótulo | XML + rubrica + perfil + estatísticas do replay |

---

## 7. Protocolo de execução

- Sessão nova e isolada para cada avaliação (sem contexto acumulado)
- Busca web desativada durante a avaliação
- Prompt copiado integralmente do arquivo congelado (hash acima)
- Saída JSON copiada antes de qualquer análise
- Três repetições por regra por modelo por condição
- Ordem de apresentação das regras: aleatória, seed=20261101

---

## 8. Plano de análise pré-registrado

### 8.1 Agregação das repetições (RQ1 e RQ2)
- **Análise principal:** moda das k=3 repetições
- **Empate:** rótulo mais severo (I > PA > A)
- **Análise de sensibilidade:** por repetição individual

### 8.2 Critérios incluídos no κ
- **Incluídos:** C2, C3, C4, C5 (applicable=true nos dois lados)
- **Excluídos:** C1 — tratado como gate determinístico, reportado à parte
- **Critérios NA:** excluídos do denominador do κ para aquele item

### 8.3 Métricas por RQ

**RQ1 — Concordância LLM × humano:**
- κ ponderado de Cohen (pesos lineares) por critério e agregado
- IC 95% via bootstrap (2.000 reamostras de regras inteiras, seed=20261101)
- Piso: classe majoritária por critério
- Teto: κ humano × humano (calculado antes da consolidação do padrão-ouro)

**RQ2 — Consistência entre modelos:**
- α de Krippendorff ordinal (múltiplos avaliadores)
- Calculado por critério e agregado
- IC 95% via bootstrap (mesmos parâmetros)

**RQ3 — Capacidade de diagnóstico:**
- Recall de defeitos por categoria e por operador de mutação
- Mutation score: proporção de mutantes detectados sobre o total
- Precisão das alegações após adjudicação cega
- Acurácia de localização: atributo XML citado corresponde ao local real

### 8.4 Adjudicação de alegações (precisão — RQ3)
- Dois pesquisadores independentes e cegos classificam cada alegação
- Categorias: legítimo / alucinação / duvidoso
- Discordâncias resolvidas por consenso
- Concordância entre adjudicadores reportada (κ de Cohen)

### 8.5 Casamento problema × defeito (RQ3)
- **Estrito:** mesma categoria de defeito + mesmo atributo XML
- **Frouxo:** mesma categoria de defeito (atributo pode diferir)
- Ambos reportados no artigo

### 8.6 Significância prática
- κ ≥ 0,60 como limiar de concordância substancial (Landis & Koch, 1977)
- Diferenças entre condições reportadas com IC — sem teste de hipótese

---

## 9. Critérios de avaliação (C1–C5) — resumo

| Critério | O que avalia | applicable |
|---|---|---|
| C1 — Correção Estrutural | A regra carrega sem erro no motor Wazuh? | sempre true |
| C2 — Lógica de Correspondência | As condições capturam o comportamento declarado? | sempre true |
| C3 — Correlação | frequency/timeframe/agrupamento coerentes com o padrão alvo? | false quando evento único sem necessidade de correlação |
| C4 — Severidade Adequada ao Perfil | O level está correto pela tabela de decisão P1–P4? | false quando C1=I |
| C5 — Tratamento de Ruído | A regra gera volume aceitável de falsos positivos? | false quando comportamento específico demais |

Descrição completa e âncoras de calibração: `rubric/rubric_evaluator.md`
(hash: ccf11dff5129eab7308142c70f528e85307584ed)

---

## 10. Piloto de calibração — resumo dos resultados

Executado antes do congelamento com 6 regras de calibração × 3 modelos × 1
repetição (18 avaliações). Resultados registrados em
`runs/piloto-v1/evaluations.jsonl`.

| Critério | Acerto | Observação |
|---|---|---|
| C4 | 18/18 (100%) | Tabela de decisão validada |
| C1 | 17/18 (94%) | 1 divergência — ChatGPT/CAL-05 |
| C3 | 17/18 (94%) | 1 divergência — Gemini/CAL-02 |
| C2 | 15/18 (83%) | Divergência em over-match de negação |
| C5 | 13/18 (72%) | Maior fonte de divergência — ajustado na v1.1 |

Ajustes aplicados na rubrica v1.1 antes deste congelamento:
- C2: clarificação sobre condições de negação simples
- C3: nota sobre "baixa intensidade implica janela mais longa"
- C5: orientação sobre regras com group low_priority

---

## 11. O que NÃO está pré-registrado

Os seguintes elementos serão definidos durante a execução e reportados
com transparência no artigo:

- Identidade dos avaliadores humanos
- Resultado da rotulação humana e κ humano × humano
- Casos adjudicados como duvidosos
- Análises exploratórias post-hoc (claramente sinalizadas como tal)

---

*Este documento foi gerado em 2026-11-08 e deve ser publicado no
repositório antes do início da execução oficial do experimento.*

