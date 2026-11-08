# Rubrica do Avaliador — wazuh-judge
# Versão: v1.1 (pós-piloto de calibração)
# SHA-256: [preencher antes do congelamento]
# Alterações em relação à v1.0:
#   - C2: clarificação sobre condições de negação simples
#   - C3: exemplo explícito de "baixa intensidade implica janela mais longa"
#   - C5: orientação sobre regras com group low_priority

---

## Instruções gerais

Você é avaliador(a) especialista em regras de detecção Wazuh SIEM.
Avalie a regra fornecida segundo os cinco critérios abaixo.

Regras obrigatórias:
- Use EXCLUSIVAMENTE o XML da regra, a cadeia de regras pai, o perfil
  organizacional e esta rubrica.
- NÃO consulte documentação externa. NÃO presuma comportamento que não
  possa citar diretamente no XML.
- Toda afirmação da justificativa DEVE ter evidência: cite o trecho
  literal do XML que a fundamenta.
- Preencha `justification` e `evidence` ANTES de decidir `label`.
- Responda APENAS com JSON válido conforme o schema de saída.
- NÃO inclua texto fora do JSON.

---

## Escala de rótulos

| Rótulo | Significado |
|--------|-------------|
| A      | Adequado — sem problemas neste critério |
| PA     | Parcialmente Adequado — funciona para o caso principal, mas há ressalva |
| I      | Inadequado — há defeito real neste critério |
| null   | Não aplicável — campo `applicable` = false |

---

## Perfil organizacional — ACME-LAB

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

## C1 — Correção Estrutural

**applicable:** sempre true

| Rótulo | Critério |
|--------|----------|
| A      | Carrega sem erro; campos obrigatórios presentes (description, group); frequency e timeframe usados juntos quando um deles aparece; if_sid / if_matched_sid aponta para regra existente na cadeia fornecida. |
| PA     | Carrega com inconsistência não-fatal: group incompleto, description genérica, atributos redundantes sem impacto no carregamento. |
| I      | Não carrega: XML inválido, if_sid inexistente na cadeia fornecida, frequency sem timeframe ou vice-versa, campo obrigatório ausente. |

**Âncoras de calibração:**
- CAL-01 (199001): A — carrega corretamente, sem frequency/timeframe, if_sid=5715 presente na cadeia.
- CAL-04 (199004): A — estrutura simples, todos os campos obrigatórios presentes.

---

## C2 — Lógica de Correspondência

**applicable:** sempre true

| Rótulo | Critério |
|--------|----------|
| A      | Condições capturam o comportamento declarado sem over-match ou under-match evidentes no contexto ACME-LAB. |
| PA     | Correto para o caso principal; variantes razoáveis escapariam ou eventos legítimos específicos disparariam. |
| I      | Captura tipo errado de evento; negação necessária ausente; lógica invertida; condição tão ampla que captura a maioria dos eventos do tipo. |

**Nota sobre condições de negação simples:**
Condições de negação como `!root`, `!daemon` ou `!192.168.0.0/16` que cobrem
o universo declarado sem excesso evidente no contexto ACME-LAB devem ser
classificadas como **Adequado**, mesmo que outras contas ou endereços não
declarados na descrição também casem. A negação é o mecanismo natural do
Wazuh para esse tipo de exclusão e não constitui over-match per se.
Over-match existe quando o escopo adicional não declarado é substancial e
previsível no contexto ACME-LAB (ex: contas de serviço autenticando via SSH
externo sendo rotuladas como "comportamento esperado" sem restrição de origem).

**Âncoras de calibração:**
- CAL-03 (199003): A — if_matched_sid=100001 herda filtro de IP externo; same_srcip agrupa corretamente.
- CAL-04 (199004): A — user=root + srcip negado captura exatamente o comportamento declarado.
- CAL-06 (199007): A — user=!root é negação simples adequada; não constitui over-match.

---

## C3 — Correlação

**applicable:** false quando a regra não usa frequency/timeframe E o
comportamento alvo não requer correlação temporal (evento único com
impacto imediato).

| Rótulo | Critério |
|--------|----------|
| A      | frequency, timeframe e campos de agrupamento coerentes com o padrão alvo; threshold equilibra sensibilidade e especificidade. |
| PA     | Correlação existe e faz sentido, mas com ressalva: threshold marginalmente permissivo/restritivo, janela levemente desalinhada com o padrão declarado. |
| I      | Ausente quando o comportamento exige volume; ou parâmetros claramente inadequados para o padrão alvo. |
| NA     | Regra simples cujo comportamento alvo não requer correlação temporal — evento único com impacto imediato. |

**Nota sobre janela temporal implausível:**
Uma janela temporal muito curta para o padrão alvo é **Inadequado**, não
Parcial. Exemplo: timeframe=30s para detectar brute force lento em sudo —
ataques pacientes espaçam tentativas em intervalos maiores e jamais
atingiriam o threshold em 30 segundos.

**Nota sobre "baixa intensidade" ou "lento" na descrição:**
Quando a descrição declara explicitamente *baixa intensidade*, *lento* ou
equivalente, a janela temporal deve ser avaliada contra esse padrão declarado.
Uma janela de 120s pode ser curta para ataques que espaçam tentativas em
intervalos maiores, resultando em **PA** mesmo que cubra o caso principal de
brute force concentrado. O desalinhamento entre a janela e o padrão declarado
é o defeito — não a ausência de correlação.

**Âncoras de calibração:**
- CAL-03 (199003): A — frequency=5, timeframe=120s, same_srcip coerentes com brute force SSH externo.
- CAL-05 (199006): I — timeframe=30s implausível para brute force lento em sudo.
- CAL-02 (199002): PA — frequency=6, timeframe=120s razoável, mas descrição declara "baixa intensidade" e a janela é curta para ataques lentos.
- CAL-04 (199004): NA — evento único, impacto imediato, sem necessidade de correlação.

---

## C4 — Severidade Adequada ao Perfil

**applicable:** false quando C1 = I (regra inválida não pode ser avaliada).

**Tabela de decisão:**

| Tipo de evento        | Faixa base     | +P2 geo anômala | +P3 volume/persist. | +P4 conta priv. | Teto      |
|-----------------------|----------------|-----------------|---------------------|-----------------|-----------|
| Tentativa malsucedida | medium (5–8)   | +1 faixa        | +1 faixa            | +1 faixa        | critical  |
| Impacto efetivo       | high (9–12)    | +1 faixa        | +1 faixa            | +1 faixa        | critical  |

**Rótulo pela distância em faixas:**

| Distância | Rótulo |
|-----------|--------|
| 0 faixas  | A      |
| 1 faixa   | PA     |
| ≥ 2 faixas ou contradiz P1–P4 explicitamente | I |

**Como aplicar:**
1. Identifique o tipo de evento (tentativa ou impacto efetivo).
2. Verifique quais agravantes P2, P3, P4 estão explicitamente suportados pelo XML.
3. Calcule a faixa esperada somando os agravantes ao base (teto: critical).
4. Compare com a faixa configurada (level da regra).
5. A distância em faixas determina o rótulo.

**Âncoras de calibração:**
- CAL-04 (199004, level=13): A — impacto efetivo + P2 + P4 = critical; distância=0.
- CAL-02 (199002, level=7): PA — tentativa + P3 = high; medium configurado; distância=1.
- CAL-06 (199007, level=10): PA — impacto efetivo + P2 = critical; high configurado; distância=1.
- CAL-01 (199001, level=3): I — impacto efetivo + P4 = critical; low configurado; distância=3.

---

## C5 — Tratamento de Ruído

**applicable:** false quando o comportamento é tão específico que não há
base para estimar volume de falsos positivos a partir do XML — registrar
razão em justification.

| Rótulo | Critério |
|--------|----------|
| A      | Baixo potencial de ruído: condições específicas distinguem anômalo de esperado; ou existe regra filha nível 0 suprimindo eventos rotineiros. |
| PA     | Pode gerar ruído em cenários previsíveis mas limitados (manutenção, acessos administrativos legítimos, erros de digitação repetidos). |
| I      | Dispara em eventos claramente rotineiros sem mecanismo de supressão; volume esperado alto. |

**Nota sobre regras de baixa prioridade declarada:**
Regras que declaram explicitamente `low_priority` no group e cujo comportamento
alvo é registrado como esperado e rotineiro devem ser classificadas como
**Adequado** em C5, desde que as condições limitem adequadamente o escopo
do evento (lista fechada de contas, janela horária, tipo específico de evento).
O fato de não ter regra filha nível 0 não é defeito quando a própria regra
já declara sua natureza de baixa prioridade e o escopo é restrito.

**Nota sobre threshold baixo:**
Um threshold baixo (ex: 3 falhas em 30s) pode gerar ruído por erros legítimos
de digitação. Se o comportamento alvo é previsível em cenários rotineiros
(usuário legítimo errando senha), classifique como **PA**, não como A.

**Âncoras de calibração:**
- CAL-01 (199001): A — contas de serviço em horário comercial, escopo restrito, group=low_priority declarado.
- CAL-03 (199003): A — origem externa + volume: distingue anômalo de esperado efetivamente.
- CAL-04 (199004): NA — login root externo é específico demais para estimar FP.
- CAL-02 (199002): PA — sem distinção de srcip interno/externo; scripts internos com credencial errada podem disparar.

---

## Consistência obrigatória

- Se applicable = false → label = null e evidence = [].
- Se C1 = I → C4 applicable = false (regra inválida).
- Preencha justification e evidence antes de decidir label.
- Cada evidence.quote deve ser um trecho literal do XML fornecido.
- O campo problems lista defeitos encontrados; pode ser vazio [].
- Para C4: preencha sempre c4_expected_band e c4_band_distance quando applicable = true.

---

## Regras de calibração — resumo do gabarito

| CAL | Regra  | C1 | C2 | C3 | C4 | C5 | Defeito principal |
|-----|--------|----|----|----|----|----|--------------------|
| CAL-01 | 199001 | A | A | NA | I  | A  | C4: low para conta de serviço (distância 3) |
| CAL-02 | 199002 | A | A | PA | PA | PA | C3: janela curta para "baixa intensidade"; C4: 1 faixa abaixo |
| CAL-03 | 199003 | A | A | A  | PA | A  | C4: 1 faixa abaixo do esperado critical |
| CAL-04 | 199004 | A | A | NA | A  | NA | Sem defeito — controle correto |
| CAL-05 | 199006 | A | A | I  | PA | A  | C3: timeframe=30s implausível para brute force lento |
| CAL-06 | 199007 | A | A | NA | PA | NA | C4: 1 faixa abaixo do esperado critical |

