# Config Completo — Agente de IA do CRM Persia

Referencia tecnica dos campos, defaults, limites e tabelas. Contrato de tipos em
`packages/shared/src/ai-agent/` (READ-ONLY apos merge — muda via migration). Server actions
em `apps/crm/src/actions/ai-agent/`. Runtime em `apps/crm/src/lib/ai-agent/`.

> Os numeros abaixo refletem o estado do sistema em jun/2026. Confirme no codigo
> (`packages/shared/src/ai-agent/types.ts` + migrations) antes de afirmar valores criticos.

---

## 1. Campos principais — tabela `agent_configs`

| Campo | Tipo | Default / Range | Significado |
|---|---|---|---|
| `id` | UUID | auto | PK |
| `organization_id` | UUID | — | Dono |
| `name` | TEXT | obrigatorio | Nome (recomendado <100 chars) |
| `description` | TEXT | null | Opcional |
| `status` | enum | `draft` | `draft` \| `active` \| `paused` |
| `scope_type` | enum | `global` | `global` \| `department` \| `pipeline` |
| `scope_id` | UUID | null | Aponta departamento/pipeline quando escopo restrito |
| `model` | TEXT | `gpt-5-mini` | `gpt-5` \| `gpt-5-mini` \| `gpt-4o` \| `gpt-4o-mini` |
| `system_prompt` | TEXT | `''` | Prompt cru (ou null se usar structured) |
| `structured_prompt_config` | JSONB | null | Editor SDR (recomendado) — ver secao 3 |
| `behavior_mode` | enum | `flow` | so `flow` aceito (canvas) |
| `is_primary` | bool | false | No max 1 TRUE por org (roteador de entrada) |
| `new_lead_stage_id` | UUID | null | Stage CRM pra novos leads |
| `on_appointment_created_stage_id` | UUID | null | Stage ao IA criar agendamento |
| `guardrails` | JSONB | ver sec.9 | Limites de execucao |
| `debounce_window_ms` | int | 10000 | 0–60000 |
| `humanization_config` | JSONB | ver sec.6 | Comportamento humano |
| `validation` | JSONB | ver sec.7 | Validacao de resposta |
| `structured_sources` | JSONB[] | [] | Fontes de dados (json/mcp) |
| `message_templates` | JSONB[] | [] | Templates de mensagem |
| `pause_segment_ids` | string[] | [] | Silencia agente nesses segmentos |
| `calendar_connection_id` | UUID | null | Conexao Google Calendar p/ agendamento |

Arquivos: `packages/shared/src/ai-agent/types.ts:270-349`, `apps/crm/supabase/migrations/017_ai_agent_core.sql`.

---

## 2. Modelos (provider = OpenAI exclusivo)

| Modelo | Input $/1M | Output $/1M | Uso |
|---|---|---|---|
| `gpt-5` | 1.25 | 10 | Raciocinio pesado |
| `gpt-5-mini` | 0.25 | 2 | **Default** — melhor custo/qualidade |
| `gpt-4o` | 2.5 | 10 | Legado |
| `gpt-4o-mini` | 0.15 | 0.6 | Fluxos simples/baratos; tambem usado interno (sumarizacao) |

Custo por run: `calculateCostUsdCents(model, tokensIn, tokensOut)` em `cost.ts`.

---

## 3. Structured Prompt (editor SDR) — `structured_prompt_config`

Compilado para `system_prompt` no save via `compileStructuredPrompt()`
(`packages/shared/src/ai-agent/structured-prompt.ts:54-143`). Componentes:

1. **Identity** (6 campos, interpolados nas regras): `agent_name`, `company`, `segment`,
   `channel`, `region`, `goal`.
2. **Personality:**
   - `tone.preset`: `direct_commercial` | `consultive_empathic` | `formal_institutional` | `casual_youth`
   - `tone.custom_instruction`: override
   - `personality_traits[]`: bullets opcionais
3. **Master prompt:** texto livre (interpola identidade).
4. **Commercial rules:** array de fases — `{ title, profile_label, description[] }`.
5. **Prohibited actions:** lista do que o agente nunca faz.

Ordem do prompt compilado: `# Data Atual` → Identidade → Personalidade → Objetivo →
Fluxo comercial (master + regras) → Acoes proibidas.

**Injetados em runtime** (alem do prompt): bloco de knowledge (RAG), structured sources,
templates, lista de tools disponiveis, contexto do lead (nome/telefone/email), catalogos
(usuarios/agentes/tags).

### 3.1 Como o editor SDR realmente recebe os dados (NAO e um JSON unico)

O editor (`packages/ai-agent-ui/src/components/StructuredPromptEditor.tsx` + `RulesTab.tsx`)
divide o structured prompt em blocos com formas de entrada DIFERENTES. Nao existe "importar o
agente inteiro num JSO so" — cada bloco e separado:

| Bloco no editor | Campo | Como entra | Shape do JSON (quando ha import) |
|---|---|---|---|
| Cadastro de Informacoes | `identity` (6 campos) | **digitar** (form) | — sem JSON |
| Personalidade | `personality_traits` + `tone` | **digitar** (chips + preset + instrucao) | — sem JSON |
| Apresentacao do Agente | `master_prompt` | **digitar** (textarea) | — sem JSON |
| Roteiro de Abordagem | `commercial_rules` | botao **Importar JSON** (replace/merge) | `array` de `{title*, profile_label, description}` — `description` e **string** (bullets em `\n`); `id` opcional (auto nanoid) |
| Regras Gerais | `prohibited_actions` | botao **Importar JSON** | `array` de **strings** `["regra 1","regra 2"]` |
| Templates de mensagem | `message_templates` | botao **Importar JSON** | `array` de `{key*, name*, message*, usage, mode}` — `key` so `[a-z0-9_]`; `mode` opcional (default `ai_suggestion`); **`no_split` e IGNORADO no import** (so toggle na UI, e so vale em `ai_suggestion`) |
| Fontes estruturadas | `structured_sources` | modal "Adicionar fonte" (cola o `data`) | objeto JSON inline, **uma fonte por vez** (sem import em lote) |

Validadores: `parseRulesJson` (roteiro, exige `title`), `parseGeneralRulesJson` (array de
strings nao-vazias, dedup), `parseTemplateJson` (exige `key`/`name`/`message`). Roteiro e Regras
tem modo **Substituir** vs **Mesclar**. Ao montar o material para o usuario, entregue **arquivos
separados** por bloco (um para roteiro, um para regras, um para templates) + os campos manuais
em texto — nao um mega-JSON, que nao importa.

### 3.2 Fase padrao OBRIGATORIA: "Fora do escopo" → `stop_agent`

**Todo agente deve ter, como ultima fase do roteiro, uma fase "Fora do escopo"** que chama
`stop_agent` (pausa a IA e passa pra um atendente humano). Serve de rede de seguranca quando o
lead foge do assunto comercial ou pede explicitamente uma pessoa.

Modelo (adapte `profile_label`/empresa ao caso):
- **title:** `Fase N · Fora do escopo`
- **profile_label** (frase modelo): `Certo, ja ja damos continuidade por aqui.`
- **description (bullets):**
  - Quando usar: o lead pede algo fora do atendimento comercial (assunto nao relacionado ao
    produto, suporte tecnico aprofundado, encaminhamento a terceiros) ou pede para falar com
    uma pessoa/atendente.
  - Responda curto e humano, exatamente: `Certo, ja ja damos continuidade por aqui.`
  - Se for pedido de audio: `No momento nao consigo enviar audio por aqui. Certo, ja ja damos continuidade por aqui.`
  - Em seguida chame `stop_agent`.
  - Nao explique demais nem prometa prazo de retorno. Use so quando for realmente fora do
    escopo — duvida comum sobre o produto continua no fluxo normal.

Requer a tool `stop_agent` habilitada e no `enabled_tools` do flow, e `allow_human_handoff: true`
nos guardrails (senao `stop_agent` vira no-op).

---

## 4. Tools — tabela `agent_tools`

Um-para-muitos com `agent_configs`. Campos: `name` (unico por agente, snake_case),
`description` (visivel ao LLM), `input_schema` (JSON Schema strict: todos em `required[]`,
`additionalProperties:false`), `execution_mode` (`native` | `n8n_webhook` | `mcp`),
`native_handler`, `webhook_url`/`webhook_secret`, `mcp_server_id`, `is_enabled`.

### Presets nativos (40+) por categoria
- **Handoff:** `stop_agent`, `close_conversation`
- **Transferencia:** `transfer_to_user`, `transfer_to_agent`
- **Tags:** `add_tag`, `remove_tag`
- **Atribuicao:** `assign_product`, `assign_department`, `assign_source`
- **Distribuicao:** `round_robin_user`, `round_robin_agent`
- **Notificacao:** `trigger_notification`
- **Midia:** `send_media`, `send_audio`
- **Agendamento:** `create_appointment`, `list_lead_appointments`, `cancel_appointment`,
  `reschedule_appointment`, `get_available_slots`, `confirm_appointment`
- **Funil:** `move_pipeline_stage`
- **Controle de fluxo:** `emit_event` (ramifica pra saidas nomeadas de no)
- **Lead:** `set_lead_custom_field`

Presets sao "gateados" por `shipped_in_pr` (UI desabilita o que ainda nao subiu).
Criar tool: `createToolFromPreset(handler)` / `createCustomTool` / `createMcpTool`
(`apps/crm/src/actions/ai-agent/tools.ts`). Catalogo em `tool-presets.ts`.

**Webhook (n8n):** HMAC-SHA256, secret min 32 chars, allowlist de hostname (anti-SSRF).

---

## 5. Knowledge / RAG e structured sources

### structured_sources (`types.ts:20-74`)
- Tipo `json`: dados inline + `data_type` (`pricing_tables`, `products`, `services`...). Injetado
  no prompt, sem embedding. **Use pra dados pequenos.**
- Tipo `mcp`: referencia `mcp_server_connections(id)` + `allowed_tools[]` opcional ([] = todas).

### RAG (`agent_knowledge_sources`, migration 022)
- Embeddings **Voyage** (voyage-3, 1024-dim). Chunk 512 tokens, overlap 64.
- `source_type`: `faq` | `document`. Retrieval top_k (default 3, range 1–10), distancia cosine 0.75.
- **Use pra documentos grandes / FAQ extenso.**

---

## 6. Humanizacao — `humanization_config` (migration 041)

| Campo | Default | Range | Efeito |
|---|---|---|---|
| `pause_keywords` | ["PAUSAR","HUMANO","STOP IA"] | — | Lead pausa a IA |
| `resume_keywords` | ["ATIVAR","IA ON","VOLTAR IA"] | — | Lead reativa |
| `auto_pause_minutes` | 30 | 0–1440 | Pausa IA quando operador responde (0=off) |
| `split_enabled` | false | — | Quebra resposta longa em varias msgs |
| `split_threshold_chars` | 200 | 50–1000 | Limite p/ quebrar |
| `split_delay_seconds` | 2 | 0–30 | Intervalo entre msgs |
| `business_hours_enabled` | false | — | So responde 9h–18h seg-sex (America/Sao_Paulo) |
| `after_hours_message` | — | — | Mensagem fora de horario (cooldown 6h) |

Normalizado em runtime (`humanization.ts:13-150`).

---

## 7. Validacao de resposta — `validation` (migration 101)

| Campo | Default | Limite |
|---|---|---|
| `enabled` | false | — |
| `max_chars` | — | 0–4000 (0=sem limite) |
| `one_question_only` | false | — |
| `block_empty_response` | true | — |
| `forbidden_phrases` | [] | max 50 itens, 200 chars cada |
| `blocked_promises` | [] | idem |
| `on_block` | — | `rewrite` \| `fallback` \| `pause_ai` \| `alert_only` |
| `fallback_message` | — | max 1000 chars |

Arquivo: `validation.ts:14-37`.

---

## 8. Followups e debounce

### Debounce (`debounce.ts`, migration 019)
- `debounce_window_ms` default 10000, range 0–60000. Agrega msgs fragmentadas em 1 run.
- Fila em `pending_messages`. Flush por cron (~2s) → `/api/ai-agent/debounce-flush`.
- 0 = responde na hora (OTP/pagamento). Maior = chats que digitam em pedacos.

### Followups (`followups.ts`, migration 027)
- Tabela `agent_followups`, **max 5 por agente**. Delay 1–720h (`delay_hours`) ou preset teste 5min.
- `message_type`: `text` (fixo) | `ai_generated` (IA escreve baseado no historico).
- `send_window_start/end` (default 08:00–18:00), `require_ai_active`, `is_enabled`, `order_index`.
- Gatilho: `last_inbound_message_at` + delay. Idempotente por (followup_id, conversation_id).
- Estado em `agent_followup_conversation_states` (waiting|eligible|sent|paused|cancelled|finished).

---

## 9. Guardrails — `guardrails` (JSONB)

| Campo | Default | Significado |
|---|---|---|
| `max_iterations` | 5 | Limite do loop de tool-use |
| `timeout_seconds` | 30 | Wall-clock por run (1–300) |
| `cost_ceiling_tokens` | 20000 | Orcamento tokens in+out por run |
| `allow_human_handoff` | true | Se false, `stop_agent` vira no-op |

Aplicados em runtime, nao pelo LLM. `assertWithinCostLimits()` antes do run.

---

## 10. Multi-agente / roteamento

- **1 primario** por org (`is_primary=true`) = roteador de entrada (pega todo lead novo).
- **Secundarios** via `agent_entry_conditions` (migration 045). 6 tipos de condicao:
  `tag_match`, `segment_match`, `message_contains` (1a msg), `pipeline_stage_match`,
  `lead_status_match`, `channel_match` (whatsapp/instagram). Avaliacao OR, ordenada por
  `priority DESC, created_at ASC` (primeiro match ganha). Conversa fica "presa" no agente via
  `agent_conversations.config_id` (stickiness).
- **Intent router** (opcional, env `NATIVE_AI_INTENT_ROUTING=1`): se nenhuma condicao bate,
  classifica a 1a msg com gpt-4o-mini contra o `description` dos secundarios. Fallback → primario.
- **Transferencia agente→agente:** tool nativa `transfer_to_agent` (PR3, tool-presets.ts:106).
  Input: `target_agent_name` (ou `agent_config_id`) + `reason`. Runtime: troca `config_id`,
  zera `current_node_id` (entra pelo entry node do novo flow), `ai_control_epoch +1` (invalida runs
  em voo), **preserva** `history_summary` e `variables`, cancela follow-ups do anterior, audita em
  `agent_transfers` (migration 188). Alvo precisa estar `active` e ter flow.
- **Saidas distintas:** `transfer_to_agent` (→ outro agente IA) · `transfer_to_user` (→ humano,
  seta `human_handoff_at`) · `stop_agent` (pausa, humano retoma) · `close_conversation` (encerra).
- **1 flow por agente** (`agent_flows.agent_config_id`). Sem chain direta — handoff so via `transfer_to_agent`.
- Secundarios: sem limite no DB; pratico 2–5 por org. `pause_segment_ids`: lead nesses segmentos → agente pula.
- Handler: `lib/ai-agent/tools/transfer-to-agent.ts`; routing: `executor.ts` (pickAgentForLead);
  condicoes: `packages/shared/src/ai-agent/entry-conditions.ts`.

---

## 11. Flow canvas (`agent_flows`, migration 054)

Nos: `entry`, `ai_agent`, `action`, `condition`. Gatilhos de entrada: `conversation_started`,
`keyword_match`, `segment_entered`, `pipeline_stage_entered`, `appointment_created`,
`appointment_confirmed`. Edges: `default` | `tool_success:<tool>` | `<handle>` (via `emit_event`).
Max iteracoes do runner: 20. `enabled_tools` (JSONB) controla tools por flow.

---

## 12. Auditoria / observabilidade

| Tabela | Conteudo |
|---|---|
| `agent_runs` | 1 por execucao inbound: status, model, tokens_in/out, cost_usd_cents, duration_ms, is_test |
| `agent_steps` | 1 por chamada LLM/tool/guardrail: step_type, tool_id, native_handler, input, output |
| `agent_conversations` | Estado runtime: current_node_id, history_summary, variables, human_handoff_at |

Filtre `is_test=true` (runs do simulador) dos dashboards.

---

## 13. Arquivos-chave

- Tipos: `packages/shared/src/ai-agent/types.ts`, `flow.ts`, `entry-conditions.ts`
- Compilacao: `structured-prompt.ts`, `humanization.ts`, `validation.ts`, `debounce.ts`, `followups.ts`, `cost.ts`, `tool-presets.ts`
- Actions: `apps/crm/src/actions/ai-agent/{configs,tools,followups,knowledge,entry-conditions,tester}.ts`
- UI: `apps/crm/src/app/(dashboard)/automations/agents/[id]/{page,agent-editor-client}.tsx`, `agents-list-client.tsx`
- Runtime: `apps/crm/src/lib/ai-agent/{executor,flow/runner,flow/openai-runtime,tools/registry,summarization}.ts`
- Migrations: `apps/crm/supabase/migrations/017..188_ai_agent_*.sql`

---

## 14. Gotchas / licoes de campo (jun/2026 — aprendido na pratica)

Coisas que travam de verdade ao montar um agente, em ordem de quanto custam tempo:

1. **Mudou JSON no PC ≠ mudou no agente.** Regras/Roteiro/Templates so valem apos
   **Importar JSON (Substituir) + Salvar o agente** no editor. Testar sem reimportar e o erro nº 1.
2. **Simulador NAO chama webhook tool.** Em dry-run o runtime retorna stub `{simulated:true}`
   (runner.ts ~1616) — nunca bate no n8n nem traz valor real. Webhook/MCP so executam em
   **mensagem real**. Pra testar cotacao via tool, use o WhatsApp real, nao o Simulador.
3. **Criar a tool ≠ tool disponivel pro LLM.** O runtime so expoe tools no `enabled_tools` do
   FLOW (allowlist por canvas, persistido ao **salvar o canvas**). Excecao: tools `n8n_webhook`
   com `is_enabled=true` sao auto-incluidas (jun/2026, runner.ts ~326) — mas isso e recente;
   build antiga exige a tool no `enabled_tools`. Sintoma de tool ausente: agente diz "vou verificar".
4. **UI de tool webhook custom esta atras de feature flag** `native_agent_tools_ui` (default OFF,
   feature-flags.ts). Sem a flag, a aba Ferramentas e placeholder. O componente
   (`CustomWebhookToolSheet`) existe mas pode nao estar plugado. Allowlist de hostname e obrigatoria
   (org settings) e o resolve so IP publico (bloqueia IP privado/local → n8n precisa de dominio publico HTTPS:443).
5. **HMAC do webhook custom:** header `X-Persia-Signature: sha256=...` (secret + timestamp). O secret
   vai NOS DOIS lados (CRM assina, n8n valida). So no CRM = URL exposta (n8n nao valida).
6. **Prompt manda ler fonte OU chamar tool — escolha um.** Se as regras dizem "consulte a tabela",
   o agente le a fonte estruturada e NAO chama a tool de cotacao. Pra usar tool, as regras/fases
   precisam dizer "chame a ferramenta X". Referencie tools/fontes em backticks (`nome`) — o editor
   reconhece via @-mention.
7. **Fonte estruturada = dado que o agente LE; Tool = acao que ele EXECUTA.** Nao cadastre o schema
   de uma tool como fonte estruturada (polui contexto, nao serve).
8. **Erro 500 "Server Components render" ao criar tool:** mensagem generica esconde o real; o
   `digest` esta no log do servidor. Costuma ser a tela re-renderizando apos salvar (a tool ate
   foi criada — recarregue e confira) ou migration desatualizada no ambiente.
9. **Fallback seguro:** quando a UI de tools/canvas esta capenga, o caminho confiavel e
   **fonte estruturada** (preco/regra no prompt) — funciona sem depender do allowlist/webhook.
   A tool de cotacao e o "ideal" so quando a plataforma suporta de ponta a ponta.
