# Rubrica do Avaliador — Wazuh Judge
**Versão:** 1.0 | **Data:** 2026-11-02  
**Projeto:** Avaliação da Qualidade de Regras de Detecção SIEM com LLMs  
**PPGES · UNIPAMPA · LIAA**

> ⚠️ **IMPORTANTE:** Este documento é a única versão entregue a avaliadores.
> Os exemplos aqui usam exclusivamente regras de calibração (fora do dataset).
> Nenhum exemplo pertence ao conjunto de regras que você irá avaliar.

---

## Instruções gerais

Você receberá, para cada avaliação, um **pacote de contexto** contendo:

1. Perfil organizacional ACME-LAB (texto fixo)
2. Faixas de severidade e princípios P1–P4
3. Cadeia de regras pai (XMLs referenciados por `if_sid` / `if_matched_sid`)
4. Esta rubrica completa
5. O XML da regra a ser avaliada

**Para cada critério:**
- Decida se é **aplicável** (`applicable: true/false`)
- Se aplicável, atribua um rótulo: **A**, **PA** ou **I**
- Escreva a **justificativa** citando o trecho exato do XML que fundamenta sua decisão
- Se houver problema, descreva-o no campo `problems`

**Formato de saída:** JSON conforme `evaluation.schema.json`

---

## Escala de avaliação

| Símbolo | Nome | Significado |
|:---:|:---:|:---|
| **A** | Adequado | A regra atende ao critério sem ressalva técnica relevante no contexto ACME-LAB |
| **PA** | Parcialmente Adequado | A regra atende ao critério com ressalva técnica identificável e justificável |
| **I** | Inadequado | A regra falha no critério de forma clara, com impacto operacional identificável |
| **NA** | Não aplicável | Critério não avaliável para esta regra — registrar `applicable: false` |

> **Regra de ouro:** toda afirmação da justificativa deve ter evidência.
> Cite o trecho literal do XML ou referencie uma chamada de ferramenta (call_id).
> Justificativas sem referência ao artefato são marcadas como insuficientes.

---

## C1 — Correção Estrutural

**Avaliado por:** verificação determinística do motor Wazuh (`wazuh-analysisd -t`)  
**applicable:** sempre `true` — toda regra tem estrutura avaliável

**Definição:** A regra é válida, carregável e usa os atributos do Wazuh corretamente:
campos obrigatórios presentes (`description`, `group`), sintaxe XML bem formada,
`if_sid` referenciando regras existentes, `frequency` e `timeframe` presentes juntos
quando um deles é usado.

| Rótulo | O que o avaliador observa |
|:---:|:---|
| **A** | Carrega sem erro; todos os campos obrigatórios presentes; `frequency`/`timeframe` usados corretamente; `if_sid` aponta para regra existente |
| **PA** | Carrega com inconsistência não-fatal: `group` incompleto, `description` genérica sem identificar o comportamento, ou atributos redundantes sem impacto funcional |
| **I** | Não carrega (XML inválido, `if_sid` para id inexistente, `frequency` sem `timeframe`, campo obrigatório ausente) |
| **NA** | — não se aplica, toda regra tem estrutura avaliável |

**Exemplo de C1 Adequado** — `CAL-03` (id 199003):
```xml
<rule id="199003" level="5">
  <if_sid>5716</if_sid>
  <srcip>!192.168.0.0/16</srcip>
  <description>sshd: falha de autenticação SSH de IP externo</description>
  <group>authentication_failed,external_source</group>
</rule>
```
> `if_sid` referencia regra existente (5716), `description` identifica o comportamento,
> `group` presente — carrega sem erro. **C1 = A**

**Exemplo de C1 Inadequado** — versão modificada para ilustração:
```xml
<rule id="199003" level="5" frequency="3">
  <if_sid>5716</if_sid>
  <description>falha ssh</description>
</rule>
```
> `frequency` presente sem `timeframe` — motor rejeita a regra.
> `group` ausente — campo obrigatório faltando. **C1 = I**

---

## C2 — Lógica de Correspondência

**Avaliado por:** LLM e avaliador humano  
**applicable:** sempre `true` — toda regra tem lógica de correspondência avaliável

**Definição:** A condição de correspondência (`if_sid`, `dstuser`, `srcip`, `negate`)
captura o comportamento pretendido sem casar demais (over-match) ou de menos
(under-match). A lógica é coerente com o comportamento declarado na `description`.

| Rótulo | O que o avaliador observa |
|:---:|:---|
| **A** | A combinação de `if_sid` e condições captura o comportamento declarado sem over-match ou under-match evidentes no contexto ACME-LAB |
| **PA** | Correto para o caso principal, mas variantes razoáveis escapariam (under-match) ou eventos legítimos específicos disparariam (over-match em cenário previsível) |
| **I** | Captura o tipo errado de evento (`if_sid` incorreto), ignora negação necessária, inverte a lógica, ou é tão ampla que captura a maioria dos eventos do tipo |
| **NA** | — não se aplica, toda regra tem lógica avaliável |

**Exemplo de C2 Adequado** — `CAL-03` (id 199004):
```xml
<rule id="199004" level="10" frequency="5" timeframe="120">
  <if_matched_sid>199003</if_matched_sid>
  <same_srcip />
  <description>sshd: brute force SSH de IP externo confirmado</description>
</rule>
```
> `if_matched_sid` aponta para regra que já filtra IPs externos (199003);
> `same_srcip` garante que as falhas vêm da mesma origem.
> Sem over-match (IPs internos já excluídos) e sem under-match (qualquer IP externo
> com 5 falhas dispara). **C2 = A**

**Exemplo de C2 Inadequado** — `CAL-05` modificado para ilustração:
```xml
<rule id="199006b" level="8" frequency="3" timeframe="30">
  <if_sid>5715</if_sid>
  <same_user />
  <description>sudo: múltiplas falhas de autenticação (possível brute force)</description>
</rule>
```
> `if_sid` 5715 corresponde a login SSH bem-sucedido — a descrição fala em falhas
> de sudo, mas a condição captura sucessos SSH. Tipo errado de evento. **C2 = I**

---

## C3 — Correlação

**Avaliado por:** LLM e avaliador humano  
**applicable:** `false` quando a regra não usa `frequency`/`timeframe` e o comportamento
alvo não requer correlação temporal — evento único com impacto efetivo imediato

**Definição:** Para regras com `frequency`/`timeframe`: os parâmetros de agregação
são plausíveis para o comportamento que a regra pretende detectar.
Para regras simples: a ausência de correlação é adequada ao tipo de evento.

| Rótulo | O que o avaliador observa |
|:---:|:---|
| **A** | `frequency`, `timeframe` e campos de agrupamento coerentes com o padrão alvo; threshold equilibra sensibilidade e especificidade |
| **PA** | Correlação existe e faz sentido, mas com ressalva: threshold marginalmente permissivo ou restritivo, janela levemente desalinhada, campo de agrupamento não ideal mas funcional |
| **I** | Correlação ausente quando o comportamento exige volume; ou parâmetros claramente inadequados: janela temporal implausível para o comportamento declarado |
| **NA** | Regra simples e o comportamento alvo não requer correlação temporal |

**Exemplo de C3 Adequado** — `CAL-03` (id 199004):
```xml
<rule id="199004" level="10" frequency="5" timeframe="120">
  <if_matched_sid>199003</if_matched_sid>
  <same_srcip />
```
> 5 falhas em 120 segundos do mesmo IP é plausível para detectar automação
> sem capturar erros humanos isolados. `same_srcip` é o campo correto. **C3 = A**

**Exemplo de C3 Inadequado** — `CAL-05` (id 199006):
```xml
<rule id="199006" level="8" frequency="3" timeframe="30">
  <if_sid>5401</if_sid>
  <same_user />
  <description>sudo: múltiplas falhas ... (possível brute force em credenciais sudo)</description>
```
> `timeframe=30s` é implausível para brute force em sudo — atacante que testa
> senhas lentamente para evitar bloqueio levaria minutos entre tentativas.
> Janela de 30s captura apenas erros de digitação consecutivos do próprio usuário.
> Timeframe adequado seria 300–600s. **C3 = I**

**Exemplo de C3 NA** — `CAL-04` (id 199005):
```xml
<rule id="199005" level="13">
  <if_sid>5715</if_sid>
  <user>root</user>
  <description>sshd: login root bem-sucedido de IP externo</description>
```
> Regra simples sem `frequency`/`timeframe`. Login root externo bem-sucedido
> é impacto efetivo imediato — não requer correlação de volume para ser relevante.
> **C3 = NA** (`applicable: false`)

---

## C4 — Severidade Adequada ao Perfil

**Avaliado por:** LLM e avaliador humano  
**applicable:** `false` quando C1 = I (regra inválida não pode ter severidade avaliada)

**Definição:** O nível (escala 0–16) é compatível com o impacto do evento no contexto
ACME-LAB e com os quatro princípios do perfil.

### Tabela de decisão C4

| Tipo de evento | Faixa base | + P2 geo anômala | + P3 volume/persist. | + P4 conta priv. | Teto |
|:---:|:---:|:---:|:---:|:---:|:---:|
| Tentativa malsucedida | medium (5–8) | +1 faixa | +1 faixa | +1 faixa | critical |
| Impacto efetivo | high (9–12) | +1 faixa | +1 faixa | +1 faixa | critical |

> Faixas: low 1–4 · medium 5–8 · high 9–12 · critical 13–16  
> Esta é uma **convenção deste estudo**, não a classificação oficial do Wazuh.

### Rótulo pela distância em faixas

| Distância | Rótulo |
|:---:|:---:|
| 0 faixas | **A** |
| 1 faixa | **PA** |
| ≥ 2 faixas ou contradiz P1–P4 explicitamente | **I** |

| Rótulo | O que o avaliador observa |
|:---:|:---|
| **A** | Nível na faixa que os quatro princípios implicam; distância = 0 |
| **PA** | Nível a uma faixa de distância do esperado, com agravante ou atenuante identificável |
| **I** | Nível a duas ou mais faixas, ou contradiz explicitamente um princípio do perfil |
| **NA** | Regra estruturalmente inválida (C1 = I) |

**Exemplo de C4 Adequado** — `CAL-04` (id 199005):
```xml
<rule id="199005" level="13">
  <if_sid>5715</if_sid>
  <user>root</user>
  <srcip>!192.168.0.0/16</srcip>
```
> Tabela: impacto efetivo (login bem-sucedido) → high base.
> P2: IP externo (três faixas privadas negadas) → +1 = critical.
> P4: `user=root` → confirma critical. Nível 13 ∈ critical. Distância = 0. **C4 = A**

**Exemplo de C4 Parcialmente Adequado** — `CAL-06` (id 199007):
```xml
<rule id="199007" level="10">
  <if_sid>5715</if_sid>
  <srcip>!192.168.0.0/16</srcip>
  <user>!root</user>
```
> Impacto efetivo → high base. P2: IP externo → +1 = critical (13–16).
> Nível 10 ∈ high — 1 faixa abaixo do esperado. Distância = 1. **C4 = PA**

**Exemplo de C4 Inadequado** — `CAL-02` modificado para ilustração:
```xml
<rule id="199002b" level="3" frequency="6" timeframe="120">
  <if_matched_sid>5716</if_matched_sid>
  <description>múltiplas falhas SSH do mesmo IP</description>
```
> Tentativa correlacionada com P3 (volume: 6 falhas) → medium base +1 = high esperado.
> Nível 3 ∈ low — 2 faixas abaixo. Contradiz P3 explicitamente. **C4 = I**

---

## C5 — Tratamento de Ruído

**Avaliado por:** LLM e avaliador humano  
**applicable:** `false` quando o comportamento é tão específico que não há base para
estimar volume de falsos positivos — registrar razão no campo `justification`

**Definição:** A regra produz alertas para eventos que o perfil ACME-LAB considera
rotineiros e de baixo valor, sem mecanismo de supressão (regra filha nível 0).

| Rótulo | O que o avaliador observa |
|:---:|:---|
| **A** | Baixo potencial de ruído: condições específicas distinguem anômalo de esperado; ou existe regra filha nível 0 suprimindo eventos rotineiros |
| **PA** | Pode gerar ruído em cenários previsíveis mas limitados (manutenção, acessos administrativos legítimos) |
| **I** | Dispara em eventos claramente rotineiros sem mecanismo de supressão; volume esperado documentado |
| **NA** | Comportamento tão específico que não há base para estimar volume de falsos positivos |

**Exemplo de C5 Adequado** — `CAL-01` (id 199001):
```xml
<rule id="199001" level="3">
  <if_sid>5501</if_sid>
  <user>backup|nagios|zabbix|prometheus</user>
  <time>08:00-18:00</time>
```
> Restrição por lista de contas de serviço conhecidas E por horário comercial.
> O universo de eventos que dispara esta regra é genuinamente pequeno e esperado.
> Sem necessidade de nível 0. **C5 = A**

**Exemplo de C5 Inadequado** — `CAL-02` modificado para ilustração:
```xml
<rule id="199002c" level="3">
  <if_sid>5501</if_sid>
  <description>PAM: sessão aberta</description>
```
> Sem restrição de usuário ou horário — dispara para qualquer abertura de sessão PAM,
> incluindo logins rotineiros de contas de serviço e sessões CRON.
> Volume esperado muito alto sem mecanismo de supressão. **C5 = I**

---

## Formato de saída JSON

```json
{
  "run_id": "string",
  "item_id": "IT-XX",
  "condition": "E1",
  "model": "identificador-versionado",
  "rep": 1,
  "criteria": [
    {
      "id": "C1",
      "applicable": true,
      "label": "A",
      "justification": "A regra carrega sem erros; if_sid 5716 referencia regra existente; campos description e group presentes; frequency e timeframe usados em conjunto.",
      "evidence": [
        {
          "source": "xml",
          "quote": "<if_sid>5716</if_sid>"
        }
      ]
    },
    {
      "id": "C3",
      "applicable": false,
      "label": null,
      "justification": "Regra simples sem frequency/timeframe. Comportamento alvo é impacto efetivo imediato — não requer correlação temporal.",
      "evidence": []
    }
  ],
  "problems": [
    {
      "category": "severidade_incompativel_subestimacao",
      "description": "Nível 3 (low) para impacto efetivo de escalada de privilégio contradiz P1 e P4.",
      "location": "atributo level"
    }
  ],
  "meta": {
    "started_at": "2026-11-02T18:00:00Z",
    "duration_ms": 4200,
    "tokens_in": 1850,
    "tokens_out": 620,
    "tool_calls": 0,
    "prompt_hashes": {
      "system": "sha256-...",
      "rubric": "sha256-..."
    }
  }
}
```

---

## Regras de consistência obrigatórias

- Os cinco critérios (C1–C5) devem aparecer exatamente uma vez cada
- `applicable: false` implica `label: null`
- C1 é sempre `applicable: true`
- Se C1 = I, então C2 a C5 têm `applicable: false` (regra inválida não é avaliada)
- C3 pode ser NA apenas se a regra não tiver `frequency`
- Toda `evidence` com `source: xml` deve conter citação literal do XML da regra

---

*Rubrica do avaliador v1.0 — exemplos baseados exclusivamente em regras de calibração (CAL-01 a CAL-06), fora do dataset experimental.*
