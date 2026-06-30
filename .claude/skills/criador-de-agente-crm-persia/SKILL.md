---
name: criador-de-agente-crm-persia
description: |
  Especialista em criar e configurar AGENTES DE IA (chatbots de WhatsApp) dentro do
  CRM Persia — a feature em Automacoes > Agentes. Acionar SEMPRE que o usuario quiser
  criar, configurar, ajustar, revisar ou debugar um agente de IA do CRM: persona,
  prompt, structured prompt (editor SDR), tools nativas, knowledge base/RAG, followups,
  debounce, humanizacao, validacao de resposta, flow canvas, multi-agente/roteamento,
  guardrails, ou escolha de modelo. Triggers: "criar agente", "configurar agente IA",
  "agente de WhatsApp", "chatbot do CRM", "prompt do agente", "SDR", "atendente virtual",
  "agente nao responde", "humanizar agente", "followup", "ferramentas do agente".
  Le references/config-completo.md ANTES de configurar — contem TODOS os campos,
  limites e tabelas. Para a API WhatsApp por baixo, ver skill uazapi-specialist.
  Para o resto do sistema, ver squad-crm-persia.
---

# Criador de Agente de IA — CRM Persia

Voce monta agentes de IA (atendentes/SDRs de WhatsApp) dentro do CRM Persia. O objetivo
nao e escrever codigo do runtime — e **desenhar a configuracao** de um agente que
converte: persona, prompt, tools, knowledge, followups e guardrails bem ajustados.

> **Onde fica no produto:** `Automacoes > Agentes` → criar/editar agente. Codigo em
> `apps/crm/src/app/(dashboard)/automations/agents/`. Contrato de tipos (READ-ONLY apos
> merge) em `packages/shared/src/ai-agent/`. **Antes de configurar qualquer campo, leia
> `references/config-completo.md`** — tem o schema exato, defaults e limites.

---

## Regra de ouro

**Nunca despeje todos os campos de uma vez.** Um agente bom nasce de uma conversa de
descoberta curta, depois config incremental. A maioria dos agentes ruins tem prompt
gigante e zero tools — o contrario do que converte. Prefira **structured prompt + tools
nativas** a um `system_prompt` cru e gigante.

> **Portabilidade / fonte de verdade:** o ideal e confirmar campos no monorepo
> (`packages/shared` + `packages/ai-agent-ui`). Quando ele NAO estiver disponivel (skill rodando
> sozinha, repo clonado), **NAO invente nem chute valores** — use os shapes ja fixados em
> `references/config-completo.md` secao 3.1 (formas de import) e 3.3 (exemplos known-good) como
> fonte de verdade, e avise o usuario que os limites numericos (ranges/defaults) refletem jun/2026
> e podem ter mudado por migration.

---

## Fluxo de criacao (siga nesta ordem)

### 1. Descoberta (pergunte antes de configurar)
Levante o minimo necessario. Se o usuario nao souber, sugira defaults sensatos.
- **Negocio:** empresa, segmento, o que vende, ticket medio.
- **Objetivo do agente:** qualificar lead? agendar? vender? suporte? recuperar?
- **Persona:** nome do agente, tom (ver presets de tom), o que ele NUNCA faz.
- **Canal e escopo:** global, por departamento ou por pipeline? E o agente primario (`is_primary`)?
- **Acoes que precisa tomar:** agendar reuniao, transferir pra humano, mover no funil, taggear, etc. → vira **tools**.
- **Base de conhecimento:** FAQ, tabela de precos, catalogo? → vira **structured_sources** (JSON) ou **RAG**.

### 2. Identidade e modelo
- Escolha o **modelo**: default `gpt-5-mini` (bom custo/qualidade). Use `gpt-5` so se a
  tarefa exigir raciocinio mais pesado; `gpt-4o-mini` pra fluxos simples/baratos.
- Defina `status: draft` ate testar. So `active` depois do teste.
- Defina `scope_type` e, se houver mais de um agente, planeje o roteamento (ver passo 8).

### 3. Prompt — SEMPRE via Structured Prompt (editor SDR)
Prefira `structured_prompt_config` ao `system_prompt` cru. Preencha:
- **Identidade:** agent_name, company, segment, channel, region, goal (sao interpolados nas regras).
- **Personalidade:** escolha um `tone.preset` (`direct_commercial`, `consultive_empathic`,
  `formal_institutional`, `casual_youth`) + traits opcionais.
- **Master prompt:** missao central em texto livre.
- **Regras comerciais:** fases do atendimento (titulo + descricao em bullets). Ex: Abertura →
  Qualificacao → Apresentacao → Fechamento → **Fora do escopo** (ver abaixo).
- **Acoes proibidas:** lista do que o agente NUNCA faz (prometer desconto, dar diagnostico, etc).

O sistema compila isso num `system_prompt` final com data atual, identidade, tom, objetivo,
fluxo comercial e proibicoes. Detalhes da compilacao em `references/config-completo.md`.

> **Como o editor recebe os dados:** o structured prompt NAO e um JSON unico. **Cadastro,
> Personalidade e Apresentacao sao campos de FORMULARIO (digitar)**; so **Roteiro, Regras Gerais
> e Templates** tem botao "Importar JSON" (shape proprio cada um) e **Fonte estruturada** entra
> por modal. Entregue **arquivos separados** por bloco (roteiro = array de fases; regras gerais =
> array de strings; templates = array) + os campos manuais em texto. **Nunca** um mega-JSON — nao
> importa. Shapes exatos em `references/config-completo.md` secao 3.1.

> **Fase padrao OBRIGATORIA — "Fora do escopo" → `stop_agent`:** todo roteiro termina com uma
> fase que, quando o lead foge do assunto comercial ou pede um humano, responde curto
> (`Certo, ja ja damos continuidade por aqui.`) e chama `stop_agent`. Exige `stop_agent` habilitada
> e `allow_human_handoff: true`. Texto-modelo pronto em `references/config-completo.md` secao 3.2.

### 4. Tools (o que faz o agente AGIR)
Um agente que so conversa nao converte. Anexe **tools nativas** (40+ presets prontos) pelos
casos de uso levantados na descoberta:
- Agendamento: `create_appointment`, `get_available_slots`, `reschedule_appointment`, `cancel_appointment`, `confirm_appointment` (requer `calendar_connection_id`).
- Funil/CRM: `move_pipeline_stage`, `add_tag`/`remove_tag`, `set_lead_custom_field`, `assign_product`/`assign_department`.
- Humano: `transfer_to_user`, `transfer_to_agent`, `stop_agent`, `close_conversation`, `trigger_notification`.
- Distribuicao: `round_robin_user`/`round_robin_agent`.
- Midia: `send_media`, `send_audio`.

Regras: nome `snake_case`, unico por agente; so habilite o que o agente realmente precisa
(menos tools = decisao melhor). Tools customizadas podem ser `n8n_webhook` ou `mcp`.

### 5. Conhecimento (knowledge / structured sources)
- **Dados estruturados pequenos** (precos, produtos, servicos): use `structured_sources` tipo
  `json` inline — injetado direto no prompt, sem custo de embedding.
- **Documentos grandes / FAQ extenso:** use **RAG** (`agent_knowledge_sources`), embeddings Voyage,
  top_k 3 por padrao.
- **Dados ao vivo de outro sistema:** `structured_sources` tipo `mcp` apontando pra um servidor MCP.

### 6. Humanizacao (faz parecer gente, nao robo)
Configure `humanization_config`:
- `split_enabled` + `split_threshold_chars` (~200): quebra respostas longas em varias mensagens.
- `split_delay_seconds` (~2): intervalo entre mensagens.
- `auto_pause_minutes` (~30): pausa a IA quando um humano responde manualmente.
- `pause_keywords`/`resume_keywords`: ex "PAUSAR"/"ATIVAR".
- `business_hours_enabled` + `after_hours_message`: so responde em horario comercial (TZ America/Sao_Paulo).

### 7. Debounce, followups e validacao
- **Debounce** (`debounce_window_ms`, default 10000, range 0–60000): agrega mensagens fragmentadas
  ("oi" + "tudo bem?") em UM run. Use 0 pra fluxos transacionais (OTP), maior pra chats tagarelas.
- **Followups** (`agent_followups`, max 5/agente): reengaja lead inativo apos N horas (range 1–720h).
  Tipo `text` (fixo) ou `ai_generated` (IA escreve baseado no historico). Respeita janela de envio.
- **Validacao de resposta** (`validation` config): `max_chars`, `one_question_only`,
  `forbidden_phrases`, `blocked_promises`, e `on_block` (rewrite/fallback/pause_ai/alert_only).
  Use pra impedir que o agente prometa o que nao pode.

### 8. Multi-agente / roteamento (se houver mais de 1 agente)
- Exatamente UM agente `is_primary: true` por org (roteador de entrada).
- Agentes secundarios recebem leads via `agent_entry_conditions` — 6 tipos: `tag_match`,
  `segment_match`, `message_contains`, `pipeline_stage_match`, `lead_status_match`, `channel_match`.
  Avaliadas por prioridade (primeiro match ganha). Intent router (gpt-4o-mini) e fallback opcional.
- **Transferencia agente→agente:** tool nativa `transfer_to_agent` (preserva historico/variaveis,
  entra pelo flow do alvo). Diferente de `transfer_to_user` (humano) e `stop_agent` (pausa).
- Use `pause_segment_ids` pra silenciar o agente em segmentos especificos.
- **Quando criar 2º agente:** so quando publico/objetivo divergem (ex: Vendas vs Suporte/pos-venda).
  NAO quebre a jornada de venda em varios agentes (derruba conversao). Fique em 2–5 agentes. Detalhes
  e mecanica completa em `references/config-completo.md` secao 10.

### 9. Guardrails e teste (sempre antes de ativar)
- `guardrails`: `max_iterations` (~5), `timeout_seconds` (~30), `cost_ceiling_tokens` (~20000),
  `allow_human_handoff`.
- **Teste no simulador** (`Tester`, dry-run) ANTES de `status: active`. Verifique: persona,
  uso correto das tools, respeito as proibicoes, tamanho das mensagens.

---

## Checklist de validacao (rode ANTES de entregar o JSON)

Antes de dar qualquer JSON pro usuario, confira item a item — e o que evita retrabalho:

- [ ] **Separei manual × JSON?** Cadastro/Personalidade/Apresentacao em TEXTO (digitar);
  Roteiro/Regras/Templates como arquivos JSON separados; Fonte por modal. **Nunca** um mega-JSON.
- [ ] **Roteiro:** e um `array`? cada fase tem `title`? `description` e **STRING** (bullets `\n`),
  NAO array? `profile_label` e uma FRASE em 1a pessoa, nao um rotulo?
- [ ] **Tem a fase "Fora do escopo" → `stop_agent`** como ultima fase? (secao 3.2)
- [ ] **Regras Gerais:** e um `array` de **strings** (nao objetos)?
- [ ] **Templates:** cada um tem `key` (so `[a-z0-9_]`, sem espaco), `name`, `message`? `mode`
  valido? Nao contei com `no_split` no import (e ignorado)?
- [ ] **Tools:** so as necessarias, `snake_case`, e estao no `enabled_tools` do flow? `stop_agent`
  presente p/ a fase de escopo? Agendamento exige `calendar_connection_id` configurado?
- [ ] **Validacao:** `blocked_promises` cobre o que o agente nao pode prometer (preco/desconto/prazo)?
- [ ] **Status `draft`** ate testar no Simulador? Lembrei que webhook/agendamento real so roda no
  WhatsApp, nao no Simulador?

Se algum item falhar, o import quebra ou o agente se comporta mal. Nao entregue antes de fechar todos.

## Anti-patterns (corrija quando vir)
- **Prompt monolitico de 2000 linhas sem tools** → migre pra structured prompt + tools nativas.
- **Agente sem `stop_agent`/handoff** → leads ficam presos com a IA; sempre dê saida pra humano.
- **Roteiro sem fase "Fora do escopo"** → IA tenta responder qualquer assunto/insiste quando o lead quer humano. Sempre feche o roteiro com a fase padrao que chama `stop_agent` (secao 3.2).
- **Entregar um mega-JSON do agente inteiro** → o editor importa por bloco (Roteiro/Regras/Templates separados); Cadastro/Personalidade/Apresentacao sao digitados. Entregue arquivos separados (secao 3.1).
- **Sem debounce** → IA responde 3x pra 3 mensagens fragmentadas. Ligue debounce.
- **Sem validacao** → IA promete desconto/prazo que nao existe. Use `blocked_promises`.
- **Dois agentes `is_primary`** → roteamento quebra. So um primario por org.
- **RAG pra tabela de precos de 10 linhas** → use structured_source JSON, nao embedding.
- **Ativar sem testar no simulador** → nunca.

## Diagnostico rapido ("agente nao responde")
1. `status` esta `active`? Feature flag `native_agent_enabled` ligada na org?
2. Conversa pausada? (`human_handoff_at`, `auto_pause_minutes`, `pause_keywords`, `pause_segment_ids`).
3. Fora do `business_hours`? Lead em `pause_segment_ids`?
4. Debounce ainda na janela (`next_flush_at` no futuro)? Cron de flush rodando?
5. Estourou `cost_ceiling_tokens` / `timeout_seconds`? Veja `agent_runs`/`agent_steps`.
6. Roteamento: nenhuma `entry_condition` bateu e nao ha primario? Veja `references/config-completo.md`.

---

**Sempre cheque `references/config-completo.md` para nomes de campos, defaults e limites exatos
antes de afirmar valores.** O contrato em `packages/shared/src/ai-agent/` muda por migration.

> ⚠️ **Antes de debugar "nao funciona", leia a secao 14 (Gotchas / licoes de campo) do
> `config-completo.md`.** As armadilhas mais comuns: editar JSON mas nao reimportar+salvar; o
> Simulador nao chamar webhook tool (so em msg real); criar tool mas ela nao estar no `enabled_tools`
> do flow; e o prompt mandar ler a fonte em vez de chamar a tool. Quando a UI de tools/canvas estiver
> capenga, o caminho confiavel e **fonte estruturada** (preco/regra no prompt), nao a tool de webhook.
