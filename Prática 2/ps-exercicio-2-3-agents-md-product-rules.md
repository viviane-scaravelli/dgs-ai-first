# PS — Exercício 2.3: Seção "Product Rules & Guardrails" do AGENTS.md

**Papel:** Product Specialist  
**Projeto:** Assistente de IA para Atendimento — NovaTech  
**Fase:** 2 — Estruturação  
**Arquivo de destino:** `AGENTS.md` (raiz do repositório)

---

> O trecho abaixo é a seção completa pronta para ser inserida no AGENTS.md do projeto.

---

```markdown
## Product Rules & Guardrails

> Esta seção define as regras de produto, os guardrails do assistente e o glossário de domínio
> que todo agente de IA deve respeitar ao gerar código, testes, prompts ou documentação
> relacionados ao assistente NovaTech. As regras são prescritivas — leia como DEVE/NÃO DEVE,
> não como sugestões.
>
> Spec de referência: `specs/query-endpoint/requirements.md`
> Guardrails completos: `docs/guardrails.md`
> Glossário de domínio: ver seção 4 abaixo.

---

### 1. Regras de comportamento do assistente

#### 1.1 DEVE (comportamentos obrigatórios)

- **DEVE** incluir o campo `source_document` em toda resposta gerada, com o identificador
  do documento (ex: `POL-001`) e a seção (ex: `seção 3.2`). Respostas sem esse campo
  são inválidas e devem ser bloqueadas pelo `response-validator.ts`.

- **DEVE** incluir o campo `confidence` em toda resposta com um dos três valores:
  `"high"`, `"medium"` ou `"low"`. Valor ausente ou inválido = resposta bloqueada.

- **DEVE** responder em português formal, independentemente do idioma da pergunta.
  Instrução obrigatória no system prompt: ver `prompts/system-prompt.md`.

- **DEVE** apresentar ambas as versões de um documento quando houver contradição,
  utilizando o campo `source_documents` (array) com `id`, `version` e `date` de cada
  documento. Nunca escolher uma versão silenciosamente.

- **DEVE** injetar aviso de baixa confiança visível ao atendente quando `confidence=low`.
  O aviso é responsabilidade do `response-builder.ts` — não do modelo.

#### 1.2 NÃO DEVE (comportamentos proibidos)

- **NÃO DEVE** gerar valores numéricos (prazos, multiplicadores regionais, percentuais
  de SLA, fatores de peso) que não estejam literalmente presentes nos chunks recuperados
  para aquela query. Se o valor não estiver nos chunks, retornar `confidence=low` e
  indicar próximo passo.

- **NÃO DEVE** afirmar, direta ou indiretamente, que carga perigosa (classes 1 a 6
  da ANTT, conforme Resolução 5.947/2021) pode ser devolvida pelo processo padrão.
  Resposta correta: indicar Gestão de Riscos (ramal 4500) para tratamento individual.
  O `response-validator.ts` bloqueia respostas que combinem "carga perigosa" com
  afirmações de elegibilidade para devolução.

- **NÃO DEVE** reconhecer, confirmar ou elaborar sobre tiers de cliente além de
  `Gold`, `Silver` e `Standard`. Tiers inválidos incluem (não limitado a):
  `Platinum`, `Bronze`, `Diamond`, `Premium`, `Elite`. O `response-validator.ts`
  bloqueia respostas que contenham esses termos.

- **NÃO DEVE** apresentar informações exclusivamente do FAQ-Atendimento como regras
  normativas sem o aviso: `"[Fonte informal — confirme com documento normativo ou
  supervisor antes de usar]"`. Chunks do FAQ têm metadado `source_type: "faq"` —
  use-o para detectar a origem.

- **NÃO DEVE** usar `console.log` em nenhum arquivo do projeto. Use sempre o logger
  centralizado em `src/shared/logger.ts` (pino).

#### 1.3 QUANDO EM DÚVIDA (comportamentos de fallback)

- Quando `confidence=low` ou score de similaridade do Azure AI Search estiver abaixo
  do threshold configurado: prefixar a resposta com aviso padronizado (injetado pelo
  `response-builder.ts`) e incluir o próximo passo específico por tema:
  - Cargas especiais / perigosas → Gestão de Riscos, ramal 4500
  - Frete e descontos → Comercial
  - SLAs e contratos → gerente de conta do cliente
  - Outros → supervisor imediato

- Quando dois chunks com o mesmo `document_id` e datas diferentes forem recuperados:
  apresentar ambas as versões, priorizar a mais recente (maior `vigencia_inicio`),
  e incluir `conflict_warning: true` no JSON de retorno.

- Quando a pergunta não tiver nenhum chunk com score acima do threshold mínimo:
  retornar a mensagem padrão `"Não encontrei essa informação na base de conhecimento
  da NovaTech."` seguida do próximo passo. Nunca retornar resposta vazia.

---

### 2. Restrições que impactam geração de código

Todo código gerado por agentes para este projeto deve respeitar:

- **Contrato da API de resposta:** toda função que retorna uma resposta do assistente
  deve incluir os campos obrigatórios abaixo. Gerar código sem esses campos é inválido:

  ```typescript
  interface AssistantResponse {
    answer: string;                      // Resposta em português formal
    confidence: "high" | "medium" | "low";
    source_document: {
      id: string;                        // Ex: "POL-001"
      section: string;                   // Ex: "seção 3.2"
      version?: string;                  // Ex: "v3.1"
    };
    source_documents?: Array<{           // Presente apenas quando há contradição
      id: string;
      version: string;
      date: string;                      // ISO 8601
    }>;
    conflict_warning?: boolean;          // true quando source_documents tiver 2+ versões
    next_step?: string;                  // Obrigatório quando confidence=low
  }
  ```

- **Validação determinística:** o `src/services/response-validator.ts` é o guardião
  das regras determinísticas. Qualquer nova regra de enforcement de guardrail que possa
  ser verificada programaticamente DEVE ser implementada neste arquivo, não apenas
  no system prompt.

- **Injeção de avisos:** o `src/functions/query/response-builder.ts` é responsável
  por injetar avisos de `confidence=low` e `conflict_warning`. Não duplicar essa
  lógica em outros arquivos.

- **Logging:** usar exclusivamente `src/shared/logger.ts`. Proibido `console.log`,
  `console.error` ou qualquer outro método direto de console.

- **Validação de input:** usar Zod em `src/functions/query/validator.ts`.
  Nunca validar input manualmente com `if/else` ou `typeof`.

---

### 3. Referências a specs e documentos do repositório

| Artefato | Caminho | Descrição |
|---|---|---|
| Spec do query endpoint | `specs/query-endpoint/requirements.md` | Outcomes, scope boundaries, constraints e VCs do módulo principal |
| Guardrails completos | `docs/guardrails.md` | Documento de guardrails com enforcement e rastreabilidade a incidentes |
| System prompt | `prompts/system-prompt.md` | Prompt principal versionado — toda alteração exige registro em `prompts/prompt-changelog.md` |
| Tipos TypeScript do domínio | `src/shared/types.ts` | Interfaces do domínio, incluindo `AssistantResponse` |
| Validador de resposta | `src/services/response-validator.ts` | Enforcement determinístico de guardrails |
| Response builder | `src/functions/query/response-builder.ts` | Injeção de avisos e montagem da resposta final |
| ADR contexto e budget | `docs/adr/0002-estrategia-contexto.md` | Decisão sobre context budget (4K system + 8K chunks + 3 turnos) |
| ADR documentos contraditórios | `docs/adr/0003-documentos-contraditorios.md` | Decisão sobre tratamento de versões conflitantes |

---

### 4. Glossário de linguagem ubíqua do domínio

> Todo agente que gerar código, testes, documentação ou prompts para este projeto
> DEVE usar os termos abaixo com as definições canônicas aqui especificadas.
> Não use sinônimos, abreviações informais ou termos em inglês para esses conceitos
> a menos que explicitamente indicado.

| Termo | Definição canônica | Observação para agentes |
|---|---|---|
| `carga perigosa` | Mercadoria classificada nas classes 1 a 6 da ANTT (Resolução 5.947/2021). Inclui: explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes/peróxidos (5), tóxicos/infectantes (6) | Nunca substituir por "produto químico", "material perigoso" ou variações imprecisas. Em código, usar `dangerousGoods` como nome de variável/campo |
| `frete especial` | Frete aplicável exclusivamente a cargas com peso **acima de 500kg** | Nunca aplicar a fórmula de frete especial a cargas abaixo de 500kg |
| `frete padrão` | Frete para cargas com peso **até 500kg** — **não coberto** na base documental atual | O assistente deve retornar "não encontrado" para perguntas sobre frete padrão. Em código: `standardFreight` |
| `multiplicador regional` | Fator aplicado sobre o valor base do frete por região de destino. **Versão vigente: PROC-042-v2 (nov/2023)** | Usar sempre v2 salvo para chamados anteriores a 01/12/2023. Em código: `regionalMultiplier` |
| `tier Gold` | Cliente com contrato anual > R$ 500.000 **ou** > 200 operações/mês | "Gold" é classificação de cliente NovaTech, não referência ao metal. Em código: `"gold"` (lowercase) |
| `tier Silver` | Cliente com contrato entre R$ 100.000–500.000 **ou** 50–200 operações/mês | Em código: `"silver"` (lowercase) |
| `tier Standard` | Todos os demais clientes | Em código: `"standard"` (lowercase) |
| `tier inválido` | Qualquer tier além de Gold, Silver e Standard | Exemplos: Platinum, Bronze, Diamond, Premium, Elite. O validador bloqueia respostas com esses termos |
| `incidente crítico` | Chamado que atende ao menos um dos critérios da SLA-2024 seção 3: carga >R$100k sem rastreamento por 6h+; carga perigosa com irregularidade; 5+ chamados idênticos em 24h; risco à segurança | Em código: `criticalIncident: boolean` |
| `CT-e` | Conhecimento de Transporte Eletrônico — documento obrigatório para abertura de chamado de devolução | Não abreviar de outra forma. Em código: `cteNumber` |
| `SLA de resposta` | Tempo até o **primeiro retorno** ao cliente (mesmo que "estamos verificando") | Distinto de SLA de resolução. Em código: `responseSlaDuration` |
| `SLA de resolução` | Tempo até o **problema ser efetivamente resolvido** | Em código: `resolutionSlaDuration` |
| `coleta reversa` | Processo logístico de retirada de mercadoria devolvida no endereço do cliente | Em código: `reversePickup` |
| `vigência` | Período em que uma versão de documento é a referência oficial | Em código: campo `vigencia_inicio` (ISO 8601) no metadado do chunk |
| `chunk` | Trecho de documento indexado no Azure AI Search, com metadados de origem | Em código: `DocumentChunk` (ver `src/shared/types.ts`) |
| `confidence` | Nível de confiança do assistente na resposta: `"high"`, `"medium"`, `"low"` | Sempre lowercase, sempre um dos três valores. Nunca usar números ou percentuais |
```

---

*Entregável gerado como parte do Exercício 2.3 — Product Specialist | DB1 IA First*
