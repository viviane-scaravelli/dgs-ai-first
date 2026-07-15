# PS — Exercício 2.2: Guardrails como Artefato de Produto

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Fase:** 2 — Estruturação

---

## Incidentes de Referência

| # | Incidente | Tipo de falha |
|---|---|---|
| I-01 | Assistente informou prazo de 7 dias para devolução de carga perigosa — cargas perigosas NÃO podem ser devolvidas pelo processo padrão | Inversão de regra + alucinação contextual |
| I-02 | Assistente citou PROC-042 seção 2, mas usou multiplicadores da v1 (desatualizada), não da v2 (vigente) | Chunk errado / versão desatualizada |
| I-03 | Assistente disse "não encontrei informação" para pergunta sobre SLA Gold, com documento SLA-2024 indexado e contendo a resposta | Recusa inadequada / baixa confiança não justificada |

---

## DEVE — Comportamentos Obrigatórios

### G-D01 — Citar fonte em toda resposta
O assistente deve incluir o identificador do documento de origem (nome + seção) em toda resposta, mesmo quando a confiança for baixa.

**Enforcement:** Determinístico (código)  
**Implementação:** `response-validator.ts` verifica presença do campo `source_document` no JSON antes de entregar ao Teams. Se ausente, resposta é bloqueada.  
**Justificativa:** Citação de fonte é critério binário (presente/ausente) — verificável programaticamente. Depender apenas do prompt deixa margem para omissão em respostas longas.  
**Incidente prevenido:** I-02 — com `source_document` contendo versão explícita, a ambiguidade entre v1 e v2 teria sido exposta.

---

### G-D02 — Responder em português formal
O assistente deve usar português formal em toda resposta, independentemente do idioma da pergunta.

**Enforcement:** Prompt (probabilístico)  
**Implementação:** Instrução no system prompt: "Responda sempre em português formal, independentemente do idioma da pergunta."  
**Justificativa:** Formalidade não é verificável deterministicamente. Risco residual aceitável — modelos raramente ignoram essa instrução com system prompt bem configurado.  
**Incidente prevenido:** Falha de guardrail geral (não diretamente nos 3 incidentes).

---

### G-D03 — Incluir campo `confidence` em toda resposta
O assistente deve classificar sua confiança como `high`, `medium` ou `low` com base na qualidade dos chunks recuperados.

**Enforcement:** Determinístico (código)  
**Implementação:** `response-validator.ts` verifica presença e valor válido do campo `confidence`. O código também pode derivar `confidence=low` quando o score de similaridade do Azure AI Search estiver abaixo do threshold definido.  
**Justificativa:** O validador garante que o contrato de API seja sempre respeitado, complementando o julgamento do modelo com dados objetivos do retrieval.  
**Incidente prevenido:** I-03 — com `confidence=low` retornado para a query sobre SLA Gold, o atendente saberia que a resposta era suspeita, mesmo com o documento indexado.

---

## NÃO DEVE — Comportamentos Proibidos

### G-N01 — Nunca gerar valores numéricos não documentados
O assistente não deve informar prazos, multiplicadores, percentuais ou SLAs que não estejam literalmente nos chunks recuperados naquela query.

**Enforcement:** Prompt (probabilístico)  
**Implementação:** "Nunca informe valores numéricos (prazos, multiplicadores, SLAs, percentuais) que não estejam literalmente nos trechos dos documentos fornecidos. Se o valor não estiver nos trechos, diga que não encontrou e oriente o próximo passo."  
**Justificativa:** Verificação determinística de números exigiria parsing complexo com alto índice de falsos positivos. Prompt é a camada mais adequada, complementado pelos critérios VC-04 no plano de testes.  
**Incidente prevenido:** I-01 — o assistente informou "7 dias" para carga perigosa. O valor existe na base, mas não se aplica a esse tipo de carga — o guardrail instrui o modelo a verificar o contexto da regra, não apenas o valor.

---

### G-N02 — Nunca afirmar que carga perigosa pode ser devolvida pelo processo padrão
O assistente não deve, sob nenhuma circunstância, informar que cargas das classes 1 a 6 da ANTT são elegíveis para devolução pelo processo padrão.

**Enforcement:** Determinístico (código) + Prompt — duplo enforcement  
**Implementação:** `response-validator.ts` aplica filtro de padrão de texto: combinação de "carga perigosa" com "pode ser devolvida" / "prazo de X dias" / "7 dias úteis" bloqueia a resposta e substitui por mensagem padrão com ramal 4500. O prompt reforça com instrução explícita.  
**Justificativa:** Este é o guardrail de maior risco — implicações de compliance e segurança. Duplo enforcement é justificado pelo impacto. O filtro de texto é uma aproximação, não perfeita, mas funciona como rede de segurança adicional ao prompt.  
**Incidente prevenido:** I-01 — exatamente o comportamento que esse guardrail bloqueia.

---

### G-N03 — Nunca reconhecer tiers além de Gold, Silver e Standard
O assistente não deve confirmar, usar ou elaborar sobre tiers inexistentes (Platinum, Bronze, Diamond, Premium, Elite).

**Enforcement:** Determinístico (código)  
**Implementação:** `response-validator.ts` verifica presença de termos de tiers não autorizados. Lista: Platinum, Bronze, Diamond, Premium, Elite (e variações). Resposta bloqueada se detectado.  
**Justificativa:** Lista de tiers inválidos é finita e conhecida — verificação determinística é simples e confiável. Apenas prompt cria risco de o modelo "confirmar" um tier se o atendente insistir.  
**Incidente prevenido:** Alucinação documentada na Fase 1 (resposta 3 do Exercício 1.2 — assistente inventou SLAs para tier "Platinum").

---

### G-N04 — Nunca usar FAQ como fonte normativa sem aviso
O assistente não deve apresentar informações exclusivamente do FAQ-Atendimento como regras normativas sem sinalizar que a fonte é informal.

**Enforcement:** Prompt (probabilístico)  
**Implementação:** "Quando a única fonte disponível for o FAQ-Atendimento, sinalize: 'Esta informação vem de referência interna não oficial — confirme com o documento normativo ou com o supervisor antes de usar.'" Depende do pipeline marcar chunks do FAQ com `source_type: faq`.  
**Justificativa:** Distinguir FAQ de documento normativo exige metadado de origem preservado no chunk. Com esse metadado, enforcement via código também é possível futuramente.  
**Incidente prevenido:** Risco mapeado na Fase 1 — FAQ itens 4, 22 e 32 com informações não formalizadas que poderiam contaminar respostas críticas.

---

## QUANDO EM DÚVIDA — Comportamentos de Fallback

### G-Q01 — Prefixar resposta com aviso quando confiança for baixa
Quando `confidence=low`, o assistente deve iniciar a resposta com aviso padronizado antes do conteúdo.

**Enforcement:** Determinístico (código)  
**Implementação:** `response-builder.ts` injeta aviso automaticamente quando `confidence=low`, independentemente do texto gerado pelo modelo.  
**Justificativa:** Aviso de baixa confiança é crítico para o atendente não usar informação duvidosa. Deixar para o modelo decidir cria inconsistência — o código garante que `confidence=low` sempre resulte em aviso visível.  
**Incidente prevenido:** I-03 — o assistente disse "não encontrei" sem aviso de confiança. Com esse guardrail, o atendente receberia a resposta com aviso, em vez de negativa incorreta.

---

### G-Q02 — Quando duas versões de documento existirem, priorizar a mais recente e informar a anterior
O assistente deve usar a versão mais recente como base, mas sempre informar que existe versão anterior com as datas de ambas.

**Enforcement:** Prompt (probabilístico) + Código parcial (híbrido)  
**Implementação:** Prompt instrui sobre prioridade de versão. `response-builder.ts` verifica se `source_documents` contém mais de um documento com mesmo `document_id` e datas diferentes — se sim, injeta `conflict_warning` automaticamente no JSON.  
**Justificativa:** Detecção de "mesmo documento, versões diferentes" é determinística (comparação de metadados). Síntese das diferenças em linguagem natural fica com o modelo. Enforcement híbrido é o mais robusto para esse caso.  
**Incidente prevenido:** I-02 — assistente usou multiplicadores da v1 sem mencionar a v2. Com esse guardrail, a resposta teria apresentado ambas as versões com datas.

---

### G-Q03 — Quando não encontrar resposta, indicar próximo passo específico por tema
O assistente não deve apenas dizer "não encontrei" — deve orientar o próximo passo com base no tema da pergunta.

**Enforcement:** Prompt (probabilístico)  
**Implementação:** Instrução com mapeamento de próximos passos: cargas especiais → Gestão de Riscos ramal 4500; frete e descontos → Comercial; SLAs e contratos → gerente de conta; outros → supervisor imediato.  
**Justificativa:** O mapeamento tema → próximo passo é conhecimento de domínio. O código não consegue inferir qual área é responsável a partir do texto da pergunta com confiabilidade suficiente.  
**Incidente prevenido:** I-03 — assistente disse apenas "não encontrei" sem orientar o atendente. Com esse guardrail, a resposta incluiria o próximo passo específico.

---

## Consolidado — Rastreabilidade por Incidente

| Incidente | Guardrails que previnem |
|---|---|
| I-01: Prazo de devolução para carga perigosa | G-N01, G-N02 |
| I-02: Multiplicadores desatualizados (v1 vs v2) | G-D01, G-Q02 |
| I-03: "Não encontrei" para SLA Gold indexado | G-D03, G-Q01, G-Q03 |

## Consolidado — Enforcement por tipo

| Tipo | Guardrails |
|---|---|
| Determinístico (código) | G-D01, G-D03, G-N02 (parcial), G-N03, G-Q01, G-Q02 (parcial) |
| Probabilístico (prompt) | G-D02, G-N01, G-N04, G-Q03 |
| Híbrido (código + prompt) | G-N02, G-Q02 |

---

*Entregável gerado como parte do Exercício 2.2 — Product Specialist | DB1 IA First*
