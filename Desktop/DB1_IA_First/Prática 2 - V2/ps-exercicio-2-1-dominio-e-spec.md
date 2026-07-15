# PS — Exercício 2.1: Recorte de Domínio e Spec de Produto (SDD)

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Fase:** 2 — Estruturação

---

## Parte 1 — Recorte de Domínio

### Bounded Contexts

#### Contexto 1: Atendimento ao Cliente
**Dentro:** consulta ao assistente em linguagem natural, recebimento de resposta com fonte, fluxo de fallback para supervisor, fluxo de feedback sobre respostas incorretas, histórico de consultas por atendente.  
**Fora:** gerenciamento de chamados no sistema de tickets (Azure DevOps), comunicação com o cliente final, abertura de chamados no Portal do Cliente.  
**Relação:** consome dados dos contextos de Logística de Frete, Gestão de Devoluções e SLAs. É o único contexto exposto ao atendente via Teams.

#### Contexto 2: Logística de Frete
**Dentro:** cálculo de frete especial (>500kg), multiplicadores regionais, fatores de peso, descontos por volume, prazos adicionais para carga pesada.  
**Fora:** frete padrão (<500kg) — gap documental atual; tabela de fretes base mensal (referência externa).  
**Relação:** alimenta respostas do Atendimento ao Cliente; tem dependência direta do contexto de Gestão Documental (versão vigente da PROC-042).

#### Contexto 3: Gestão de Devoluções
**Dentro:** política de prazo (7 dias úteis), exceções por tipo de carga, procedimento de abertura de chamado, custos de frete reverso, devoluções parciais.  
**Fora:** tratamento de carga danificada em trânsito (gap documental); interceptação de carga em trânsito (PROC-088).  
**Relação:** cruza com Logística de Frete (custo do frete reverso usa os mesmos multiplicadores) e com SLAs (prazo de triagem definido pelo tier do cliente).

#### Contexto 4: SLAs e Contratos
**Dentro:** classificação de clientes por tier (Gold/Silver/Standard), prazos de resposta e resolução por tier, definição de incidente crítico, penalidades por descumprimento.  
**Fora:** conteúdo dos contratos individuais, negociação de SLA fora dos tiers padrão.  
**Relação:** consultado transversalmente — o tier do cliente determina o comportamento do assistente em tempo de resposta e prioridade de fallback.

#### Contexto 5: Gestão Documental
**Dentro:** versionamento de documentos normativos, controle de vigência, identificação de contradições entre versões, processo de atualização no pipeline de ingestão.  
**Fora:** criação e edição de documentos (responsabilidade das áreas Operações, Compliance e Comercial).  
**Relação:** fornece a base de verdade para todos os outros contextos.

---

### Linguagem Ubíqua do Domínio

| Termo | Definição canônica | Por que importa para agentes |
|---|---|---|
| **Carga perigosa** | Mercadoria classificada nas classes 1 a 6 da ANTT (Resolução 5.947/2021) | Sem essa definição, o agente pode tratar "produto químico" como sinônimo impreciso |
| **Frete especial** | Frete aplicável a cargas com peso acima de 500kg | Sem limiar explícito, o agente pode aplicar a fórmula a cargas padrão |
| **Frete padrão** | Frete para cargas até 500kg — não coberto na base documental atual | O agente deve saber que é gap e não inventar resposta |
| **Multiplicador regional** | Fator aplicado sobre o valor base por região — usar sempre a v2 (nov/2023) salvo chamados anteriores a 01/12/2023 | Dois valores coexistem na base; sem regra explícita o agente mistura versões |
| **Tier Gold** | Cliente com contrato anual > R$ 500.000 ou > 200 operações/mês | "Gold" é um tier de cliente, não o metal |
| **Tier Silver** | Contrato entre R$ 100.000–R$ 500.000 ou 50–200 operações/mês | — |
| **Tier Standard** | Todos os demais clientes — não existe tier Platinum ou outro | FAQ menciona "Platinum"; o agente deve rejeitar esse termo |
| **Incidente crítico** | Chamado que atende ao menos um dos 4 critérios da SLA-2024 seção 3 | Sem definição formal, o agente pode classificar incorretamente a urgência |
| **CT-e** | Conhecimento de Transporte Eletrônico — documento obrigatório para abertura de chamado de devolução | Abreviação não óbvia |
| **SLA de resposta** | Tempo até o primeiro retorno ao cliente — distinto de SLA de resolução | A confusão entre os dois gera informação errada |
| **SLA de resolução** | Tempo até o problema ser efetivamente resolvido | — |
| **Coleta reversa** | Processo logístico de retirada de mercadoria devolvida no endereço do cliente | Termo técnico do domínio |
| **Cadeia de frio** | Controle de temperatura contínuo; ruptura por > 30 min invalida devolução padrão | Critério técnico com limiar numérico específico |
| **Vigência** | Período em que uma versão de documento é a referência oficial — PROC-042 v2 vigente a partir de 01/12/2023 | Sem metadado de vigência, o agente não sabe qual versão usar |

---

## Parte 2 — requirements.md (versão final após iteração)

> Arquivo: `specs/query-endpoint/requirements.md`

```markdown
# requirements.md — Query Endpoint
Módulo: query-endpoint
Autor: Product Specialist
Status: Aprovado

## Outcomes

O-01: O atendente recebe uma resposta relevante para sua dúvida em menos de 30 segundos,
      sem precisar abrir nenhum documento manualmente.
O-02: O atendente sabe exatamente de qual documento a resposta veio e consegue verificar
      por conta própria se quiser.
O-03: Quando o assistente tem baixa confiança, o atendente recebe aviso visível e orientação
      clara sobre o que fazer a seguir.
O-04: Quando dois documentos têm informações diferentes sobre o mesmo tema, o atendente
      vê as duas versões com datas — não recebe uma resposta que mistura as duas.
O-05: O atendente nunca recebe um valor numérico que não esteja explicitamente na
      documentação indexada.

## Scope Boundaries

DENTRO (contexto: Atendimento ao Cliente):
- Receber pergunta em linguagem natural via Teams
- Recuperar chunks relevantes da base documental indexada
- Gerar resposta com citação de fonte
- Detectar e sinalizar contradições entre versões de documentos
- Sinalizar baixa confiança e sugerir próximo passo
- Respeitar o tier do cliente no contexto da consulta (quando informado)

FORA:
- Ingestão e atualização de documentos (módulo: pipeline-ingestao)
- Registro e roteamento de feedback (módulo: feedback-api)
- Interface conversacional no Teams (módulo: teams-bot)
- Consulta a sistemas externos (Azure DevOps, Portal do Cliente, ERP)
- Respostas sobre frete padrão (<500kg) — gap documental

## Constraints

C-01: O assistente NUNCA deve gerar valores numéricos (prazos, multiplicadores, SLAs)
      que não estejam literalmente nos chunks recuperados.
C-02: O assistente NUNCA deve afirmar que carga perigosa (classes 1-6 ANTT) pode ser
      devolvida pelo processo padrão.
C-03: O assistente NUNCA deve reconhecer tiers além de Gold, Silver e Standard.
C-04: Quando duas versões de um documento estiverem disponíveis, o assistente DEVE
      apresentar ambas — campo source_documents (array) com id, version e date de cada.
C-05: Toda resposta DEVE incluir campo source_document com identificador e seção.
C-06: O assistente DEVE responder em português formal.
C-07: Context budget: ~4K tokens system prompt + ~8K chunks (máx. 5 chunks ~1.500t)
      + pergunta + histórico limitado a 3 turnos (par pergunta+resposta).
C-08: Toda resposta DEVE incluir campo confidence (high/medium/low).
      Respostas com confidence=low DEVEM exibir aviso visível ao atendente.

## Prior Decisions

ADR-0001: Modelo LLM — Azure OpenAI GPT-4o.
ADR-0002: Context budget — 4K system + 8K chunks + pergunta + 3 turnos histórico.
ADR-0003: Documentos contraditórios — metadado de vigência; priorizar versão mais recente.
ADR-0004: Pipeline de RAG — Azure AI Search + Azure OpenAI.
Spec RAG (Fase 1): atualização em até 24h após publicação aprovada na fonte.

## Verification Criteria

VC-01: p95 de latência ≤ 30s sob carga de até 100 queries simultâneas.
VC-02: 100% das respostas incluem campo source_document com nome e seção.
VC-03: Queries sobre devolução de carga perigosa retornam negativa explícita e ramal 4500.
VC-04: Queries sem correspondência na base retornam mensagem padrão sem valores numéricos.
VC-05: Queries com documentos contraditórios: source_documents contém array com ambos
       (id, version, date). Resposta com apenas um = falha.
VC-06: Nenhuma resposta contém tier além de Gold, Silver e Standard.
VC-07: Respostas com confidence=low exibem aviso visível. Ausência = falha.
```

---

## Parte 3 — Histórico de Iteração com Tech Lead

| # | Ambiguidade | Impacto | Resolução na v2 |
|---|---|---|---|
| 1 | VC-01 sem definição de "condições normais" | QA não sabe quando é violação em pico | Adicionado p95 e carga de até 100 queries |
| 2 | C-07 "turno" não definido | Dev não sabe o que incluir no histórico | Turno = par pergunta+resposta; máximo 3 pares |
| 3 | VC-05 sem contrato de formato | Dev não sabe o que implementar | `source_documents` array com id, version e date |
| 4 | Ausência de definição de confiança | Impossível implementar O-03 sem campo estruturado | Novo C-08: campo `confidence` com high/medium/low |

---

*Entregável gerado como parte do Exercício 2.1 — Product Specialist | DB1 IA First*
