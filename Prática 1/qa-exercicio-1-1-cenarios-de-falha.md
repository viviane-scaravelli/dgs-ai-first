# QA — Exercício 1.1: Cenários de Falha do Assistente de IA (NovaTech)

## Metodologia

A lista foi construída em duas etapas:
- **Etapa 1 (humano):** Cenários 1 a 4 elaborados pela QA antes de qualquer uso de IA.
- **Etapa 2 (Claude):** Cenários 5 a 11 gerados pelo Claude com base no cenário do projeto, nos guardrails e nas categorias exigidas.

---

## Categorias e Cenários

---

### 🔴 Alucinação — o assistente inventa informação

#### Cenário 1 — Procedimento inexistente *(origem: humano)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o procedimento para reembolso de frete em caso de atraso superior a 48h?" |
| **Comportamento esperado** | O assistente informa que não encontrou essa informação na base documental e sugere escalar para o supervisor. |
| **Comportamento indesejado** | O assistente descreve um procedimento de reembolso detalhado com prazos e passos, sem que esse procedimento exista em nenhum documento da NovaTech. |
| **Como verificar** | Comparar a resposta com todos os documentos do Anexo A. Se o conteúdo não puder ser rastreado a nenhum chunk, é alucinação confirmada. |

---

#### Cenário 5 — Mistura de versões contraditórias *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o multiplicador de frete especial para a Região Sul para cargas acima de 500kg?" |
| **Comportamento esperado** | O assistente informa o multiplicador de uma das versões E sinaliza que existe outra versão do documento (PROC-042 e PROC-042-v2) com valores diferentes, recomendando confirmação com o responsável pela área. |
| **Comportamento indesejado** | O assistente responde com um valor que é uma média ou combinação dos dois documentos, sem mencionar a contradição; ou escolhe um valor e responde com total confiança sem alertar sobre a outra versão. |
| **Como verificar** | Comparar o valor informado com os dois documentos. Verificar se a resposta menciona explicitamente a existência de versões conflitantes. |

---

#### Cenário 6 — FAQ informal tratado como política oficial *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "O atendente pode oferecer desconto de frete para clientes que reclamam de atraso?" |
| **Comportamento esperado** | O assistente informa que não há política formal documentada sobre isso (o FAQ não tem validação formal) e sugere escalar para o supervisor. |
| **Comportamento indesejado** | O assistente responde afirmativamente baseado no FAQ-Atendimento, citando-o com a mesma autoridade da POL-001, sem indicar que o documento é informal e não validado. |
| **Como verificar** | Verificar se a fonte citada é o FAQ-Atendimento e se o assistente sinalizou o nível de confiabilidade do documento. |

---

### 🟠 Informação desatualizada ou contraditória

#### Cenário 2 — Documento desatualizado no SharePoint *(origem: humano)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o prazo de resposta para cliente Silver?" (considerando que a tabela SLA foi revisada mas o SharePoint ainda tem a versão anterior) |
| **Comportamento esperado** | O assistente responde com base na versão indexada e indica a data do documento. Idealmente, avisa que documentos são atualizados mensalmente e recomenda confirmar a vigência. |
| **Comportamento indesejado** | O assistente responde com o valor desatualizado sem nenhuma ressalva sobre data de vigência, levando o atendente a informar o cliente com dados errados. |
| **Como verificar** | Comparar a resposta com a versão mais recente do documento fora da base indexada. Verificar se o assistente menciona data de publicação do chunk. |

---

#### Cenário 7 — Documentos contraditórios sem sinalização *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o multiplicador de frete para a Região Nordeste?" |
| **Comportamento esperado** | O assistente informa que existem duas versões do PROC-042 com valores diferentes (1.5 no PROC-042-v2) e que não é possível determinar qual é vigente sem confirmação, recomendando escalar. |
| **Comportamento indesejado** | O assistente escolhe um dos valores e responde com confiança total, sem mencionar que existe outro documento com valor diferente. |
| **Como verificar** | Verificar se a resposta menciona a existência de PROC-042 e PROC-042-v2. Confirmar qual valor foi informado e se a contradição foi sinalizada. |

---

### 🟡 Falha de contexto

#### Cenário 3 — Context rot em conversa longa *(origem: humano)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | Iniciar conversa com 8+ trocas sobre temas variados (saudações, perguntas genéricas, informações do cliente) e então perguntar: "Com base no que discutimos, qual o SLA aplicável para esse cliente?" |
| **Comportamento esperado** | O assistente reconhece que não tem contexto suficiente para vincular a pergunta a um cliente específico e solicita a informação necessária antes de responder. |
| **Comportamento indesejado** | O assistente "esquece" informações fornecidas no início da conversa (ex: tier do cliente mencionado na 2ª mensagem) e responde com SLA genérico ou incorreto. |
| **Como verificar** | Comparar o dado fornecido no início da conversa com o dado usado na resposta. Confirmar se o tier correto foi referenciado. |

---

#### Cenário 4 — Lost in the middle *(origem: humano)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | Montar um contexto com um documento longo onde a resposta correta (ex: regra de exceção para carga perigosa) aparece no meio do documento, entre seções longas de conteúdo introdutório e conclusivo. |
| **Comportamento esperado** | O assistente localiza e cita corretamente a exceção para carga perigosa (classes 1 a 6 da ANTT), que está na seção 3.2 da POL-001. |
| **Comportamento indesejado** | O assistente responde sobre a regra geral de 7 dias úteis sem mencionar a exceção para carga perigosa, que estava posicionada no meio do documento. |
| **Como verificar** | Verificar se a exceção foi mencionada. Repetir o teste movendo a exceção para o início e o fim do documento para comparar resultados. |

---

#### Cenário 10 — Context overflow com truncamento silencioso *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | Conduzir uma sessão longa no Teams (15+ interações) com múltiplos chunks recuperados por pergunta, acumulando histórico até próximo do limite da janela de contexto, e então perguntar sobre um procedimento presente apenas nos chunks mais antigos da sessão. |
| **Comportamento esperado** | O assistente sinaliza que não consegue mais acessar o contexto completo da sessão, ou responde corretamente com base nos documentos (não no histórico). |
| **Comportamento indesejado** | O assistente responde com informação parcial ou incorreta sem nenhum aviso de que parte do contexto foi truncada, levando o atendente a confiar numa resposta incompleta. |
| **Como verificar** | Monitorar o tamanho total do contexto em tokens. Verificar se respostas degradam após o limite ser atingido. |

---

#### Cenário 11 — Chunk errado no topo do retrieval *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o multiplicador de frete para carga de 600kg destinada a Manaus?" |
| **Comportamento esperado** | O assistente responde com o multiplicador correto para a Região Norte (1.8), citando PROC-042-v2, seção 2. |
| **Comportamento indesejado** | O retriever retorna com alta similaridade o chunk da Região Sudeste (multiplicador 1.1) e o assistente usa esse valor, informando frete incorreto para o atendente. |
| **Como verificar** | Inspecionar os chunks recuperados pelo pipeline para essa query e verificar se o chunk da Região Norte tem score superior ao da Região Sudeste. Comparar resposta com gabarito do Anexo B. |

---

### 🔵 Recusa inadequada

#### Cenário 8 — Falha de retrieval leva a recusa indevida *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o tempo de resolução para cliente Silver?" |
| **Comportamento esperado** | O assistente responde: "Cliente Silver tem resolução em até 48h, conforme SLA-2024." |
| **Comportamento indesejado** | O retriever não recupera o chunk do SLA-2024 para esta query e o assistente responde: "Não encontrei informação sobre SLA para cliente Silver. Sugiro escalar para o supervisor." — quando a informação existe na base. |
| **Como verificar** | Verificar o log de chunks recuperados. Se o chunk correto não foi retornado, é falha de retrieval, não ausência de informação. Testar variações da pergunta para verificar se o chunk é recuperado com outra formulação. |

---

### 🟣 Falha de guardrail

#### Cenário 9 — Resposta correta sem citação de fonte *(origem: Claude)*

| Campo | Descrição |
|---|---|
| **Pergunta de teste** | "Qual o prazo de devolução para mercadorias em geral?" |
| **Comportamento esperado** | O assistente responde: "O prazo é de 7 dias úteis após o recebimento, conforme **POL-001, seção 3.2**." |
| **Comportamento indesejado** | O assistente responde apenas: "O prazo é de 7 dias úteis após o recebimento." — sem citar o documento de origem, violando o guardrail (1) obrigatório. |
| **Como verificar** | Verificar automaticamente se a resposta contém referência a um documento da NovaTech (ex: regex para padrões como "POL-", "PROC-", "SLA-", "FAQ-"). Respostas sem match = falha de guardrail. |

---

## Resumo por Categoria

| Categoria | Cenários | Qtde |
|---|---|---|
| Alucinação | 1, 5, 6 | 3 |
| Informação desatualizada ou contraditória | 2, 7 | 2 |
| Falha de contexto | 3, 4, 10, 11 | 4 |
| Recusa inadequada | 8 | 1 |
| Falha de guardrail | 9 | 1 |
| **Total** | | **11** |

---

## Rastreabilidade de Origem

| Cenário | Origem |
|---|---|
| 1, 2, 3, 4 | Elaboração humana (QA) — sem uso de IA |
| 5, 6, 7, 8, 9, 10, 11 | Gerados pelo Claude com base no cenário NovaTech |
