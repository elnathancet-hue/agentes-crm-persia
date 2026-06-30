# Config técnica + ativação

Tudo que **não** faz parte do Structured Prompt é configurado em outras abas/toggles do
editor (não é JSON de importação). Valores recomendados abaixo.

## Configuração geral do agente
| Item | Valor | Onde |
|---|---|---|
| Status | `draft` (mudar pra `active` só após testar) | topo do editor |
| Modelo | `gpt-5-mini` | aba de configuração |
| Escopo | `global` | aba de configuração |
| Agente primário | **Sim** (`is_primary`) — é a porta de entrada do WhatsApp | aba de configuração |
| Debounce | `12000` ms | aba Regras |

## Ferramentas — como habilitar de verdade (UI real)

As tools se ligam pelo card **"Ferramentas do agente"** na aba **Regras** ("Como o agente decide").
**Atenção:** só 5 tools têm toggle 1-clique; o resto exige a flag `native_agent_tools_ui` (default
**OFF**) na aba **Ferramentas**.

**Nível 1 — toggles 1-clique (qualquer org, sem flag):**
- **Agendar reunião** = `offer_appointment_slots` ✅ (principal de agendamento)
- **Etiquetar lead** = `add_tag`
- **Mover no funil** = `move_pipeline_stage`
- **Notificar equipe** = `trigger_notification`
- (Enviar mídia = `send_media` — não usado pela Pérsia)

**Nível 2 — exigem a flag `native_agent_tools_ui` ON (aba Ferramentas, admin/DB):**
- **Handoff:** `transfer_to_user`, `stop_agent` ← a fase "Fora do escopo" depende do `stop_agent`
- **Agenda de apoio:** `reschedule_appointment`, `cancel_appointment`, `confirm_appointment`,
  `get_available_slots`, `list_lead_appointments`
- **Distribuição:** `round_robin_user` · **Lead:** `set_lead_custom_field`

> ⚠️ **Sem a flag, a Pérsia faz o essencial** (qualificar, **agendar reunião**, etiquetar, mover no
> funil, notificar), mas **não** consegue transferir pra humano, parar (`stop_agent`) nem
> remarcar/cancelar. Pra o agente ficar completo (handoff + remarcar), peça ao admin pra ligar
> `native_agent_tools_ui`. Enquanto não ligar, ajuste o roteiro pra não depender dessas tools.

> ⚠️ **Mencionar a tool com `@`/crase no roteiro NÃO a habilita.** O dropdown do `@` só lista o
> que já está ligado. **Ordem certa:** 1) ligar a tool no card; 2) salvar; 3) só então referenciar
> no roteiro. Sintoma de tool não ligada: o agente diz "vou verificar" e não age.

> **Agendamento = agenda NATIVA (não Google Calendar).** `offer_appointment_slots` manda os
> horários como **menu interativo**; o lead **toca** e agenda sozinho (booking determinístico, não
> reconfirme). Multi-especialista → menu de profissional primeiro. **Pré-req na org:** serviço
> `reuniao` em `agenda_services` + `availability_rules` (+ `user_services` se multi). **Não** usa
> `calendar_connection_id` (isso é só do `schedule_event`/Google).

## Humanização
- `split_enabled`: ligado · `split_threshold_chars`: 220 · `split_delay_seconds`: 2
- `auto_pause_minutes`: 30 (pausa a IA quando um humano responde)
- `pause_keywords`: PAUSAR, HUMANO, STOP IA · `resume_keywords`: ATIVAR, IA ON, VOLTAR IA
- `business_hours`: desligado (SDR responde 24/7; ligar só se quiser horário comercial)

## Validação de resposta
- `enabled`: ligado · `max_chars`: 600 · `one_question_only`: ligado
- `blocked_promises`: preço, preços, valor do plano, mensalidade, custa, R$, desconto, teste grátis, grátis, garanto, prazo de implementação
- `on_block`: `rewrite`
- `fallback_message`: "Sobre valores eu prefiro não chutar nada. Os planos são personalizados e sob consulta. Que tal eu já agendar uma conversa rápida com um especialista pra te passar tudo certinho?"

## Followups (máx. 5; reengaja lead inativo)
1. **1h** — `ai_generated` — janela 08:00–20:00 — "Reengaje com leveza retomando onde parou. Uma pergunta para reabrir o diálogo. Não pareça cobrança."
2. **24h** — `ai_generated` — janela 09:00–19:00 — "Lembre o valor principal (organizar o WhatsApp, não perder lead) e convide de novo para a demo."
3. **72h** — `text` — janela 09:00–18:00 — "Oi! Por aqui é a Pérsia 🙂 Vou deixar a porta aberta: quando quiser ver a PérsiaCRM organizando seu atendimento no WhatsApp, é só me chamar que eu agendo uma conversa rápida com um especialista. Combinado?"

## Guardrails
`max_iterations`: 5 · `timeout_seconds`: 30 · `cost_ceiling_tokens`: 20000 · `allow_human_handoff`: ligado

## Pré-requisitos do ambiente (no seu CRM)
- **Agenda nativa:** serviço `reuniao` cadastrado em `agenda_services` + `availability_rules`
  (horários) definidos. Se houver mais de um especialista, adicionar cada um em `user_services`
  pro serviço `reuniao` (o menu de profissional aparece sozinho). **Sem** Google Calendar.
- `new_lead_stage_id` → stage do funil pra lead novo
- `on_appointment_created_stage_id` → stage quando a demo é agendada

---

# Checklist de ativação

## 1. Structured Prompt
- [ ] **Cadastro / Personalidade / Apresentação:** digitar à mão (ver [`01-campos-manuais.md`](01-campos-manuais.md)).
- [ ] **Roteiro de Abordagem:** botão *Importar JSON* → colar [`02-roteiro.json`](02-roteiro.json) → modo **Substituir** → Importar fases.
- [ ] **Regras Gerais:** botão *Importar JSON* → colar [`03-regras-gerais.json`](03-regras-gerais.json) → Importar regras.
- [ ] **Salvar o agente.** ⚠️ Mudar o JSON no PC não muda no agente; só vale após *Importar + Salvar*.

## 2. Templates e Fonte
- [ ] **Templates:** botão *Importar JSON* → colar [`04-templates.json`](04-templates.json). (⚠️ `no_split` não vem na importação; ligue o toggle na mão se precisar — relevante pro `planos_resumo`.)
- [ ] **Fonte estruturada:** Adicionar fonte → tipo JSON → colar o `data` de [`05-fonte-estruturada.json`](05-fonte-estruturada.json).

## 3. Config + tools
- [ ] Modelo, escopo, `is_primary`, debounce, humanização, validação, followups, guardrails (valores acima).
- [ ] **Tools nível 1** (card *Ferramentas do agente*, aba Regras): ligar **Agendar reunião**, **Etiquetar lead**, **Mover no funil**, **Notificar equipe** → **Salvar**.
- [ ] **Tools nível 2** (handoff, remarcar/cancelar etc.): exigem a flag `native_agent_tools_ui` ON (admin/DB) → aba **Ferramentas**. Sem a flag, a Pérsia não transfere nem para (`stop_agent`).
- [ ] Só referenciar tool com `@` no roteiro **depois** de ligada (o `@` só lista o que já está ligado).
- [ ] **Agenda nativa:** serviço `reuniao` em `agenda_services` + `availability_rules` (e `user_services` se multi-especialista) + os 2 stages do funil.

## 4. Testar (Simulador)
- [ ] Diálogo: abertura → qualificação (3 critérios) → valor → tentativa de agendamento → fora do escopo (peça "quero falar com uma pessoa" e confirme o `stop_agent`).
- [ ] Persona (consultiva, uma pergunta por vez, sem textão, sem travessão).
- [ ] Peça o preço de propósito → deve recusar valor e convidar pra agendar (a validação reescreve se escapar).
- [ ] ⚠️ O Simulador **não** envia o menu interativo real — teste o `offer_appointment_slots` (menu + toque que agenda) pelo **WhatsApp real**.

## 5. Ativar
- [ ] `status`: `draft` → `active`. Conferir a feature flag `native_agent_enabled` na org.

## Se "não responder"
1. `status = active` e `native_agent_enabled` ligada?
2. Conversa pausada? (`auto_pause_minutes`, `pause_keywords`, operador respondeu, `pause_segment_ids`).
3. Debounce ainda na janela (12s)? Cron de flush rodando?
4. Estourou `cost_ceiling_tokens` (20k) ou `timeout_seconds` (30s)? Veja `agent_runs` / `agent_steps`.
5. É o primário e está pegando o lead novo?
