# PS — Exercício 1.1: Mapeamento de Intent com Engenharia de Contexto

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Abordagem:** Progressive Disclosure (3 etapas)

---

## Estratégia de Contexto Adotada

A abordagem escolhida é o **progressive disclosure**: em vez de fornecer os 5 documentos completos de uma vez, a informação é fornecida em camadas crescentes de detalhe. O racional é simples — modelos de linguagem têm um orçamento de atenção finito. Quando o contexto é muito grande e misturado desde o início, o modelo tende a:

- Perder detalhes em documentos posicionados no meio (efeito *lost in the middle*)
- Fazer análises superficiais por falta de foco
- Misturar temas sem priorização clara

A abordagem em 3 etapas resolve isso: a Etapa 1 constrói um mapa estruturado, a Etapa 2 mergulha onde mais importa, e a Etapa 3 cruza o resultado com a fonte informal.

---

## Etapa 1 — Visão Geral

### Decisão de Contexto

Fornecido ao modelo apenas **títulos, metadados e resumos de 2-3 linhas** por documento — sem conteúdo completo. Objetivo: obter um mapa de cobertura temática e hipóteses de gaps com o mínimo de ruído. Colocar o conteúdo completo nessa etapa saturaria o contexto antes de ter clareza sobre o que merece atenção.

### Prompt

```
Você está ajudando um Product Specialist a conduzir a fase de Intent de um projeto de IA
para a empresa NovaTech (logística). Abaixo estão os metadados e resumos dos 5 documentos
disponíveis — sem conteúdo completo, apenas visão geral. Com base nisso:

1. Mapeie os temas cobertos pela documentação.
2. Identifique hipóteses de gaps: o que provavelmente falta?
3. Indique quais documentos merecem análise mais profunda e por quê.

Documentos:
- POL-001 (v3.1, jan/2024, Diretoria de Operações) — Política normativa de devolução de
  mercadorias. Cobre regras de prazo, exceções para tipos de carga e procedimento de abertura
  de chamado.
- PROC-042 (v1.0, mar/2023, Diretoria Comercial, ⚠️ sem indicação de vigência) —
  Procedimento de cálculo de frete especial (>500kg). Define fórmula, multiplicadores regionais
  e prazos de entrega.
- PROC-042-v2 (v2.0, nov/2023, Diretoria Comercial, ⚠️ sem indicação de que substitui v1) —
  Revisão do mesmo procedimento de frete especial. Multiplicadores e prazos atualizados, com
  seção de disposições transitórias.
- SLA-2024 (v2024.1, jan/2024, Comercial + Operações) — Tabela contratual de SLA por tier de
  cliente (Gold, Silver, Standard). Define prazos de resposta, resolução e penalidades.
- FAQ-Atendimento (sem versão controlada, mantido informalmente pelo time de atendimento,
  ⚠️ não validado por Compliance) — 47 perguntas e respostas práticas do dia a dia. Pode
  conter informações desatualizadas.
```

### Output Obtido

**Temas cobertos pela documentação:**
- Devolução de mercadorias (regras, prazos, exceções por tipo de carga)
- Cálculo de frete especial (fórmula, multiplicadores regionais, fatores de peso)
- Níveis de serviço por tipo de cliente (SLA Gold/Silver/Standard)
- Práticas informais de atendimento (FAQ)

**Hipóteses de gaps identificados:**
1. Frete padrão (abaixo de 500kg) — nenhum documento cobre
2. Carga danificada em trânsito — sem política formal identificada
3. Seguro de carga — possivelmente mencionado no FAQ, sem documento normativo
4. Processo de escalação para Gestão de Riscos — referenciado na POL-001, mas não detalhado
5. Frete expresso para cargas perigosas — provavelmente apenas prática informal

**Documentos que merecem análise mais profunda:**
- PROC-042 v1 e v2: mesma numeração, datas diferentes, sem hierarquia clara — risco direto de respostas contraditórias no assistente
- FAQ-Atendimento: documento informal usado como referência operacional — pode conter práticas não documentadas ou desatualizadas que contaminam o assistente

### Análise da Qualidade da Etapa 1

A abordagem com apenas metadados funcionou bem para construir o mapa de cobertura. O modelo identificou os 5 gaps corretamente e priorizou os PROC-042 como foco — que é exatamente a escolha certa. A ausência de conteúdo completo forçou o modelo a raciocinar sobre estrutura, não sobre detalhes, o que produziu uma análise mais estratégica.

---

## Etapa 2 — Análise Profunda

### Decisão de Contexto

Com base no mapa da Etapa 1, selecionados os **dois PROC-042** para análise profunda. Justificativa:
- São os únicos documentos com conflito direto e explícito
- O conflito tem impacto financeiro imediato (cálculo errado de frete cobrado ao cliente)
- A v2 tem seção de disposições transitórias que complica ainda mais a interpretação

Os demais documentos (POL-001, SLA-2024, FAQ) não são incluídos nessa etapa para não diluir o foco do modelo. O FAQ é reservado para a Etapa 3.

### Prompt

```
Continuando a análise da NovaTech. O mapa da Etapa 1 identificou que os dois documentos
abaixo coexistem sem hierarquia clara. Analise as inconsistências entre eles:

1. Liste todas as diferenças entre v1 e v2 (valores, prazos, regras).
2. Avalie o risco de cada diferença para o assistente de IA.
3. Proponha como tratar cada inconsistência na especificação do produto.

[PROC-042 v1 — conteúdo completo]
[PROC-042 v2 — conteúdo completo]
```

### Output Obtido

**Inconsistências mapeadas:**

| Dimensão | PROC-042 v1 (mar/2023) | PROC-042 v2 (nov/2023) | Risco |
|---|---|---|---|
| Multiplicador Sul | 1.2 | 1.3 | Alto — erro de 8,3% no valor do frete |
| Multiplicador Sudeste | 1.0 | 1.1 | Alto — erro de 10% |
| Multiplicador Centro-Oeste | 1.3 | 1.4 | Alto — erro de 7,7% |
| Multiplicador Nordeste | 1.4 | 1.5 | Alto — erro de 7,1% |
| Multiplicador Norte | 1.6 | 1.8 | Alto — erro de 12,5% |
| Fator de peso (1.001–3.000kg) | 1.2 | 1.15 | Médio — impacto em cargas intermediárias |
| Fator de peso (>3.000kg) | 1.5 | 1.4 | Médio — erro de 6,7% |
| Prazo adicional de entrega | +2 dias úteis | +3 dias úteis | Alto — promessa ao cliente |
| Desconto por volume | >10 fretes/mês (sem %) | ≥8 fretes: 5%; ≥15 fretes: 10% | Médio — regra mais granular na v2 |

**Tratamento proposto:**
- A v2 deve ser a referência, por ter data mais recente (nov/2023) e seção de disposições transitórias
- Metadado de vigência (data de início, data de fim, status) deve ser explícito no pipeline de RAG
- Chamados abertos antes de 01/12/2023 ainda em processamento: aplicar v1 (conforme seção 5 da v2)
- Ambos os documentos devem permanecer indexados com metadado de status — não excluir a v1

### Análise da Qualidade da Etapa 2

A análise foi substancialmente mais precisa do que seria possível na Etapa 1. Com o conteúdo completo dos dois documentos e o mapa da etapa anterior como âncora, o modelo identificou todas as 9 diferenças e classificou riscos com granularidade suficiente para alimentar uma spec de produto. O fato de fornecer apenas os dois documentos relevantes (sem POL-001 ou SLA) eliminou distrações e permitiu uma comparação linha a linha.

---

## Etapa 3 — Cruzamento com FAQ-Atendimento

### Decisão de Contexto

O FAQ entra agora — e não antes — por duas razões: (1) é um documento informal que, se fornecido primeiro, poderia "contaminar" o raciocínio do modelo com práticas não validadas; (2) o valor do FAQ nessa análise é de contraste, não de fundação. Ao fornecer os outputs das Etapas 1 e 2 como âncora, o modelo tem um referencial sólido para avaliar criticamente o que o FAQ confirma, contradiz ou adiciona.

### Prompt

```
Com base no mapa de temas (Etapa 1) e nas inconsistências do PROC-042 (Etapa 2), analise
o FAQ-Atendimento abaixo e responda:

1. Onde o FAQ confirma as inconsistências já identificadas?
2. Onde o FAQ introduz informações novas que não existem nos documentos formais (gaps)?
3. Quais respostas do FAQ representam risco se indexadas pelo assistente de IA?

[FAQ-Atendimento — conteúdo completo]
```

### Output Obtido

**FAQ confirmando inconsistências já mapeadas:**
- Item 8 ("Como funciona o frete especial?"): o próprio atendente reconhece que existem duas versões da PROC-042 e orienta usar a v2 na dúvida, mas admite que o contrato do cliente pode estar na tabela antiga — valida diretamente o risco identificado na Etapa 2.

**FAQ introduzindo novos gaps (informações sem documento formal):**
- Item 22 (seguro de carga): percentuais mencionados (0,3% padrão / 0,8% perigosa) sem nenhum documento normativo como fonte — risco confirmado de alucinação
- Item 32 (carga perigosa + frete expresso): afirma ser possível "com autorização do Compliance", mas não há PROC ou POL que formalize esse fluxo
- Item 38 (carga danificada): descreve processo de registro em 48h e encaminhamento para sinistros@novatech.com.br, porém sem nenhum documento normativo que sustente esse procedimento

**Respostas do FAQ com risco alto de indexação:**

| Item | Conteúdo do FAQ | Risco |
|---|---|---|
| Item 4 | "Não diga que é impossível — já houve exceções" para carga perigosa | Contradiz POL-001 diretamente; o assistente pode informar que devolução é possível |
| Item 32 | Frete expresso para carga perigosa "com autorização" | Processo não formalizado; assistente pode orientar um fluxo inexistente |
| Item 22 | Percentuais de seguro de carga | Valores sem fonte formal; podem estar desatualizados para contratos mais antigos |

### Análise da Qualidade da Etapa 3

A etapa de cruzamento produziu o resultado mais rico do exercício. A âncora das etapas anteriores permitiu que o modelo avaliasse o FAQ criticamente — não como fonte de verdade, mas como espelho da prática real. A distinção entre "confirma inconsistência já mapeada" e "introduz novo gap" foi precisa e acionável. Sem as etapas 1 e 2 como referencial, o modelo teria provavelmente tratado o FAQ com o mesmo peso dos documentos formais.

---

## Mapa de Riscos para Discovery Humano

### Risco 1 — Documentos contraditórios sem hierarquia definida (PROC-042 v1 e v2)

**Descrição:** Ambas as versões da PROC-042 estão ativas no SharePoint sem indicação de qual é vigente. O assistente pode recuperar qualquer uma delas e gerar cálculos de frete com erros de até 12,5%.

**Impacto:** Alto — erro financeiro direto no atendimento ao cliente.

**Como levar ao discovery humano:** Perguntar à Diretoria Comercial qual versão é formalmente vigente e solicitar arquivamento ou marcação explícita da v1 como obsoleta. Propor processo de versionamento com campo de data de vigência obrigatório para todos os PROCs.

### Risco 2 — FAQ informal indexado junto com documentos normativos

**Descrição:** O FAQ contém pelo menos 3 itens com informações que contradizem ou extrapolam a documentação formal (itens 4, 22 e 32). Se indexado sem distinção de qualidade, o assistente pode misturar orientações informais com regras normativas.

**Impacto:** Alto — pode gerar orientações incorretas que comprometem compliance e confiança no assistente.

**Como levar ao discovery humano:** Apresentar ao time de Compliance os 3 itens de risco e solicitar validação. Propor que o FAQ seja mantido como fonte secundária com peso reduzido no pipeline de RAG, ou que seja revisado antes da indexação.

---

## Reflexão: Progressive Disclosure vs. Tudo de Uma Vez

Se os 5 documentos completos tivessem sido colados no primeiro prompt, o modelo provavelmente teria:

1. **Perdido foco nos PROC-042**: com POL-001, SLA-2024 e FAQ concorrendo por atenção, a comparação linha a linha das duas versões de frete seria menos precisa.
2. **Tratado o FAQ no mesmo nível dos documentos formais**: sem uma âncora de documentos normativos estabelecida antes, o FAQ teria mais peso relativo na análise.
3. **Gerado uma análise horizontal, não aprofundada**: a tendência é produzir um resumo de todos os documentos em vez de uma análise profunda dos que mais importam.
4. **Efeito *lost in the middle***: com ~17.000 palavras de contexto de uma vez, informações nos documentos do meio (PROC-042) seriam processadas com menos atenção do que as do início (POL-001) e do fim (FAQ).

A abordagem em 3 etapas gerou outputs progressivamente mais ricos e focados, com cada etapa alimentando a seguinte. O custo foi 3 prompts em vez de 1 — e o ganho foi uma análise qualitativamente superior e rastreável por etapa.

---

*Entregável gerado como parte do Exercício 1.1 — Product Specialist | DB1 IA First*
