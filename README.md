# Agentes de IA — CRM Persia

Repositório com a **configuração de dois agentes de IA (SDRs de WhatsApp)** do CRM Persia
(`Automações > Agentes`), mais a skill que os monta. Cada agente é um chatbot que atende
leads no WhatsApp, qualifica e conduz para a conversão.

## Agentes

### 🟦 [`assistente-persia/`](assistente-persia/) — Pérsia (PérsiaCRM)
SDR da própria **PérsiaCRM** (SaaS de IA + CRM no WhatsApp). Atende quem clica em "Falar com
especialista" no site, qualifica (vende pelo WhatsApp? tem time? qual a dor?), apresenta valor
sem fechar preço (sob consulta) e **agenda a demo** ou transfere pra um especialista.
- Roteiro de **7 fases** (incl. Tratamento de objeções e Fora do escopo → `stop_agent`).
- Separação correta **manual × Importar JSON** + checklist de ativação.
- Ver [`assistente-persia/README.md`](assistente-persia/README.md).

### 🟩 [`prompt_jordan/`](prompt_jordan/) — Jordan (Humana Saúde)
SDR de **planos de saúde** (Humana Saúde, Teresina-PI). Qualifica por idade/PF-PJ, apresenta
planos e faz **cotação via n8n**, com tabela de preços e promoções.
- `roteiro-de-abordagem (1).json`, `regras-gerais.json`, `templates-4c868e4e (2).json`
- `precos.json`, `promocoes-junho-2026.json` (fontes estruturadas)
- `cotacao-tool-schema.json` + `cotar-plano-n8n.json` (tool de cotação via webhook n8n)
- `suporte-fase2.md` (notas) · `COPY do Agente JORDAN.pdf` (cópia de referência)

## Skill incluída
[`.claude/skills/criador-de-agente-crm-persia/`](.claude/skills/criador-de-agente-crm-persia/) —
ao clonar este repo com o Claude Code, a skill que cria/configura agentes do CRM Persia fica
disponível automaticamente (vale pros dois agentes). Contém `SKILL.md` + `references/config-completo.md`
(formatos de import, exemplos known-good e checklist de validação).

## Como usar
O conteúdo **não é código** — é configuração pra importar no editor de agentes do CRM Persia.
Cada agente tem seus arquivos; o editor importa **por bloco** (Roteiro, Regras Gerais, Templates
e Fonte estruturada têm "Importar JSON"; Cadastro/Personalidade/Apresentação são digitados).
Detalhes e checklist em cada pasta.
