# Jordan — Funil (Kanban) + Tags

Organização de leads adicionada ao roteiro do Jordan. Usa só as **2 tools de 1-clique**
(`add_tag` = *Etiquetar lead*, `move_pipeline_stage` = *Mover no funil*), que funcionam
**sem** a flag `native_agent_tools_ui`.

## Como habilitar
No agente Jordan → aba **Regras** → card **"Ferramentas do agente"** → ligar:
- **Etiquetar lead** (`add_tag`)
- **Mover no funil** (`move_pipeline_stage`)

Depois **Salvar**. (Só referenciar no roteiro depois de ligadas — o roteiro já traz as chamadas.)

## Funil (criar estas etapas no CRM, com os nomes EXATOS)
1. **Novo Lead** — estado inicial (lead entra)
2. **Qualificando** — coletando idade/perfil
3. **Cotação Enviada** — valores apresentados
4. **Em Negociação** — dúvidas / objeção
5. **Documentação** — sinal de compra, docs pedidos
6. **Fechado / Perdido** — contratou (humano assume) ou saiu do escopo

> ⚠️ Os nomes usados em `move_pipeline_stage` precisam bater **exatamente** com os nomes
> das etapas no CRM. Se você renomear no CRM, ajuste no roteiro.

## Tags (criar no CRM)
| Grupo | Tags |
|---|---|
| Perfil | `pf` · `pj-mei` · `menor-de-idade` |
| Obstetrícia | `com-obstetricia` · `sem-obstetricia` |
| Interesse | `interesse-vital` · `interesse-superior` |
| Sinal | `lead-quente` |
| Objeção | `objecao-preco` · `objecao-vai-pensar` |
| Situação | `reativacao` · `pediu-presencial` · `fora-escopo` |

## Onde cada tool dispara (mapeado nas fases do roteiro)
| Fase | Tags | Move para |
|---|---|---|
| 2 · Idade | — | **Qualificando** |
| 3 · Menor | `menor-de-idade`, `pf` | — |
| 4 · PF ou PJ/MEI | `pf` **ou** `pj-mei` | — |
| 5 · Obstetrícia | `com-obstetricia` / `sem-obstetricia` | — |
| 6 · Cotação PF | `interesse-vital`/`superior` (se citar) | **Cotação Enviada** |
| 7 · Cotação PJ | `interesse-vital`/`superior` (se citar) | **Cotação Enviada** |
| 8 · Dúvidas/objeção | `objecao-preco` / `objecao-vai-pensar` | **Em Negociação** |
| 10 · Documentos | `lead-quente` | **Documentação** |
| 11 · Reativação | `reativacao` | — |
| 12 · Fora do escopo | `pediu-presencial` / `fora-escopo` | **Fechado / Perdido** |

> **Fechamento ganho:** quando o lead efetivamente contrata, quem move para uma etapa de
> "Ganho" é o humano que assume após a Fase 10 (o Jordan não confirma contratação sozinho).
> Se quiser uma etapa separada de **Ganho**, crie no CRM e mova manualmente no fechamento.

## Nota
Preço permanece via **fonte estruturada** (`precos.json` / `promocoes-junho-2026.json`) — o
Jordan lê a tabela, não usa a tool de cotação n8n. Nada mais no agente foi alterado além
das chamadas de tag/funil adicionadas às fases.
