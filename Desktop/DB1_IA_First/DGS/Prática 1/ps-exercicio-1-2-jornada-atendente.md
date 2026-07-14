# PS — Exercício 1.2: Design de Jornada com Componente de IA

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech

---

## Contexto

Com base no discovery simulado: atendentes abrem em média 4 fontes por chamado. As dúvidas mais comuns são prazos de entrega (35%), regras de frete (25%), política de devolução (20%) e outros (20%). Em 15% dos casos o atendente não encontra resposta e escala para o supervisor.

---

## Fluxo Principal — Caminho Feliz

1. **Recebimento da dúvida** — Atendente recebe pergunta do cliente via chamado (Teams/portal). Identifica o tema: prazo, frete, devolução ou SLA.
2. **Consulta ao assistente** — Digita a pergunta em linguagem natural no assistente integrado ao Teams.
3. **Resposta com fonte** — O assistente retorna: conteúdo da resposta, documento de origem (ex: "POL-001, seção 3.2"), trecho relevante destacado e nível de confiança (alto/médio).
4. **Validação rápida** — Atendente lê a resposta e a fonte citada. Se coerente com o contexto do cliente, usa no atendimento.
5. **Uso no atendimento** — Atendente responde ao cliente com a informação fundamentada. Fecha o chamado com tag "assistido por IA".

---

## Fluxo de Fallback — Quando o Assistente Não Tem Resposta Confiante

**Gatilhos:**
- O assistente exibe aviso: "Não encontrei informação suficiente na base para responder com segurança."
- O assistente retorna resposta, mas o atendente discorda com base em experiência ou contexto do cliente.
- A pergunta envolve situação de exceção (carga perigosa, contrato antigo, frete expresso especial).

**Passos:**
1. Atendente não complementa a resposta com suposições próprias (guardrail de comportamento humano).
2. Escalona para o supervisor com contexto completo: pergunta original, resposta do assistente, motivo da dúvida.
3. Registra ocorrência de "resposta não encontrada" no sistema de chamados para rastreamento.
4. Responde ao cliente com prazo de retorno conforme SLA do tier: "Estou verificando com a equipe especializada e retorno em até X horas."

---

## Fluxo de Feedback — Quando a Resposta Está Errada, Desatualizada ou Incompleta

**Gatilhos:**
- Atendente percebe contradição com o que o supervisor confirmou.
- Cliente informa que a informação está desatualizada.
- Atendente identifica que o documento citado tem versão mais recente.

**Passos:**
1. **Sinaliza no assistente** com botão "Resposta incorreta", classificando o problema: incorreta / desatualizada / incompleta / fonte errada.
2. **Descreve brevemente** o problema (campo obrigatório): ex. "O multiplicador para o Norte informado é 1.6, mas o contrato usa a tabela v2 (1.8)."
3. **Ticket automático gerado** para o time de curadoria responsável pelo tema (Compliance / Operações / Comercial).
4. **Atendente é notificado** quando o documento for atualizado ou quando o feedback for descartado com justificativa.

---

## Guardrails de Comportamento do Assistente

### Guardrail 1 — Nunca inventar prazos, valores ou multiplicadores
O assistente só informa prazos (entrega, devolução, SLA) e valores numéricos (multiplicadores de frete, percentuais de seguro) se o dado estiver explicitamente no documento recuperado. Se o trecho não contiver o número exato, responde: "Não encontrei o valor específico na documentação. Recomendo consultar [documento] ou acionar o Comercial." — jamais interpola ou estima.

### Guardrail 2 — Sinalizar conflito entre versões de documento
Quando dois chunks recuperados contiverem informações contraditórias sobre o mesmo tema (ex: PROC-042 v1 e v2 com multiplicadores diferentes), o assistente não escolhe silenciosamente. Exibe obrigatoriamente: "Encontrei versões diferentes desta informação. [Versão A — fonte, data]. [Versão B — fonte, data]. Confirme com o Comercial qual se aplica ao contrato deste cliente."

### Guardrail 3 — Nunca orientar processo não documentado
Se a pergunta envolver situação coberta apenas pelo FAQ informal (ex: frete expresso para carga perigosa, exceções de devolução), o assistente não reproduz a orientação do FAQ como regra. Responde: "Este caso envolve exceção que requer tratamento especializado. Acione Gestão de Riscos (ramal 4500)."

---

## Diagrama Visual

O diagrama de fluxo foi gerado com os três caminhos:
- **Azul** — Fluxo principal (caminho feliz)
- **Laranja** — Fluxo de fallback (sem resposta confiante)
- **Verde** — Fluxo de feedback (resposta incorreta/desatualizada)
- **Roxo** — Guardrails de comportamento

---

*Entregável gerado como parte do Exercício 1.2 — Product Specialist | DB1 IA First*
