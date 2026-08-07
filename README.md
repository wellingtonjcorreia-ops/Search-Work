# 📋 Candidaturas — Wellington Correia

Painel pessoal para automatizar a busca e o envio de candidaturas a vagas de
**FP&A / Controladoria / Planejamento Financeiro**, combinando um agente de IA
(Claude), um banco de dados (Supabase) e um workflow local (n8n) que dispara
tudo a partir de comandos dados por chat.

🔗 Painel em produção: `candidaturas-wellington.vercel.app`

---

## Como o sistema funciona

```
Wellington (chat)  →  Claude (agente)  →  Supabase (banco)  ←→  Painel web (Vercel)
                                              ↑
                                    n8n (Mac, polling 1x/min)
                                              ↓
                                    Notificação push (ntfy)
```

1. **Busca de vagas** — Wellington pede no chat ("buscar"). O Claude varre o
   Indeed (via conector de vagas) e o LinkedIn (via Claude in Chrome),
   pontua cada vaga de 0 a 100 com base em aderência de função, senioridade,
   localização, porte da empresa e requisitos atendidos — com penalidade para
   vagas que exigem inglês avançado — e grava as aprovadas (nota ≥ 45) na
   tabela `vagas`.
2. **Aprovação** — Wellington revisa as vagas no painel web e marca as que
   quer enviar (`status = 'Aprovada para envio'`).
3. **Envio de candidaturas** — ao clicar em "Enviar" no painel (ou pedir
   "enviar" no chat), o sistema grava um comando na tabela `comandos`. Um
   workflow **n8n** rodando no Mac do Wellington monitora essa tabela a cada
   minuto e manda uma notificação push (via **ntfy**) avisando que há um
   pedido pendente. O Claude então processa as vagas aprovadas: em
   candidaturas simples (poucos campos, sem CAPTCHA, sem etapas extras) tenta
   preencher e enviar automaticamente pelo Chrome; qualquer coisa mais
   complexa vai para `'Enviar manualmente'`.
4. Cada execução (busca ou envio) fica registrada na tabela `execucoes`, e
   cada mudança de status de uma vaga fica registrada em `vaga_eventos`.

O n8n **não decide nada sozinho** — ele só observa a fila de comandos e
notifica. Quem executa a busca e o envio é o Claude, via chat.

---

## Stack técnica

- **Frontend**: página única (`index.html`), **HTML/CSS/JavaScript puro**,
  sem framework e sem build step — os módulos JS são importados diretamente
  via ES Modules (`<script type="module">`).
- **Banco/Auth**: [Supabase](https://supabase.com) (Postgres + Auth), acessado
  no browser através do pacote `@supabase/supabase-js` carregado via CDN
  (`esm.sh`).
- **Autenticação**: link mágico por e-mail (passwordless) — não existe senha
  para lembrar nem para vazar. Quem protege os dados é a política de **RLS**
  (Row Level Security) do Supabase, não o segredo da chave pública usada no
  frontend.
- **Hospedagem**: [Vercel](https://vercel.com) (deploy estático).
- **Automação local**: [n8n](https://n8n.io) rodando no Mac do Wellington,
  monitorando a tabela `comandos` a cada 1 minuto e notificando via
  [ntfy](https://ntfy.sh).
- **Agente**: Claude, orquestrando busca (Indeed + LinkedIn), pontuação de
  vagas e preenchimento de candidaturas simples via Claude in Chrome.

---

## Modelo de dados (Supabase — projeto `Search-Work-BR`)

### `vagas`
Cada linha é uma vaga encontrada e pontuada.

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | uuid | chave primária |
| `owner_id` | uuid | dono do registro |
| `cargo`, `empresa` | text | únicos em conjunto (evita duplicidade) |
| `localizacao`, `modalidade` | text | cidade/UF e Presencial/Híbrido/Remoto |
| `publicado_em` | date | data de publicação da vaga |
| `link` | text | URL da vaga |
| `pontuacao` | integer | nota de 0 a 100 dada pelo Claude |
| `palavras` | text | 3 palavras-chave da vaga |
| `briefing` | text | o que destacar no currículo para essa vaga |
| `status` | text | `Nova` → `Aprovada para envio` → `Enviada` / `Enviar manualmente` |
| `ja_candidatei` | boolean | | 
| `candidatura_em` | date | | 
| `motivo` | text | por que não deu para automatizar o envio |
| `fonte` | text | `Indeed` ou `LinkedIn` |
| `curriculo_path`, `carta_path` | text | anexos usados na candidatura |
| `criado_em`, `atualizado_em` | timestamptz | |

### `comandos`
Fila de pedidos que o n8n observa.

| Coluna | Tipo | Descrição |
|---|---|---|
| `acao` | text | `buscar_vagas` ou `processar_aprovadas` |
| `estado` | text | `pendente` → `executando` → `concluido` |
| `detalhe`, `resultado` | text | |
| `pedido_em`, `atualizado_em`, `concluido_em` | timestamptz | |

### `execucoes`
Histórico de cada rodada de busca ou envio.

| Coluna | Tipo | Descrição |
|---|---|---|
| `tipo` | text | `busca` ou `envio` |
| `origem` | text | de onde partiu o pedido (ex: `chat`) |
| `estado` | text | `rodando` / `concluido` |
| `fontes` | text | fontes cobertas na rodada (ex: `Indeed + LinkedIn`) |
| `encontradas`, `novas`, `enviadas`, `manuais` | integer | contadores da rodada |
| `resumo` | text | uma linha sobre o resultado |
| `iniciado_em`, `concluido_em` | timestamptz | |

### `vaga_eventos`
Trilha de auditoria de cada mudança de status de uma vaga.

| Coluna | Tipo | Descrição |
|---|---|---|
| `vaga_id` | uuid | referência a `vagas.id` |
| `de_status`, `para_status` | text | transição de status |
| `origem` | text | `app` (painel) ou `robo` (Claude) |
| `observacao` | text | |
| `criado_em` | timestamptz | |

---

## Critério de pontuação das vagas (0–100)

- **Até 40 pts** — aderência da função (FP&A/Controladoria puro pontua alto)
- **Até 20 pts** — senioridade (coordenação/gerência alto, júnior baixo)
- **Até 20 pts** — localização (Balneário Piçarras e região ou remoto pontuam alto)
- **Até 10 pts** — porte e estabilidade aparentes da empresa
- **Até 10 pts** — requisitos que o candidato atende de fato (ERP, BI, Excel)
- **Penalidade de −25 pts** se a vaga exigir inglês avançado/fluente

Vagas com nota abaixo de 45 são descartadas e não entram no painel.

---

## Rodando localmente

Como é uma página estática sem build, basta servir o arquivo:

```bash
npx serve .
# ou simplesmente abrir o index.html no navegador
```

O único pré-requisito é que a URL e a chave pública (anon key) do projeto
Supabase estejam configuradas no início do script do `index.html`. Ambas são
públicas por design — a segurança fica por conta das políticas de RLS
configuradas no banco, então nunca use a **service_role key** no frontend.

## Segurança

- Autenticação passwordless (magic link) via Supabase Auth.
- Toda leitura/escrita passa pela RLS do Supabase, filtrando por `owner_id`.
- O n8n só lê a tabela `comandos` e nunca executa ações — apenas notifica.
