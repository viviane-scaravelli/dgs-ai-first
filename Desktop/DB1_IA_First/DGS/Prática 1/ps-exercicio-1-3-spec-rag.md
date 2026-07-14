# PS — Exercício 1.3: Especificação de Requisitos de RAG

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Formato:** Spec de produto — não técnica, mas testável pelo QA

---

## Spec v1 — Versão Inicial

### 1. Fontes de dados a indexar

- **Indexar:** documentos do SharePoint (PDFs e DOCXs das áreas Operações, Compliance e Comercial), páginas da wiki Confluence (TI e Comercial) e planilhas de referência da pasta de rede.
- **Não indexar:** documentos marcados como obsoletos, rascunhos sem aprovação formal e o FAQ-Atendimento em sua forma atual.
- O FAQ-Atendimento poderá ser indexado após revisão e validação pelo Compliance, com tag de "fonte secundária".

### 2. Tratamento de documentos contraditórios

- Quando dois documentos cobrirem o mesmo tema com informações diferentes, o assistente deve exibir ambas as versões com data de emissão e não escolher automaticamente.
- A versão mais recente deve ser apresentada primeiro.
- O assistente deve incluir aviso explícito: "Existem versões diferentes deste procedimento. Confirme com o responsável da área qual se aplica ao caso."

### 3. Comportamento quando não há resposta na base

- O assistente deve responder explicitamente que não encontrou a informação, sem tentar responder com conhecimento geral.
- Deve sugerir a próxima ação: escalar para supervisor ou consultar área responsável.

### 4. Requisitos de atualização

- Novos documentos publicados devem estar disponíveis no assistente em até 24 horas.

### 5. Requisitos de rastreabilidade

- Toda resposta deve citar o documento de origem com nome e seção.
- O trecho relevante deve ser exibido junto com a resposta.

---

## Feedback do Claude (Revisor)

**Prompt de iteração:**
> Você é um revisor de produto sênior. Leia a spec abaixo e identifique gaps, ambiguidades ou requisitos que um QA não conseguiria verificar objetivamente.

**Gaps e ambiguidades identificados:**

**Gap 1 — Critério de "obsoleto" não definido**
A spec diz "não indexar documentos marcados como obsoletos", mas não define quem marca, onde e com qual mecanismo. Se não há processo formal de arquivamento no SharePoint da NovaTech (e pelo cenário, não há), esse requisito é inoperável.

**Gap 2 — "Fonte secundária" sem comportamento definido**
A spec menciona que o FAQ pode ser indexado como "fonte secundária", mas não especifica o que isso significa na prática: o assistente usa ou não usa? Com qual peso? Em quais condições?

**Gap 3 — Prazo de 24h sem critério de início**
"Disponível em até 24h" a partir de quando? Da publicação no SharePoint? Da aprovação formal? Da detecção pelo pipeline? São momentos diferentes.

**Gap 4 — Rastreabilidade sem critério de qualidade do trecho**
"Exibir o trecho relevante" não define o que é relevante: tamanho mínimo, máximo, se é o chunk exato recuperado ou um recorte editado. Um QA não sabe o que verificar.

**Gap 5 — Sem requisito para documentos já indexados e posteriormente invalidados**
A spec define o que não indexar, mas não define o que acontece com documentos que já estão indexados e foram invalidados (ex: v1 do PROC-042 após adoção da v2).

**Gap 6 — Ausência de critério para perguntas multi-documento**
Quando a resposta correta exige cruzar dois documentos (ex: SLA Gold + prazo de devolução da POL-001), a spec não define se o assistente deve ou pode fazer esse cruzamento.

---

## Spec v2 — Versão Refinada

### 1. Fontes de dados a indexar

**Fontes elegíveis para indexação:**
- SharePoint corporativo: documentos com status "Publicado" e aprovação formal registrada (campo de metadado "Status = Aprovado" ou equivalente definido com a NovaTech na fase de discovery).
- Wiki Confluence: páginas com data de última edição nos últimos 12 meses e sem tag "Arquivado" ou "Em revisão".
- Pasta de rede: planilhas de referência com nome seguindo o padrão de versionamento mensal definido pelo Comercial (ex: `frete-base-AAAAMM.xlsx`).

**Fontes excluídas da indexação:**
- Documentos sem data de aprovação registrada.
- Documentos com duas ou mais versões cujas datas de vigência se sobrepõem sem resolução formal — esses casos devem ser sinalizados para curadoria humana antes da indexação.
- O FAQ-Atendimento, enquanto não validado pelo Compliance.

**Condição para indexação do FAQ:**
O FAQ-Atendimento pode ser indexado como fonte de segunda ordem após: (1) revisão item a item pelo Compliance, (2) remoção ou correção dos itens com conflito com documentos normativos, e (3) atribuição de responsável formal pelo documento. Enquanto indexado como fonte secundária, o assistente só utiliza o FAQ quando nenhum documento normativo cobre a pergunta, e deve sinalizar explicitamente: "Esta informação vem de um documento de referência interna, não de política oficial."

### 2. Tratamento de documentos contraditórios

Quando o pipeline recuperar dois ou mais trechos com informações conflitantes sobre o mesmo tema (identificados por sobreposição de assunto e data de emissão diferente):

- O assistente deve apresentar as duas versões separadamente, com: nome do documento, número da versão, data de emissão e o trecho específico conflitante.
- A versão mais recente deve ser apresentada primeiro.
- O assistente deve incluir aviso padronizado: *"Encontrei versões diferentes desta informação. Apresento ambas abaixo. Confirme com [área responsável] qual se aplica ao contrato ou contexto deste cliente."*
- O assistente não deve escolher uma versão silenciosamente, nem calcular um valor médio ou combinado.

**Critério verificável pelo QA:** Dado um par de perguntas que acione os dois PROC-042, a resposta deve conter os dois multiplicadores, a data de cada versão e o aviso de conflito. Ausência de qualquer desses três elementos = falha.

**Desindexação retroativa:** Quando uma nova versão de documento for indexada com metadado de vigência explícito (data de início de vigência), a versão anterior deve ser marcada como "histórico" e recuperada apenas quando a pergunta mencionar explicitamente o período anterior. Documentos marcados como "histórico" devem aparecer com aviso visual diferenciado na resposta.

### 3. Comportamento quando não há resposta na base

Quando nenhum chunk recuperado contiver informação suficiente para responder à pergunta do atendente:

- O assistente deve responder com mensagem padronizada: *"Não encontrei essa informação na base de conhecimento da NovaTech."*
- O assistente não deve complementar com conhecimento geral do modelo (ex: inferir prazos de entrega com base em rotas geográficas comuns).
- Deve indicar a próxima ação conforme o tema: Gestão de Riscos (ramal 4500) para cargas especiais, Comercial para contratos e descontos, supervisor imediato para demais casos.
- A resposta de "não encontrado" deve ser registrada automaticamente para análise de gaps recorrentes.

**Critério verificável pelo QA:** Para perguntas sobre temas não cobertos na base (ex: seguro de carga, frete padrão abaixo de 500kg), o assistente deve retornar a mensagem padronizada sem inventar valores. Qualquer valor numérico na resposta = falha.

### 4. Requisitos de atualização

- **Marco de início do prazo:** o prazo começa a contar a partir do momento em que o documento é publicado na fonte de origem (SharePoint, Confluence ou pasta de rede) com status "Aprovado".
- **Prazo máximo:** o documento deve estar disponível no assistente em até **24 horas** após a publicação aprovada.
- **Verificação:** o pipeline de ingestão deve registrar o timestamp de detecção e o timestamp de disponibilização no índice. A diferença entre os dois deve ser ≤ 24h.
- **Alerta de atraso:** se o prazo for excedido, o time de operações de IA deve ser notificado automaticamente.
- **Atualização de urgência:** documentos classificados como "Normativo Crítico" (ex: mudanças em POL ou PROC com impacto financeiro direto) devem ser indexados em até **4 horas**.

**Critério verificável pelo QA:** Publicar um documento de teste na fonte de origem com timestamp registrado e verificar o timestamp de disponibilidade no índice. Diferença > 24h = falha.

### 5. Requisitos de rastreabilidade

**Citação de fonte:**
- Toda resposta deve citar: nome do documento, número da versão (quando disponível), seção ou página específica.
- Formato mínimo aceitável: *"[Nome do documento], [versão], seção [X.X]."*
- Respostas sem citação de fonte = falha de guardrail, independentemente da correção do conteúdo.

**Exibição de trecho:**
- O trecho do documento de origem deve ser exibido junto à resposta, delimitado visivelmente (ex: caixa ou bloco recuado).
- Tamanho mínimo do trecho: a frase completa que contém a informação respondida.
- Tamanho máximo: o parágrafo ou item de lista que contém a frase, sem truncamento de contexto.
- O trecho deve ser o texto exato do documento recuperado — sem paráfrase ou edição pelo modelo.

**Perguntas multi-documento:**
Quando a resposta correta exigir informações de dois ou mais documentos (ex: SLA de um cliente Gold + prazo de devolução da POL-001), o assistente deve: citar cada fonte separadamente, apresentar os trechos de cada documento, e conectar as informações em linguagem natural apenas na síntese final. A síntese deve ser claramente separada dos trechos de fonte.

**Critério verificável pelo QA:** Para qualquer resposta, deve ser possível localizar manualmente o trecho exibido no documento de origem. Se o trecho não existir no documento citado = falha de rastreabilidade.

---

## Histórico de Iteração — O que mudou da v1 para a v2

| Gap identificado | Problema na v1 | Solução na v2 |
|---|---|---|
| Critério de "obsoleto" | Sem definição operacional | Vinculado ao campo de metadado "Status = Aprovado" na fonte |
| "Fonte secundária" | Comportamento indefinido | Condições explícitas de uso + aviso obrigatório ao atendente |
| Prazo de 24h | Marco de início vago | Definido como timestamp de publicação aprovada na fonte |
| Trecho "relevante" | Critério subjetivo | Tamanho mínimo/máximo definido; texto exato sem paráfrase |
| Desindexação retroativa | Ausente | Requisito de marcação como "histórico" com comportamento definido |
| Perguntas multi-documento | Ausente | Citação separada por fonte + síntese demarcada |

---

*Entregável gerado como parte do Exercício 1.3 — Product Specialist | DB1 IA First*
