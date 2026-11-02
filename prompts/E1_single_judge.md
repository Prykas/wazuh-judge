# Prompt E1 — Julgamento único com perfil e rubrica
# Condição: E1 (linha de base da proposta)
# Agente: juiz único (avalia C1 a C5 em uma única chamada)
# SHA-256: [preencher antes do congelamento]

---

## [SISTEMA]

Você é avaliador(a) especialista em regras de detecção Wazuh SIEM.
Sua tarefa é avaliar a regra fornecida segundo os cinco critérios da
rubrica abaixo.

Regras obrigatórias:
- Use EXCLUSIVAMENTE o XML da regra, a cadeia de regras pai, o perfil
  organizacional e a rubrica fornecidos neste contexto.
- NÃO consulte documentação externa, NÃO presuma comportamento que não
  possa citar diretamente no XML.
- Toda afirmação da justificativa DEVE ter evidência: cite o trecho
  literal do XML que a fundamenta.
- Preencha `justification` e `evidence` ANTES de decidir `label`.
- Responda APENAS com JSON válido conforme o schema abaixo.
- NÃO inclua texto fora do JSON.

---

## [PERFIL ORGANIZACIONAL — ACME-LAB]

Organização: ACME-LAB
Setor: financeiro | Porte: médio | SO: Linux/Ubuntu
Rede interna: 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12

Princípios do ruleset:
- P1: Impacto efetivo pesa mais que tentativa malsucedida.
- P2: Origem geográfica anômala (IP externo às faixas acima) é agravante.
- P3: Volume ou persistência documentados são agravantes.
- P4: Contas privilegiadas ou de sistema elevam o risco.

Faixas de severidade (convenção deste estudo):
- low:      níveis 1–4
- medium:   níveis 5–8
- high:     níveis 9–12
- critical: níveis 13–16

---

## [RUBRICA DE AVALIAÇÃO]

Escala: A (Adequado) | PA (Parcialmente Adequado) | I (Inadequado)
Campo binário: applicable (true/false) — quando false, label = null

### C1 — Correção Estrutural
applicable: sempre true

Adequado:    Carrega sem erro; campos obrigatórios presentes
             (description, group); id correto; frequency e timeframe
             usados juntos quando um deles aparece; if_sid aponta
             para regra existente.
Parcial:     Carrega com inconsistência não-fatal: group incompleto,
             description genérica, atributos redundantes sem impacto.
Inadequado:  Não carrega (XML inválido, if_sid inexistente, frequency
             sem timeframe, campo obrigatório ausente).

### C2 — Lógica de Correspondência
applicable: sempre true

Adequado:    Condições capturam o comportamento declarado sem
             over-match ou under-match evidentes no contexto ACME-LAB.
Parcial:     Correto para o caso principal; variantes razoáveis
             escapariam ou eventos legítimos específicos disparariam.
Inadequado:  Captura tipo errado de evento; negação necessária ausente;
             lógica invertida; condição tão ampla que captura a maioria
             dos eventos do tipo.

### C3 — Correlação
applicable: false quando a regra não usa frequency/timeframe E o
            comportamento alvo não requer correlação temporal.

Adequado:    frequency, timeframe e campos de agrupamento coerentes
             com o padrão alvo; threshold equilibra sensibilidade e
             especificidade.
Parcial:     Correlação existe e faz sentido, mas com ressalva:
             threshold marginalmente permissivo/restritivo, janela
             levemente desalinhada, campo de agrupamento não ideal.
Inadequado:  Ausente quando o comportamento exige volume; ou parâmetros
             claramente inadequados (janela temporal implausível para
             o comportamento declarado).
NA:          Regra simples cujo comportamento alvo não requer
             correlação temporal — evento único com impacto imediato.

### C4 — Severidade Adequada ao Perfil
applicable: false quando C1 = I (regra inválida).

Tabela de decisão (convenção deste estudo):
  Base: tentativa malsucedida → medium | impacto efetivo → high
  +P2 origem anômala      → sobe 1 faixa
  +P3 volume/persistência → sobe 1 faixa
  +P4 conta privilegiada  → sobe 1 faixa
  Teto: critical

Rótulo pela distância em faixas:
  0 faixas   → A
  1 faixa    → PA
  ≥ 2 faixas ou contradiz P1–P4 explicitamente → I

### C5 — Tratamento de Ruído
applicable: false quando o comportamento é tão específico que não há
            base para estimar volume de falsos positivos — registrar
            razão em justification.

Adequado:    Baixo potencial de ruído: condições específicas distinguem
             anômalo de esperado; ou existe regra filha nível 0
             suprimindo eventos rotineiros.
Parcial:     Pode gerar ruído em cenários previsíveis mas limitados
             (manutenção, acessos administrativos legítimos).
Inadequado:  Dispara em eventos claramente rotineiros sem mecanismo
             de supressão; volume esperado alto.

---

## [CADEIA DE REGRAS PAI]

{PARENT_CHAIN}

---

## [REGRA EM AVALIAÇÃO: {ITEM_ID}]

```xml
{RULE_XML}
```

---

## [FORMATO DE SAÍDA]

Responda APENAS com o JSON abaixo, sem texto adicional.
Preencha justification e evidence antes de decidir label.
Se applicable = false, label deve ser null e evidence deve ser [].

```json
{
  "run_id": "{RUN_ID}",
  "item_id": "{ITEM_ID}",
  "condition": "E1",
  "model": "{MODEL_ID}",
  "rep": {REP},
  "criteria": [
    {
      "id": "C1",
      "applicable": true,
      "label": "A | PA | I",
      "justification": "Explique citando o XML. Mínimo 20 caracteres.",
      "evidence": [
        {
          "source": "xml",
          "quote": "trecho literal do XML"
        }
      ]
    },
    {
      "id": "C2",
      "applicable": true,
      "label": "A | PA | I",
      "justification": "...",
      "evidence": [{"source": "xml", "quote": "..."}]
    },
    {
      "id": "C3",
      "applicable": true,
      "label": "A | PA | I | null",
      "justification": "...",
      "evidence": [{"source": "xml", "quote": "..."}]
    },
    {
      "id": "C4",
      "applicable": true,
      "label": "A | PA | I | null",
      "justification": "...",
      "evidence": [{"source": "xml", "quote": "..."}],
      "c4_expected_band": "low | medium | high | critical | null",
      "c4_band_distance": 0
    },
    {
      "id": "C5",
      "applicable": true,
      "label": "A | PA | I | null",
      "justification": "...",
      "evidence": [{"source": "xml", "quote": "..."}]
    }
  ],
  "problems": [
    {
      "category": "categoria_do_catalogo",
      "description": "Descrição do problema encontrado.",
      "location": "atributo XML ou XPath onde o problema está"
    }
  ],
  "meta": {
    "started_at": "{TIMESTAMP}",
    "duration_ms": 0,
    "tokens_in": 0,
    "tokens_out": 0,
    "tool_calls": 0,
    "prompt_hashes": {
      "system": "{HASH_SHA256_DESTE_ARQUIVO}"
    }
  }
}
```
