# Assistente Pérsia — SDR de WhatsApp (PérsiaCRM)

Configuração **do zero** de um Agente de IA do CRM Persia (`Automações > Agentes`)
que atende os leads que clicam em **"Falar com especialista"** no site
[persiaapp.com/crm](https://www.persiaapp.com/crm), qualifica e agenda a demo.

> Isto **não é código** — é a *configuração* de um agente. Parte é digitada no editor,
> parte é importada por JSON. Veja o mapa abaixo.

---

## O que esse agente faz
1. **Atende** o lead no WhatsApp assim que ele clica no botão do site.
2. **Qualifica** — vende pelo WhatsApp? tem time/volume? qual a dor?
3. **Apresenta valor** da PérsiaCRM, sem fechar preço (é "sob consulta").
4. **Agenda** a demo com especialista (tenta no Google Calendar).
5. **Transfere/para** quando o lead quer humano ou foge do escopo (`stop_agent`).
6. **Organiza no funil** (stage, tags, campos) e **reengaja** lead frio (followups).

## Como o editor funciona (importante)
No editor SDR, **só 4 blocos têm "Importar JSON"**. O resto é formulário:

| Bloco | Entra como | Arquivo |
|---|---|---|
| Cadastro de Informações | digitar | [`01-campos-manuais.md`](01-campos-manuais.md) |
| Personalidade | digitar | [`01-campos-manuais.md`](01-campos-manuais.md) |
| Apresentação do Agente | digitar | [`01-campos-manuais.md`](01-campos-manuais.md) |
| **Roteiro de Abordagem** (fases) | **Importar JSON** (array de fases) | [`02-roteiro.json`](02-roteiro.json) |
| **Regras Gerais** | **Importar JSON** (array de strings) | [`03-regras-gerais.json`](03-regras-gerais.json) |
| **Templates** | **Importar JSON** (array de templates) | [`04-templates.json`](04-templates.json) |
| **Fonte estruturada** | modal "Adicionar fonte" (colar JSON) | [`05-fonte-estruturada.json`](05-fonte-estruturada.json) |
| Modelo, tools, humanização, validação, followups, guardrails | abas/toggles | [`06-config-e-ativacao.md`](06-config-e-ativacao.md) |

Passo a passo completo de import → teste → ativação em [`06-config-e-ativacao.md`](06-config-e-ativacao.md).

## Decisões de design
- **Persona Pérsia**, tom Consultivo & Empático, uma pergunta por mensagem, sem textão, **sem travessão** (estilo WhatsApp).
- **Modelo** `gpt-5-mini`, agente **primário** (porta de entrada do WhatsApp).
- **Preço blindado:** "sob consulta" é Regra Geral + `blocked_promises` na validação (reescreve se a IA escapar).
- **Agendamento híbrido:** tenta marcar no Calendar; se travar ou o lead pedir gente, transfere.
- **Roteiro com 7 fases** (Abertura → Qualificação → Valor → Agendar → **Objeções** → Pós → **Fora do escopo → `stop_agent`**).
- **Conhecimento** via fonte estruturada (planos sem preço) — sem RAG, que seria exagero aqui.

## Skill incluída no repo
A skill `criador-de-agente-crm-persia` está em [`.claude/skills/`](.claude/skills/), então ao
**clonar este repo** (com o Claude Code instalado) ela fica disponível automaticamente neste
projeto — útil pra ajustar/recriar o agente em outra máquina. Conteúdo: `SKILL.md` +
`references/config-completo.md` (texto autocontido; os caminhos do monorepo citados são só
referência, não precisam estar presentes).

## Formato confirmado no código
`packages/ai-agent-ui/src/components/StructuredPromptEditor.tsx` e `RulesTab.tsx`:
- **Roteiro:** array `[{ "title"*, "profile_label", "description" }]` (`description` é **string**, bullets em `\n`).
- **Regras Gerais:** array de **strings** `["...", "..."]`.
- **Templates:** array `[{ "key"*, "name"*, "message"*, "usage", "mode" }]` (`no_split` é ignorado na importação).
- **Identidade/Personalidade/Apresentação:** form fields (sem JSON).
