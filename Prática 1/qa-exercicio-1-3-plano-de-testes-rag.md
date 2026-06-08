# QA — Exercício 1.3: Plano de Testes — Pipeline de RAG (NovaTech)

## Contexto

O pipeline de RAG da NovaTech indexa ~1.250 fontes (800 PDFs no SharePoint, 400 páginas no Confluence, 50 planilhas) e responde perguntas de 45 atendentes via Microsoft Teams. O plano cobre cada etapa do pipeline separadamente e o fluxo integrado.

> **Nota sobre testes de IA:** Diferente de testes tradicionais, testes de sistemas de IA são não-determinísticos — a mesma pergunta pode gerar respostas diferentes entre execuções. Por isso, os critérios de aprovação aqui usam graus de qualidade (escala) e verificações estruturais (presença de fonte, ausência de termos proibidos), não apenas pass/fail binário. Testes de regressão devem usar conjuntos fixos de perguntas com respostas esperadas e tolerâncias definidas.

---

## 1. Testes de Ingestão

**Objetivo:** Verificar que os documentos foram corretamente extraídos, convertidos em texto e indexados no Azure AI Search.

### TC-ING-01 — Contagem de documentos indexados
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Procedimento** | Consultar o índice após pipeline de ingestão e contar o número de documentos indexados. Comparar com o inventário conhecido (ex: 800 PDFs do SharePoint). |
| **Critério de aprovação** | Número de documentos indexados ≥ 95% do total esperado. Documentos faltantes devem ser listados em log. |
| **O que pode dar errado** | PDFs escaneados sem OCR geram chunks vazios; arquivos corrompidos são silenciosamente ignorados. |

### TC-ING-02 — Integridade de extração de texto
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Procedimento** | Para uma amostra de 10 documentos, comparar o texto extraído com o conteúdo visual do documento original. Verificar se tabelas, listas e seções numeradas foram preservadas. |
| **Critério de aprovação** | Sem perda de dados estruturais críticos (ex: valores de tabelas de frete, prazos de SLA). Tolerância de 0% para campos numéricos. |
| **O que pode dar errado** | Tabelas complexas em PDF são convertidas em texto corrido e perdem estrutura; fluxogramas como imagem geram chunks vazios. |

### TC-ING-03 — Metadados de versionamento
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Procedimento** | Verificar se todos os chunks indexados possuem os metadados obrigatórios: nome do documento, data de publicação, versão, seção de origem. |
| **Critério de aprovação** | 100% dos chunks com metadados completos. Chunks sem data de publicação devem falhar a ingestão com erro explícito. |
| **O que pode dar errado** | Documentos sem metadados são indexados sem data; versões conflitantes (PROC-042 e PROC-042-v2) ficam indistinguíveis sem versionamento. |

### TC-ING-04 — Documentos contraditórios marcados
| Campo | Descrição |
|---|---|
| **Tipo** | Manual + Automático |
| **Procedimento** | Após ingestão, consultar chunks com mesmo identificador de procedimento e versões diferentes (ex: PROC-042). Verificar se ambos estão presentes e se possuem flag de "versão conflitante". |
| **Critério de aprovação** | Chunks de PROC-042 e PROC-042-v2 existem no índice. Ambos têm metadado `versao_conflitante: true`. |
| **O que pode dar errado** | Pipeline sobrescreve versão anterior silenciosamente; ou indexa ambas sem nenhum sinalizador, deixando o LLM decidir sem contexto. |

### TC-ING-05 — Atualização dentro do SLA de 24h
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Procedimento** | Publicar um documento de teste no SharePoint e medir o tempo até ele aparecer como recuperável nas buscas do pipeline. |
| **Critério de aprovação** | Documento disponível no índice em até 24h após publicação, conforme requisito de produto. |
| **O que pode dar errado** | Pipeline de ingestão roda manualmente ou em janelas infrequentes; notificação de novo documento não aciona re-ingestão automática. |

---

## 2. Testes de Retrieval

**Objetivo:** Dada uma pergunta conhecida, verificar se os chunks corretos são recuperados com score de similaridade suficiente.

> Os pares abaixo usam o mapa de cobertura do Anexo B como gabarito.

### TC-RET-01 — Regra de devolução geral
| Campo | Descrição |
|---|---|
| **Pergunta** | "Qual o prazo para devolução de mercadoria?" |
| **Chunk esperado** | POL-001, seção 3.2 — prazo de 7 dias úteis |
| **Tipo** | Automático |
| **Critério de aprovação** | Chunk da POL-001 §3.2 está entre os 3 primeiros resultados com score > 0.75. |
| **Falha esperada** | Retriever retorna FAQ-Atendimento em vez da POL-001 (fonte informal no lugar da oficial). |

### TC-RET-02 — Exceção de carga perigosa
| Campo | Descrição |
|---|---|
| **Pergunta** | "Posso devolver carga perigosa classe 3?" |
| **Chunk esperado** | POL-001, seção 3.2 — exceção para classes 1–6 ANTT |
| **Tipo** | Automático |
| **Critério de aprovação** | Chunk da exceção aparece no topo. Score > 0.75. |
| **Falha esperada** | Retriever retorna chunk da regra geral de 7 dias (sem a exceção), levando o LLM a responder incorretamente. |

### TC-RET-03 — SLA por tier de cliente
| Campo | Descrição |
|---|---|
| **Pergunta** | "Qual o tempo de resolução para cliente Gold?" |
| **Chunk esperado** | SLA-2024 — linha do cliente Gold: resolução em até 24h |
| **Tipo** | Automático |
| **Critério de aprovação** | Chunk do SLA-2024 referente ao Gold é o primeiro resultado. Chunks de outros tiers (Silver, Standard) não contaminam o top-1. |
| **Falha esperada** | Chunk do Silver é retornado no lugar do Gold por similaridade semântica entre os tiers. |

### TC-RET-04 — Frete especial por região
| Campo | Descrição |
|---|---|
| **Pergunta** | "Qual o multiplicador de frete para carga acima de 500kg destinada a Manaus?" |
| **Chunk esperado** | PROC-042-v2, seção 2 — Região Norte, multiplicador 1.8 |
| **Tipo** | Automático |
| **Critério de aprovação** | Chunk da Região Norte (1.8) tem score superior ao da Região Sudeste (1.1). |
| **Falha esperada** | Chunk da Região Sudeste retornado no topo por similaridade léxica com "multiplicador de frete". |

### TC-RET-05 — Pergunta sem resposta na base
| Campo | Descrição |
|---|---|
| **Pergunta** | "Qual o procedimento para reembolso de frete por atraso?" |
| **Chunk esperado** | Nenhum chunk relevante (procedimento não existe na base) |
| **Tipo** | Automático |
| **Critério de aprovação** | Os chunks recuperados têm score < 0.60 (baixa similaridade). O pipeline sinaliza baixa confiança ao LLM. |
| **Falha esperada** | Retriever retorna chunks vagamente relacionados a "frete" com score alto, induzindo o LLM a inventar um procedimento. |

### TC-RET-06 — Pergunta multi-domínio
| Campo | Descrição |
|---|---|
| **Pergunta** | "Cliente Gold quer devolver carga e saber o prazo de resposta para a reclamação." |
| **Chunks esperados** | POL-001 §3.2 (devolução) + SLA-2024 linha Gold (prazo de resposta) |
| **Tipo** | Automático |
| **Critério de aprovação** | Ambos os chunks aparecem nos top-5 resultados. Nenhum chunk irrelevante aparece no top-3. |
| **Falha esperada** | Retriever retorna apenas um dos dois temas (retrieval single-domain) forçando resposta incompleta. |

---

## 3. Testes de Geração

**Objetivo:** Dados os chunks corretos, verificar se o LLM gera resposta adequada.

### TC-GER-01 — Resposta correta com fonte
| Campo | Descrição |
|---|---|
| **Tipo** | Automático (verificação estrutural) + Manual (verificação factual) |
| **Procedimento** | Para cada pergunta do mapa de cobertura, fornecer os chunks corretos ao LLM e verificar a resposta. |
| **Critério de aprovação** | Resposta contém citação de fonte (regex: `POL-\d+`, `PROC-\d+`, `SLA-\d+`). Valor informado corresponde ao chunk fornecido. |

### TC-GER-02 — Respeito à exceção crítica
| Campo | Descrição |
|---|---|
| **Tipo** | Manual |
| **Procedimento** | Fornecer chunk da POL-001 §3.2 completo (regra + exceção) e perguntar "Posso devolver carga perigosa?". |
| **Critério de aprovação** | Resposta explicita que carga perigosa **não** pode ser devolvida, referenciando a exceção do §3.2. |
| **Falha esperada** | LLM responde com a regra geral de 7 dias ignorando a exceção contida no mesmo chunk (lost in the middle dentro do chunk). |

### TC-GER-03 — Comportamento com chunks contraditórios
| Campo | Descrição |
|---|---|
| **Tipo** | Manual |
| **Procedimento** | Fornecer chunks do PROC-042 (multiplicadores antigos) e PROC-042-v2 (multiplicadores novos) na mesma query. |
| **Critério de aprovação** | LLM sinaliza que existem duas versões com valores diferentes e recomenda confirmar a vigente, sem inventar um valor médio. |
| **Falha esperada** | LLM mistura multiplicadores das duas versões ou escolhe um sem alertar sobre a contradição. |

### TC-GER-04 — Recusa quando não há resposta
| Campo | Descrição |
|---|---|
| **Tipo** | Automático (detecção de padrão) |
| **Procedimento** | Fornecer chunks de baixa relevância e perguntar sobre procedimento inexistente. |
| **Critério de aprovação** | Resposta contém padrão de recusa explícita (ex: "não encontrei", "não há informação", "sugiro escalar"). Não contém valores ou prazos inventados. |

---

## 4. Testes de Contexto

**Objetivo:** Verificar que o contexto montado (prompt + chunks + histórico) está dentro do orçamento e que falhas de gerenciamento de contexto não degradam respostas.

### TC-CTX-01 — Orçamento de contexto por query
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Procedimento** | Medir o total de tokens do contexto montado por query: system prompt (~2K) + chunks recuperados + pergunta + histórico. |
| **Critério de aprovação** | Total ≤ 90% da janela do modelo (ex: ≤ 115K tokens para GPT-4o com 128K). Queries acima do limite devem acionar truncamento controlado com log. |

### TC-CTX-02 — Context rot em sessão longa
| Campo | Descrição |
|---|---|
| **Tipo** | Manual |
| **Procedimento** | Conduzir sessão com 10+ turnos. No turno 1, informar tier do cliente (Gold). No turno 10, perguntar: "Qual o SLA para esse cliente?". |
| **Critério de aprovação** | Resposta usa o tier Gold informado no turno 1. |
| **Falha esperada** | Resposta usa SLA genérico ou pede que o tier seja informado novamente — indica que o histórico inicial foi descartado do contexto. |

### TC-CTX-03 — Lost in the middle em documento longo
| Campo | Descrição |
|---|---|
| **Tipo** | Manual |
| **Procedimento** | Construir chunk artificial onde a informação crítica (ex: exceção de carga perigosa) está posicionada no centro, entre blocos longos de texto introdutório e conclusivo. Comparar resposta com versão onde a exceção está no início do chunk. |
| **Critério de aprovação** | Resposta identifica a exceção em ambas as posições. |
| **Falha esperada** | Exceção é mencionada quando está no início, mas ignorada quando está no meio — evidência de lost in the middle. |

### TC-CTX-04 — Context overflow com truncamento
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Procedimento** | Forçar query onde histórico + chunks + sistema ultrapassa o limite da janela. Verificar comportamento do pipeline. |
| **Critério de aprovação** | Pipeline trunca o histórico (não os chunks nem o system prompt), registra aviso em log, e a resposta ainda é baseada nos documentos. |
| **Falha esperada** | Pipeline trunca chunks ou system prompt para caber o histórico, degradando a qualidade da resposta sem aviso ao atendente. |

---

## 5. Testes de Ponta a Ponta

**Objetivo:** Simular o fluxo completo — pergunta do atendente → resposta com fonte — usando perguntas realistas do contexto de logística.

### TC-E2E-01 a TC-E2E-05 — Conjunto de perguntas e respostas esperadas

| ID | Pergunta | Resposta esperada (resumo) | Critério |
|---|---|---|---|
| TC-E2E-01 | "Qual o prazo de devolução para cliente Standard?" | 7 dias úteis (POL-001 §3.2), exceto carga perigosa | Contém prazo + exceção + fonte |
| TC-E2E-02 | "Cliente Silver, quanto tempo tem para resolução?" | Até 48h (SLA-2024) | Contém 48h + Gold não mencionado + fonte |
| TC-E2E-03 | "Frete para 800kg para Porto Alegre, qual o multiplicador?" | Região Sul, multiplicador 1.3 (PROC-042-v2 §2) | Contém 1.3 + fonte + ressalva sobre versão |
| TC-E2E-04 | "O cliente pode abrir chamado por qual canal para devolução?" | Portal, com fotos da mercadoria (POL-001 §3.2) | Contém "portal" + "fotos" + fonte |
| TC-E2E-05 | "Qual o SLA de resposta para cliente Platinum?" | Não existe tier Platinum — recusa explícita com sugestão de escalar | Contém recusa + não contém prazo inventado |

---

## 6. Testes de Regressão

**Objetivo:** Quando o prompt ou um documento é atualizado, garantir que os testes anteriores continuam passando.

### TC-REG-01 — Regressão após atualização de documento
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Gatilho** | Novo documento indexado (ex: nova versão do SLA ou do PROC-042) |
| **Procedimento** | Rodar automaticamente o conjunto TC-RET-01 a TC-RET-06 e TC-E2E-01 a TC-E2E-05 após qualquer re-ingestão. |
| **Critério de aprovação** | 100% dos testes de retrieval passam. Testes de geração com tolerância de ±10% no score de qualidade. |

### TC-REG-02 — Regressão após alteração de prompt
| Campo | Descrição |
|---|---|
| **Tipo** | Automático |
| **Gatilho** | Commit em arquivo de system prompt no repositório |
| **Procedimento** | Pipeline de CI roda TC-GER-01 a TC-GER-04 e os 5 testes E2E com a nova versão do prompt. Compara resultados com baseline da versão anterior. |
| **Critério de aprovação** | Nenhum teste que passava na versão anterior passa a falhar. Melhorias são documentadas no changelog do prompt. |

### TC-REG-03 — Monitoramento contínuo de alucinação
| Campo | Descrição |
|---|---|
| **Tipo** | Automático (amostragem) |
| **Procedimento** | Amostrar 5% das queries reais em produção diariamente. Verificar automaticamente: presença de citação de fonte, ausência de valores não documentados (regex para tiers/prazos fora do gabarito), e taxa de recusa em perguntas sem resposta. |
| **Critério de aprovação** | Taxa de citação de fonte ≥ 95%. Taxa de alucinação detectável ≤ 2%. Qualquer degradação acima de 5 pontos percentuais aciona alerta. |

---

## Resumo do Plano

| Categoria | Qtde de Testes | Tipo Predominante |
|---|---|---|
| Ingestão | 5 | Automático |
| Retrieval | 6 | Automático |
| Geração | 4 | Automático + Manual |
| Contexto | 4 | Manual + Automático |
| Ponta a Ponta | 5 | Automático + Manual |
| Regressão | 3 | Automático |
| **Total** | **27** | |

### Por que testes de IA são diferentes
- **Não-determinísticos:** a mesma pergunta pode gerar respostas diferentes. Use conjuntos fixos com múltiplas execuções e médias.
- **Graus de qualidade:** não é pass/fail — é "a resposta contém a exceção crítica?", "o score de retrieval está acima do limiar?".
- **Regressão por amostragem:** impossível testar 100% das queries em produção; monitoramento contínuo por amostra é obrigatório.
- **Feedback loop:** falhas reportadas por atendentes em produção devem retroalimentar novos casos de teste.
