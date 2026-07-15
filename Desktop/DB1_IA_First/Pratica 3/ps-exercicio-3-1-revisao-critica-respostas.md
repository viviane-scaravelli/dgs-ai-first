# PS — Exercício 3.1: Revisão Crítica das Respostas do Assistente

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Fase:** 3 — Governança e Validação  
**Tópico:** Revisão Crítica de Outputs de IA

---

## Parte 1 — Minha Avaliação (antes do Claude)

Fonte de verdade utilizada: Anexo A (documentação NovaTech) + Anexo B (chunks de referência do pipeline de RAG).

---

### Resposta #1 — "Qual o prazo de devolução para produtos standard?"

| Campo | Valor |
|---|---|
| Resposta | "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos." |
| Fonte citada | POL-001, seção 3.2 |
| Confiança | Alta |

**Avaliação: ✅ Correta**

O prazo de 7 dias úteis está em POL-001-A (seção 3.1) e o procedimento de abertura de chamado com fotos está em POL-001-C (seção 3.3). Observação menor: o assistente citou "seção 3.2" (que é a seção de exceções), sendo que o prazo consta na seção 3.1. É uma imprecisão no número de seção, mas o conteúdo da resposta é factualmente correto. A resposta é adequada para uso.

---

### Resposta #2 — "Meu cliente é Silver. Qual o prazo de resolução?"

| Campo | Valor |
|---|---|
| Resposta | "O cliente Silver tem resolução em até 48h." |
| Fonte citada | SLA-2024 |
| Confiança | Alta |

**Avaliação: ✅ Correta**

SLA-2024-B confirma: "Silver: resolução em até 48h úteis." Fonte correta, conteúdo correto, confiança alta justificada. Resposta adequada.

---

### Resposta #3 — "Posso devolver carga perigosa classe 3?"

| Campo | Valor |
|---|---|
| Resposta | "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor." |
| Fonte citada | POL-001, seção 3.2 |
| Confiança | Alta |

**Avaliação: ✅ Correta**

POL-001-B (seção 3.2) é exatamente a fonte correta para essa exceção. O assistente identificou corretamente que carga classe 3 (líquidos inflamáveis) está incluída nas classes 1-6 da ANTT. A orientação de escalar é adequada — embora o documento indique especificamente "ramal 4500 (Gestão de Riscos)", a resposta está no espírito correto. Guardrail G-N02 sendo respeitado. Resposta adequada.

---

### Resposta #4 — "Qual a política para carga danificada durante transporte?"

| Campo | Valor |
|---|---|
| Resposta | "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos." |
| Fonte citada | Nenhuma |
| Confiança | Alta |

**Avaliação: ❌ Incorreta — ALUCINAÇÃO**

**Tipo de erro: Alucinação com alta confiança sem fonte**

Não existe documento formal sobre política de carga danificada na base da NovaTech. O que existe é o FAQ-38, que é uma referência informal e não constitui política normativa. O assistente:
1. Apresentou detalhes específicos ("reembolso integral", "negligência da transportadora", "laudo técnico e fotos") como se fossem política formalizada.
2. Citou "Nenhuma" como fonte — o próprio assistente admite não ter fonte, mas manteve confiança alta e não acionou fallback.
3. Violou C-01 (nunca gerar valores/condições que não estejam nos chunks) e G-D01 (resposta sem source_document deveria ser bloqueada).

O FAQ-38 menciona investigação e reembolso "se comprovada responsabilidade", mas: (a) é fonte informal, (b) não especifica "reembolso integral", (c) diz que passa pelo Jurídico. O assistente expandiu e formalizou uma informação informal — isso é alucinação.

**Impacto:** Um atendente que usar essa resposta pode comprometer a empresa em situações de negociação de sinistros, criando expectativa de "reembolso integral" que pode não se concretizar.

---

### Resposta #5 — "Qual o SLA do cliente Enterprise?"

| Campo | Valor |
|---|---|
| Resposta | "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor." |
| Fonte citada | — |
| Confiança | Baixa |

**Avaliação: ✅ Correta (comportamento adequado)**

SLA-2024-A afirma explicitamente: "Não existem outros tiers além dos três listados." O assistente reconheceu o tier inválido, listou os válidos e orientou o próximo passo. É o comportamento esperado pelo guardrail G-N03 e o caso de fallback G-Q03. Confiança baixa é apropriada. Resposta adequada.

---

### Resposta #6 — "Posso enviar carga perigosa com frete expresso?"

| Campo | Valor |
|---|---|
| Resposta | "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada." |
| Fonte citada | FAQ-Atendimento, item 32 |
| Confiança | Alta |

**Avaliação: ❌ Incorreta — FONTE NÃO CONFIÁVEL**

**Tipo de erro: Fonte não confiável usada sem aviso + confiança alta indevida para tema de risco**

O FAQ-32 existe na base, e o assistente de fato o citou — então não é alucinação no sentido estrito. O problema é:
1. FAQ-32 é fonte informal ("item de FAQ de atendimento"), não documento normativo validado.
2. Carga perigosa é tema de alto risco regulatório. Informação sobre esse tema deveria exigir fonte formal (não FAQ).
3. O assistente não incluiu o aviso obrigatório exigido pelo guardrail G-N04: *"[Fonte informal — confirme com documento normativo ou supervisor antes de usar]"*.
4. Confiança "Alta" para informação de carga perigosa vinda de FAQ informal é incorreta — deveria ser no máximo "Média" com aviso explícito.

O FAQ-32 inclusive contém a ressalva "na prática, demora uns 2 dias para conseguir a autorização" — nuance que a resposta omitiu, aumentando o risco de expectativa errada do cliente.

**Impacto:** Atendente pode orientar envio de carga perigosa expressa sem alertar o cliente sobre a complexidade real do processo, gerando reclamação e risco de não-conformidade regulatória.

---

## Resumo da Avaliação Própria

| # | Avaliação | Tipo de erro |
|---|---|---|
| 1 | ✅ Correta | — (imprecisão menor na seção) |
| 2 | ✅ Correta | — |
| 3 | ✅ Correta | — |
| 4 | ❌ Incorreta | Alucinação com alta confiança sem fonte |
| 5 | ✅ Correta | — |
| 6 | ❌ Incorreta | Fonte não confiável sem aviso obrigatório |

---

## Parte 2 — Claude como Segundo Avaliador

*A seguir, apresento o resultado da avaliação do Claude e comparo com a minha.*

**Prompt enviado ao Claude:**

> "Você é um avaliador de qualidade de assistentes de IA. Avalie as 6 respostas abaixo contra a documentação NovaTech (Anexo A e B). Para cada resposta, classifique como correta, parcialmente correta ou incorreta. Justifique com base nos documentos. Identifique o tipo de erro quando houver."
> *(Contexto: chunks do Anexo B fornecidos como fonte de verdade)*

**Avaliação do Claude:**

| # | Avaliação Claude | Observações |
|---|---|---|
| 1 | ✅ Correta | Claude identificou a mesma imprecisão na seção (3.2 vs 3.1), mas considerou aceitável. Conteúdo correto. |
| 2 | ✅ Correta | Concordância total. |
| 3 | ✅ Correta | Claude destacou que "ramal 4500" seria mais preciso do que "supervisor", mas classificou como adequada. |
| 4 | ❌ Incorreta — Alucinação | Claude chegou à mesma conclusão: não há documento formal sobre política de danos. Destacou que FAQ-38 existe mas é informal e não contém os termos específicos usados na resposta. Flagrou a ausência de fonte com confiança alta como red flag crítico. |
| 5 | ✅ Correta | Concordância total. Claude elogiou o comportamento de fallback. |
| 6 | ❌ Incorreta — Fonte não confiável | Claude concordou com a classificação e acrescentou: o FAQ-32 contém uma nuance importante ("demora uns 2 dias") que foi omitida, agravando o problema. Também destacou que confiança "Alta" para tema de risco de carga perigosa proveniente de FAQ é inadequada independentemente do conteúdo. |

---

## Parte 3 — Comparação: Minha Avaliação vs. Claude

**Concordâncias:**
- Ambas as avaliações identificaram as mesmas 2 respostas problemáticas: #4 (alucinação) e #6 (fonte não confiável).
- Ambas classificaram #1, #2, #3 e #5 como corretas/adequadas.
- Ambas identificaram a imprecisão de seção na #1 sem considerá-la bloqueante.
- Concordamos no tipo de erro em cada caso (alucinação vs fonte não confiável).

**Divergências:**
- Claude acrescentou em #6 que a omissão da nuance "2 dias para autorização" agrava o problema — ponto que não havia na minha análise.
- Claude foi mais contundente sobre #3: sugeriu que "ramal 4500" deveria sempre ser mencionado em respostas sobre carga perigosa, não só "escalar ao supervisor". Concordo que é uma melhoria, mas não um erro de avaliação.
- Minha análise destacou a violação específica dos guardrails (G-N02, G-N04) por número; o Claude fez análise mais semântica, chegando à mesma conclusão por caminho diferente.

**Conclusão da comparação:** Alta convergência. O Claude funcionou como validação útil e acrescentou uma nuance relevante na resposta #6 que reforça o problema.

---

## Parte 4 — Propostas de Ajuste de Produto

### Problema #4 — Alucinação sobre política de danos

**Tipo de erro:** Alucinação — resposta gerada com alta confiança sem fonte na base  
**Causa raiz:** O assistente recebeu uma pergunta legítima sobre um tema não coberto por documento formal e, em vez de acionar o fallback, "completou" a resposta com informação plausível extrapolada do FAQ informal.

**Propostas de ajuste:**

1. **Pipeline — Documentar o gap:** Criar documento normativo formal sobre política de carga danificada (ou confirmar que não existe e registrar como out-of-scope). A ausência de documento aumenta o risco de alucinação para esse tema.

2. **Prompt — Instrução de fallback mais rígida:** Adicionar ao system prompt: *"Se nenhum chunk recuperado contiver a resposta E o campo source_document não puder ser preenchido, você DEVE retornar a mensagem padrão de não encontrado com o próximo passo. Nunca responda com confiança alta sem fonte."*

3. **Código (harness) — Bloquear resposta sem source_document:** O `response-validator.ts` já tem essa regra (G-D01). A resposta #4 deveria ter sido bloqueada por ter `source_document: null` com `confidence: "high"`. Verificar se o validator está sendo aplicado antes da entrega ao bot do Teams — pode haver um bypass no fluxo atual.

---

### Problema #6 — Fonte não confiável sem aviso (carga perigosa + FAQ)

**Tipo de erro:** Fonte não confiável — informação de alto risco citada de FAQ informal sem disclaimer  
**Causa raiz:** O pipeline não está marcando chunks do FAQ com metadado `source_type: "faq"`, ou o `response-builder.ts` não está usando esse metadado para injetar o aviso obrigatório.

**Propostas de ajuste:**

1. **Pipeline — Garantir metadado source_type nos chunks de FAQ:** Verificar com o time de desenvolvimento se o campo `source_type: "faq"` está sendo preservado após a ingestão e indexação no Azure AI Search. Sem esse metadado, o enforcement determinístico do G-N04 não funciona.

2. **Código (harness) — Injetar aviso automático para fonte FAQ:** O `response-builder.ts` deve verificar `source_type` da fonte e, se for `"faq"`, injetar aviso visível ao atendente: *"[Fonte informal — confirme com documento normativo ou supervisor antes de aplicar]"*. Esse aviso não deve depender do modelo incluí-lo.

3. **Prompt — Threshold de confiança para temas de risco:** Adicionar ao system prompt: *"Para perguntas sobre carga perigosa, quando a única fonte disponível for o FAQ-Atendimento, use confidence: 'medium' (nunca 'high') e inclua o aviso de fonte informal."* Isso é uma camada probabilística complementar ao enforcement determinístico.

4. **Interface — Diferenciação visual de fonte informal no Teams:** No Adaptive Card de resposta, exibir fonte FAQ com indicação visual diferente de documento normativo (ex: ícone de aviso, cor diferente). O atendente precisa reconhecer instantaneamente que a fonte tem confiabilidade menor.

---

## Consolidado: Gaps Identificados para Go-Live

| # | Problema | Risco | Ação bloqueante para go-live |
|---|---|---|---|
| 1 | Resposta #4: alucinação sem fonte, confidence alta | Alto — pode gerar compromisso contratual indevido | Verificar se response-validator está bloqueando respostas sem source_document antes do go-live |
| 2 | Resposta #6: FAQ citado sem aviso, confidence alta | Alto — tema regulatório (carga perigosa) | Confirmar metadado source_type nos chunks de FAQ e ativar injeção de aviso no response-builder |
| 3 | Gap documental: política de carga danificada | Médio — aumenta risco de alucinação futura | Documentar como out-of-scope ou criar documento formal |
| 4 | Imprecisão de seção na #1 | Baixo — conteúdo correto | Monitorar; corrigir no próximo ajuste de prompt |

---

*Entregável gerado como parte do Exercício 3.1 — Product Specialist | DB1 IA First*
