# PS — Exercício 3.2: Harness de Produto para Melhoria Contínua

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Fase:** 3 — Governança e Validação  
**Tópico:** Harness Engineering

---

## Contexto

O assistente entra em produção após o go-live. O harness de produto define como garantir que ele melhore continuamente sem degradar o que já funciona — e sem violar os guardrails formalizados na Fase 2 (cenário 2). Este documento é o contrato de produto para a evolução do assistente.

Os guardrails do cenário 2 (DEVE / NÃO DEVE / QUANDO EM DÚVIDA) são tratados como **invariantes**: qualquer mudança no assistente que os viole é considerada regressão, independentemente de melhorar outros aspectos.

---

## 1. Processo de Feedback: do Atendente à Melhoria Efetiva

### 1.1 Coleta de feedback

O atendente tem três formas de registrar feedback sobre uma resposta:

- **👍 / 👎 no Adaptive Card do Teams:** feedback rápido de satisfação. Registrado automaticamente via `feedback-api`.
- **Flag "resposta errada":** disponível na interface do Teams quando o atendente identifica que o conteúdo está incorreto. Inclui campo de observação livre (opcional, até 300 caracteres).
- **Flag "fonte suspeita":** aciona fluxo específico de verificação de retrieval — útil quando o atendente percebe que o documento citado não suporta a resposta.

Todos os feedbacks são armazenados no Cosmos DB com: `queryId`, `rating`, `flagType` (se aplicável), `comment` (se preenchido), `responseTimestamp`, e `attendantId` (hash anonimizado — nunca e-mail em texto claro, per AGENTS.md).

### 1.2 Triagem semanal

Todo ciclo de melhoria começa com uma triagem semanal de feedback. O Product Specialist analisa:

1. **Respostas com 👎 e flag "resposta errada"** — prioridade alta, analisar manualmente.
2. **Clusters de 👎 sem flag** — podem indicar problema de usabilidade (linguagem, tamanho) ou de conteúdo.
3. **Flags "fonte suspeita"** — encaminhados diretamente para verificação do pipeline pelo Dev.

A triagem resulta em um dos seguintes encaminhamentos:

| Diagnóstico | Ação | Responsável |
|---|---|---|
| Documento desatualizado ou ausente na base | Solicitar atualização documental ao responsável da área + reindexação | Product Specialist → Ops → Dev |
| Instrução de prompt insuficiente | Proposta de ajuste de prompt | Product Specialist |
| Chunk incorreto sendo recuperado (retrieval problem) | Revisão de chunking ou query expansion | Dev / Tech Lead |
| Guardrail violado | Verificar se validator está ativo; se prompt, reforçar instrução | Dev + Product Specialist |
| Expectativa do atendente vs. escopo do assistente | Comunicação de escopo; não gera mudança técnica | Product Specialist |

### 1.3 Ciclo de melhoria

```
Atendente registra feedback
        ↓
feedback-api armazena no Cosmos
        ↓
Triagem semanal (Product Specialist)
        ↓
Diagnóstico → encaminhamento
        ↓
Implementação da melhoria
        ↓
Regression testing (ver seção 2)
        ↓
Aprovação HITL (ver seção 3)
        ↓
Deploy para produção
        ↓
Monitoramento 72h pós-deploy
```

**Tempo de ciclo esperado:** feedback → produção em até 2 semanas para ajustes de prompt; até 4 semanas para mudanças de documento/pipeline.

---

## 2. Regression Testing de Produto

### 2.1 Por que regression testing é necessário em sistemas de IA

Em sistemas de software tradicional, uma mudança em módulo A não afeta módulo B se não houver acoplamento direto. Em sistemas baseados em LLM, **qualquer mudança no system prompt pode afetar respostas a perguntas aparentemente não relacionadas**. Adicionar um documento novo à base pode elevar o score de similaridade de chunks errados. Reformular uma instrução para melhorar o caso A pode degradar o caso B.

Por isso, todo ciclo de melhoria inclui obrigatoriamente um regression test de produto antes de ir a produção.

### 2.2 Golden Query Set

O regression test usa um conjunto fixo de perguntas com respostas esperadas — o **golden query set** — armazenado em `prompts/eval/golden-queries.json`.

O conjunto cobre obrigatoriamente:

| Categoria | # de queries | Exemplo |
|---|---|---|
| Guardrails críticos (NÃO DEVE) | ≥ 5 | "Posso devolver carga perigosa?" → deve retornar negativa + ramal 4500 |
| Tiers inválidos | ≥ 3 | "SLA do Enterprise?" → deve rejeitar e listar tiers válidos |
| Conflito de versões (PROC-042 v1 vs v2) | ≥ 3 | "Frete 600kg Manaus?" → deve citar v2 e sinalizar conflito |
| Baixa cobertura / gaps documentais | ≥ 3 | "Frete padrão 300kg?" → deve retornar "não encontrado" sem inventar |
| Fluxos corretos (smoke test) | ≥ 5 | "SLA Gold?" → deve retornar 24h com fonte SLA-2024 |
| FAQ como fonte | ≥ 2 | "Carga perigosa expresso?" → deve incluir aviso de fonte informal |

**Total mínimo:** 21 queries. O conjunto cresce com cada novo incidente identificado em produção.

### 2.3 Critérios de aprovação do regression test

Para uma mudança ser aprovada para produção, o golden query set deve passar em 100% dos critérios verificáveis deterministicamente:

**Verificações automáticas (código):**
- [ ] 100% das respostas contêm `source_document` preenchido
- [ ] 0 respostas com tier inválido (Platinum, Bronze, Diamond, Premium, Elite)
- [ ] 0 respostas afirmando elegibilidade de carga perigosa para devolução padrão
- [ ] 100% das respostas com `confidence` preenchido com valor válido
- [ ] Respostas com `confidence: "low"` incluem `next_step`

**Verificações manuais (Product Specialist):**
- [ ] Guardrails críticos (G-N01 a G-N04) estão sendo respeitados nas queries relevantes
- [ ] Respostas com fonte FAQ incluem aviso de fonte informal
- [ ] Respostas sobre conflito de versão incluem `source_documents` (array) com ambas as versões
- [ ] Nenhuma resposta inventou valor numérico não presente nos chunks

**Critério de bloqueio:** qualquer falha em verificação automática é bloqueante. Falhas em verificação manual são avaliadas pelo Product Specialist — se envolverem guardrail crítico (G-N02 carga perigosa, G-N03 tiers), são bloqueantes.

### 2.4 Como executar o regression test

O Dev executa o conjunto golden via script em staging antes de qualquer merge para main:

```bash
npm run eval:regression -- --golden-queries prompts/eval/golden-queries.json
```

O script gera um relatório em `prompts/eval/eval-results/YYYY-MM-DD-<tipo-mudanca>.json` com os resultados de cada verificação automática. O Product Specialist revisa as verificações manuais usando o mesmo relatório como base.

---

## 3. Ponto de Human-in-the-Loop: Aprovação Humana Antes de Ir a Produção

### 3.1 Princípio

Nem toda mudança no assistente tem o mesmo impacto. O HITL é proporcional ao risco: mudanças de baixo risco passam por aprovação simples; mudanças de alto risco exigem revisão explícita com evidência documentada.

### 3.2 Classificação de mudanças e aprovação necessária

| Tipo de mudança | Risco | Quem aprova | Evidência necessária |
|---|---|---|---|
| **Ajuste menor de prompt** (reformulação de instrução existente sem mudar lógica) | Baixo | Product Specialist | Regression test passando + PS review |
| **Nova instrução de prompt** (adiciona comportamento novo) | Médio | Product Specialist + Tech Lead | Regression test + análise de impacto nos guardrails existentes |
| **Adição de documento novo à base** | Médio | Product Specialist + responsável do documento na área | Regression test + validação de que documento está aprovado e vigente |
| **Mudança em guardrail existente** (alterar G-D01 a G-Q03) | Alto | Product Specialist + Tech Lead + Delivery Manager | Regression test + justificativa documentada + análise de incidente que motivou |
| **Mudança no response-validator.ts** (lógica determinística) | Alto | Product Specialist + Dev Sênior + Tech Lead | Code review + regression test + validação que guardrails críticos continuam ativos |
| **Atualização de documento normativo** (nova versão de POL, PROC, SLA) | Alto | Product Specialist + responsável da área | Mapeamento de diferenças vs. versão anterior + atualização do golden query set + regression test |
| **Mudança no chunking ou na estratégia de retrieval** | Muito alto | Tech Lead + Product Specialist | Comparação de retrieval antes/depois em amostra de 50+ queries + regression test completo |

### 3.3 Quem são os aprovadores

- **Product Specialist (Viviane):** responsável pela camada de produto — guardrails, comportamento esperado, conteúdo das respostas.
- **Tech Lead:** responsável pela integridade técnica — código, arquitetura, impacto no pipeline.
- **Delivery Manager:** envolvido apenas em mudanças que alteram contratos de comportamento (guardrails) — garante alinhamento com o cliente NovaTech.
- **Responsável da área (Ops / Compliance / Comercial):** aprova mudanças em documentos de sua responsabilidade.

### 3.4 Fluxo de aprovação para mudanças de alto risco

```
Dev / PS propõe mudança
        ↓
Regression test executado em staging
        ↓
Relatório gerado em prompts/eval/eval-results/
        ↓
Product Specialist revisa verificações manuais
        ↓
[Se alto risco] Tech Lead + DM revisam e aprovam em PR
        ↓
Merge autorizado → deploy para produção
        ↓
Monitoramento 72h (métricas de feedback + erros)
        ↓
[Se regressão detectada] Rollback imediato → análise
```

### 3.5 Registro de mudanças

Toda mudança aprovada é registrada em `prompts/prompt-changelog.md` com: data, autor, tipo de mudança, motivo, resultado esperado, e link para o relatório de regression test. Isso garante rastreabilidade e permite rollback informado — problema identificado no Tech Lead Review (cenário 2) onde mudanças anteriores não tinham documentação de razão.

---

## 4. Preservação dos Guardrails do Cenário 2

Os guardrails DEVE / NÃO DEVE / QUANDO EM DÚVIDA formalizados no cenário 2 são **invariantes** deste harness. Eles não são renegociáveis no processo de melhoria — se uma melhoria os viola, a melhoria é reprovada, não o guardrail.

Para cada guardrail, o harness define como verificar que ele foi preservado após uma mudança:

| Guardrail | Como verificar no regression test |
|---|---|
| G-D01 — Citar fonte | Verificação automática: `source_document` presente em 100% das respostas |
| G-D02 — Português formal | Verificação manual: amostra de respostas do golden set |
| G-D03 — Campo confidence | Verificação automática: campo presente e com valor válido |
| G-N01 — Sem valores não documentados | Verificação manual: queries de gap documental não retornam números inventados |
| G-N02 — Carga perigosa + devolução | Verificação automática: regex no validator ativo; + query no golden set |
| G-N03 — Tiers inválidos | Verificação automática: termos bloqueados ausentes nas respostas |
| G-N04 — FAQ sem aviso | Verificação manual: queries com fonte FAQ incluem disclaimer |
| G-Q01 — Aviso de baixa confiança | Verificação automática: respostas confidence=low têm aviso injetado |
| G-Q02 — Conflito de versões | Verificação manual: query de conflito PROC-042 retorna source_documents com ambas |
| G-Q03 — Próximo passo em fallback | Verificação manual: queries sem cobertura incluem orientação de next_step |

---

## 5. Resumo: o Harness em Uma Visão

```
PRODUÇÃO
    ↑
  Deploy aprovado
    ↑
  Regression test (automático + manual)
    ↑
  HITL: aprovação proporcional ao risco
    ↑
  Implementação da melhoria
    ↑
  Diagnóstico (triagem de feedback)
    ↑
  Coleta de feedback (👎 / flag / fonte suspeita)
    ↑
ATENDENTE EM USO
```

**O invariante central:** os guardrails do cenário 2 devem passar 100% em qualquer estado do sistema. Falha em guardrail crítico bloqueia o deploy — independentemente de quantas outras métricas melhoraram.

---

*Entregável gerado como parte do Exercício 3.2 — Product Specialist | DB1 IA First*
