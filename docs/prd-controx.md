# PRD — Controx CRM

| Campo | Valor |
|---|---|
| Produto | Controx — CRM multiempresa para leads de WhatsApp com agente de IA |
| Domínio | `controx.com.br` |
| Versão do documento | 1.0 |
| Data | 04/11/2026 |
| Base | Telas de produção (Métricas, Pipeline, Mensagens, Agenda, Integrações, Empresas) |

---

## 1. Visão geral

O Controx é um CRM web, multiempresa (multi-tenant), que recebe leads vindos do WhatsApp (via fluxos do N8N e um agente de IA/SDR), organiza esses leads num pipeline kanban, mostra as conversas em tempo real, sincroniza reuniões com o Google Calendar e devolve eventos de conversão ao Meta Ads (Conversions API) para medir qual campanha e qual anúncio geram vendas.

Cada empresa cliente tem login, dados, webhook e integrações próprios. O administrador da plataforma enxerga todas as empresas, cria logins e pode abrir o dashboard de qualquer cliente.

### 1.1 Problema

- Leads de anúncios "clique para WhatsApp" chegam soltos, sem registro de origem (campanha/anúncio).
- O atendimento é feito por um agente de IA no N8N, mas a equipe não tem onde acompanhar as conversas nem como assumir a conversa quando precisa.
- Reuniões agendadas pelo agente não aparecem num lugar único junto com o funil.
- O Meta Ads não recebe de volta os eventos de agendamento e venda, então a otimização de campanha fica limitada ao evento de lead.

### 1.2 Objetivos

1. Centralizar todo lead de WhatsApp de uma empresa num pipeline com etapas fixas.
2. Exibir as conversas (cliente e agente) e permitir resposta manual pelo CRM, pausando o agente.
3. Criar e exibir reuniões no Google Calendar conectado da empresa.
4. Medir o funil (leads → qualificados → agendados → compareceram → fechamentos) e o resultado por campanha do Meta Ads.
5. Enviar eventos `Lead`, `Schedule`, `Contact` e `Purchase` ao Meta pela Conversions API.
6. Garantir isolamento total de dados entre empresas.

### 1.3 Fora do escopo (v1)

- Disparo de mensagens em massa / campanhas de WhatsApp.
- Faturamento/cobrança das empresas clientes.
- App mobile nativo (o sistema é web responsivo; as telas foram validadas em iPad).
- As métricas listadas em "Métricas a definir" (seção 6.1.4) — aparecem na interface como placeholders.

---

## 2. Personas e papéis

| Papel | Quem é | O que pode fazer |
|---|---|---|
| **Admin da plataforma** | Dono/operador do Controx | Ver e criar empresas, criar/remover logins, gerar nova senha, abrir o dashboard de qualquer empresa, configurar o cliente OAuth do Google (uma vez para todas as empresas). Único que vê o menu **Empresas**. |
| **admin** (da empresa) | Dono/gestor da empresa cliente | Acesso completo aos dados da própria empresa: Métricas, Pipeline, Mensagens, Agenda e Integrações. |
| **vendedor** | Equipe comercial da empresa | Trabalha leads e conversas da própria empresa (Pipeline, Mensagens, Agenda, Métricas). Não altera integrações. |

Papéis selecionáveis ao criar login: `admin` e `vendedor`.

---

## 3. Arquitetura

```
 WhatsApp (Cloud API / Evolution)
        │
        ▼
 N8N (agente SDR de IA, qualificação, agendamento)
        │  POST webhook de leads  (slug + token da empresa)
        ▼
 Supabase Edge Function  ──►  Postgres (RLS por empresa)  ──►  Frontend Controx (controx.com.br)
        │                              ▲                               │
        │                              │ Realtime                      │ resposta manual
        ▼                              │                               ▼
 Meta Conversions API          Google Calendar API           URL de envio (fluxo N8N que
 (Lead/Schedule/Contact/        (eventos do CRM +             envia a mensagem no WhatsApp)
  Purchase)                      compromissos existentes)
```

- **Backend:** Supabase — Postgres, Auth (e-mail/senha + provedor Google OAuth), Row Level Security, Realtime, Edge Functions.
- **Entrada de dados:** webhook HTTP por empresa (`/functions/v1/...`), chamado pelo N8N.
- **Saída de mensagens:** o CRM faz POST na "URL de envio" configurada pela empresa (um webhook do N8N que entrega a mensagem no WhatsApp).
- **Agenda:** Google Calendar API usando o token OAuth da conta conectada pela empresa.
- **Meta:** Conversions API com o Pixel/Dataset ID da empresa.

---

## 4. Princípios de isolamento (texto exibido no sistema)

Bloco "**COMO O ISOLAMENTO FUNCIONA**" (tela Empresas), texto literal:

- Todos os dados ficam no Supabase com políticas de acesso por empresa (RLS): um usuário só enxerga a empresa à qual pertence.
- O webhook de leads leva o slug e um token por empresa: o lead cai no CRM certo e um token vazado só afeta aquele cliente.
- O admin da plataforma enxerga todas as empresas, cria logins e pode abrir o dashboard de qualquer cliente.
- Integrações (Google, Meta) são configuradas por empresa, com as credenciais de cada cliente.

---

## 5. Design system

### 5.1 Cores

| Token | Uso | Valor aproximado |
|---|---|---|
| `--bg` | Fundo da área de conteúdo | `#F3F3F3` |
| `--surface` | Cards | `#FFFFFF` |
| `--sidebar` | Menu lateral, cards KPI escuros, avatares, badges de contagem | `#0E0E0E` |
| `--accent` | Amarelo da marca: item ativo, botões primários, barras do funil, números nos cards escuros, borda do dia atual | `#FFD60A` |
| `--accent-soft` | Botão primário desabilitado | `#FFE88A` |
| `--text` | Texto principal | `#111111` |
| `--muted` | Subtítulos, rótulos, datas | `#6B6B6B` |
| `--border` | Bordas de inputs e cards de lead | `#E2E2E2` |
| `--success` / `--success-bg` | Badges "Ativo", "Conectado", "Resposta pelo CRM ativa", pill de e-mail Google | `#1E8E3E` / `#E6F4EA` |
| `--danger` / `--danger-bg` | Badges "Desconectado", botões "Desconectar"/"Remover" | `#D93025` / `#FDECEA` |

### 5.2 Tipografia

- Sans-serif do sistema (SF Pro / Inter). Títulos de página ~32px, peso 700.
- Rótulos de seção em CAIXA ALTA, espaçamento entre letras ampliado, cinza, peso 600 (ex.: "FUNIL", "MÉTRICAS A DEFINIR").
- Código e payloads em fonte monoespaçada, com fundo cinza claro e cantos arredondados (ex.: slug `mirian`, `Schedule`).

### 5.3 Componentes

- **Card:** fundo branco, raio ~16px, sem borda, sombra muito leve, padding ~32px.
- **Card KPI escuro:** fundo preto, rótulo cinza claro em caixa alta, número grande amarelo.
- **Card KPI claro:** fundo branco, número grande preto.
- **Botão primário:** fundo amarelo, texto preto em negrito, raio ~10px. Ex.: "+ Nova empresa", "+ Criar login", "Salvar URL de envio", "Salvar e conectar". Desabilitado = amarelo claro com texto esmaecido.
- **Botão secundário:** fundo branco, borda preta 2px, texto preto em negrito. Ex.: "Copiar", "Copiar URL", "Regenerar token", "Desativar", "Nova senha", "‹ Semana anterior", "Hoje", "Próxima ›".
- **Botão destrutivo:** fundo branco, borda vermelha 2px, texto vermelho. Ex.: "Desconectar", "Remover".
- **Badge/pill de status:** formato pílula, texto em negrito. Verde (Ativo, Conectado), vermelho (Desconectado, desconectado), amarelo (dashboard aberto, a definir).
- **Badge de contagem:** círculo preto com número amarelo (colunas do pipeline).
- **Estado vazio:** caixa com borda tracejada cinza e texto centralizado (ex.: "Vazio", "Nenhuma reunião futura.", "Nenhum evento ainda.").
- **Inputs:** fundo branco, borda cinza 1px, raio ~10px, placeholder cinza.
- **Avatar:** círculo preto com a inicial do nome em amarelo, negrito.
- **Collapsible:** "▶ Título" em negrito; expande o conteúdo de ajuda.

### 5.4 Layout

- **Sidebar fixa à esquerda** (~400px em desktop/tablet), fundo preto:
  - Topo: logo (quadrado amarelo arredondado com "C" preto) + "Controx" em branco, negrito.
  - Menu, nesta ordem, com ícone + rótulo:
    1. ◔ Métricas
    2. ▦ Pipeline
    3. ✉ Mensagens
    4. ▤ Agenda
    5. ⇄ Integrações
    6. ⌂ Empresas (somente admin da plataforma)
  - Item ativo: fundo amarelo, raio ~10px, texto preto negrito. Inativo: texto branco.
  - Rodapé (separado por linha fina): nome da empresa em uso (branco, negrito), "Nome · e-mail" do usuário (cinza, e-mail sublinhado) e botão "**Sair**" (contorno amarelo, texto amarelo, largura total).
- **Área de conteúdo:** título da página + subtítulo cinza; ações principais alinhadas à direita do título.

---

## 6. Telas e requisitos funcionais

### 6.1 Métricas

**Título:** "Métricas"
**Subtítulo:** "Visão geral do funil e dos resultados por campanha do Meta Ads."

#### 6.1.1 Cards KPI (linha com 4 cards)

| Card | Estilo | Valor | Linha de apoio |
|---|---|---|---|
| AGENDADOS | escuro | nº de leads que chegaram a "Reunião agendada" | "X% dos leads" |
| ATENDERAM (COMPARECERAM) | escuro | nº de reuniões com comparecimento | "X% dos agendados · N faltaram" |
| FECHAMENTOS | claro | nº de fechamentos ganhos | "N perdidos · X% conversão" |
| RECEITA | claro | soma do valor dos fechamentos ganhos, formato `R$ 0` | "ticket médio R$ X" |

Regras:
- Percentuais arredondados para inteiro; divisão por zero exibe `0%`.
- Conversão = ganhos ÷ leads recebidos.
- Ticket médio = receita ÷ ganhos (0 se não houver ganhos).

#### 6.1.2 Card "FUNIL"

Cinco linhas, cada uma com rótulo à esquerda, "**N** (X%)" à direita e uma barra de progresso amarela (trilho cinza claro) proporcional ao total de leads:

1. Leads recebidos — sempre 100%
2. Qualificados
3. Reuniões agendadas
4. Compareceram
5. Fechamentos (ganho)

Rodapé cinza: "Não qualificados: N · Ligações atendidas: N".

Regra de contagem: o funil é **cumulativo por etapa alcançada** — um lead que está em Follow-up conta como Qualificado (exemplo real: 5 leads, 1 em Follow-up → Qualificados 1 (20%)). Leads marcados como "não" na qualificação entram em "Não qualificados".

#### 6.1.3 Card "TRACKING META ADS — QUAL ANÚNCIO/CAMPANHA GERA VENDAS"

Tabela com colunas: **CAMPANHA**, **LEADS**, **AGENDADOS**, **FECHAMENTOS**, **RECEITA**, **CONVERSÃO** (barra + %), **ANÚNCIO QUE MAIS VENDEU**.

- Uma linha por valor de `campanha` dos leads; linha com campanha vazia agrupa os leads sem origem.
- "Anúncio que mais vendeu" = anúncio com mais fechamentos ganhos na campanha; "—" se não houver.
- Ordenação: por nº de leads, decrescente.
- Rodapé: "Campanha e anúncio chegam junto com o lead (parâmetros do clique no WhatsApp). Os eventos `Schedule` e `Purchase` voltam ao Meta pela Conversions API."

#### 6.1.4 Card "MÉTRICAS A DEFINIR"

Lista de itens com pill amarela "a definir" (placeholders do roadmap, sem cálculo na v1):

1. Tempo médio entre o lead chegar e a primeira resposta
2. Tempo médio do lead até o fechamento (ciclo de venda)
3. Taxa de no-show por dia da semana / horário
4. Custo por lead e custo por fechamento (precisa do gasto do Meta)
5. Motivos de perda mais frequentes
6. Follow-ups por lead até fechar

Layout: cards KPI no topo; abaixo, FUNIL (esquerda) e MÉTRICAS A DEFINIR (direita) lado a lado; abaixo, TRACKING META ADS em largura total.

### 6.2 Pipeline

**Título:** "Pipeline"
**Subtítulo:** "{N} leads · {Nome da empresa} · entrada de leads {ativo|inativo}"
**Ação à direita:** campo "Buscar lead..." (filtra por nome, telefone ou campanha).

Kanban com rolagem horizontal e 6 colunas fixas:

| # | Coluna | Descrição (subtítulo) |
|---|---|---|
| 1 | Novo lead | Chegou pelo WhatsApp |
| 2 | Qualificação | Sim ou não |
| 3 | Reunião agendada | Google Calendar |
| 4 | Ligação | Card de ligação |
| 5 | Follow-up | Card de follow-up |
| 6 | Fechamento | Ganho ou perdido |

Cabeçalho da coluna: título em negrito, subtítulo cinza, badge de contagem (círculo preto, número amarelo) e linha amarela inferior.

**Card de lead:**
- Nome em negrito (se não houver nome, exibe o telefone).
- Linha: telefone (sublinhado, link `wa.me`/abre a conversa) · tempo relativo desde a chegada ("há 19 h", "há 2 d", "há 1 d").
- Pill cinza com o nome da campanha (ex.: "Teste chat n8n", "Teste"); pill vazia quando sem campanha.
- Card com borda cinza, raio ~12px.

**Comportamento:**
- Arrastar e soltar entre colunas altera a etapa do lead.
- Ao mover para "Reunião agendada": cria/vincula evento no Google Calendar e dispara `Schedule` ao Meta (se habilitado).
- Ao mover para "Ligação": dispara `Contact` (se habilitado).
- Ao mover para "Fechamento": pedir resultado (ganho/perdido), valor (para ganho) e motivo (para perdido); ganho dispara `Purchase` com o valor.
- Coluna vazia exibe estado vazio tracejado "Vazio".
- Novos leads entram em "Novo lead" em tempo real (Supabase Realtime).
- Clique no card abre o detalhe do lead (dados, campanha/anúncio, histórico de etapas e atalho para a conversa).

### 6.3 Mensagens

**Título:** "Mensagens"
**Subtítulo:** "Conversas do WhatsApp da empresa. Responda por aqui ou deixe com o agente."

Layout em dois painéis dentro de um card:

**Painel esquerdo (lista):**
- Campo "Buscar conversa..." no topo.
- Itens ordenados pela última mensagem (mais recente primeiro): avatar (inicial), nome do contato (ou telefone), horário à direita (`HH:MM` se hoje, `DD/MM` se anterior) e prévia da última mensagem em uma linha com reticências.
- Prévia com ícone ↩ quando a última mensagem foi enviada pelo agente/empresa.
- Mídias aparecem como marcadores: "[áudio]", "[Mensagem]" (tipo não textual/desconhecido).

**Painel direito (conversa):**
- Sem seleção: ícone de balão, "**Selecione uma conversa**" e "As mensagens do cliente e as respostas do agente aparecem aqui em tempo real. Você pode assumir a conversa e responder."
- Com seleção: histórico em balões (cliente à esquerda, agente/empresa à direita), atualizado em tempo real; campo de resposta e botão enviar.
- **Assumir conversa:** ao responder pelo CRM, a conversa é marcada como "humano no controle" e o agente é pausado para aquele contato (o fluxo do N8N consulta essa flag antes de responder). Deve haver ação para devolver a conversa ao agente.
- O envio faz POST na "URL de envio" da empresa (seção 6.5.1). Se a URL não estiver configurada, o campo de resposta fica desabilitado com aviso.

### 6.4 Agenda

**Título:** "Agenda"
**Subtítulo:** "Sincronizada com Google Calendar · {e-mail da conta} · calendário "{nome do calendário}"" (e-mails sublinhados). Sem Google conectado: convite para conectar em Integrações.
**Ações à direita:** "‹ Semana anterior", "Hoje", "Próxima ›".

**Grade semanal:** 7 cards (Dom, Seg, Ter, Qua, Qui, Sex, Sáb), cada um com o dia abreviado em negrito à esquerda e a data `d/m` à direita. O dia atual tem borda amarela de 3px.

**Eventos no dia:**
- Horário em negrito (`08:00`) + título (ex.: "Gogh | Conversa de 15 min - Lucas Eduardo").
- Dois estilos (conforme legenda):
  - **Reunião do CRM:** bloco preto, título amarelo.
  - **Compromisso do Google:** bloco cinza claro com barra escura à esquerda.

**Card "PRÓXIMAS REUNIÕES":** lista as próximas reuniões do CRM (a partir de agora). Vazio: "Nenhuma reunião futura."

**Card "LEGENDA":**
- "Reunião do CRM" — "Criada pelo fluxo de qualificação, gravada no Google Calendar"
- "Compromisso do Google" — "Já existia na agenda conectada; bloqueia o horário"
- Texto: "Ao agendar, o CRM cria o evento no calendário conectado da empresa. Clique num compromisso do Google para abri-lo."

**Regras:**
- Eventos lidos do calendário conectado para a semana visível.
- Clique em compromisso do Google abre o evento no Google Calendar (`htmlLink`).
- Clique em reunião do CRM abre o lead correspondente.
- Compromissos existentes bloqueiam o horário para o agendamento automático.

### 6.5 Integrações

**Título:** "Integrações"
**Subtítulo:** "Configuração exclusiva da empresa **{Nome da empresa}**."

Três cards lado a lado e, abaixo, a tabela de eventos do Meta.

#### 6.5.1 Card "ENTRADA DE LEADS DO WHATSAPP"

- Badge de status: "Ativo" (verde) / "Inativo".
- Rótulo "Webhook exclusivo desta empresa" + input somente leitura com a URL completa (contém slug e token).
- Botões: "Copiar URL", "Regenerar token" (pede confirmação; invalida o token anterior), "Desativar" / "Ativar".
- "Último lead recebido: DD/MM/AAAA, HH:MM:SS".
- Documentação inline:
  - "Novo lead — POST JSON: `{ nome, telefone, campanha, anuncio, mensagem }` ou o payload bruto da WhatsApp Cloud API."
  - "Mensagens da conversa — POST JSON: `{ tipo: "mensagem", telefone, texto, autor: "cliente" | "agente" }`. Elas aparecem em Mensagens."
- Rótulo "URL de envio (fluxo do N8N que manda a mensagem no WhatsApp)" + input editável + botão "Salvar URL de envio" (habilitado apenas quando o valor mudou).
- Badge "Resposta pelo CRM ativa" (verde) quando a URL de envio está salva.
- Collapsible "▶ Como configurar no N8N (responder e pausar o agente)": passo a passo para (1) enviar leads e mensagens ao webhook, (2) expor um webhook de envio que recebe `{ telefone, texto }` e entrega no WhatsApp, (3) checar a flag de pausa antes do agente responder.

#### 6.5.2 Card "GOOGLE CALENDAR / GMAIL"

- Badge "Conectado" (verde) / "Desconectado" (vermelho).
- Conectado: "Conta: {e-mail}" e "Calendário: {calendário}" (sublinhados).
- Texto: "Reuniões agendadas no CRM viram eventos neste calendário, e os compromissos dele aparecem na Agenda."
- Botão "Desconectar" (destrutivo) / "Conectar Google" quando desconectado.
- Collapsible "▶ Passo a passo para conectar o Google Agenda".
- Caixa tracejada (nota técnica): "**Só para o admin da plataforma** (uma vez, vale para todas as empresas): criar o cliente OAuth no Google Cloud com a Calendar API ativa, origem `https://controx.com.br`, URI de redirecionamento `https://<projeto>.supabase.co/auth/v1/callback`, e ativar o provedor Google no Supabase Auth com esse ID e chave. Enquanto isso não for feito, o botão acima falha para todo mundo."
- Escopos OAuth: `calendar.events`, `calendar.readonly`, `userinfo.email`; acesso offline (refresh token) guardado por empresa.

#### 6.5.3 Card "META ADS · CONVERSIONS API"

- Badge "Conectado" / "Desconectado".
- Rótulo "Pixel / Dataset ID" + input (placeholder "Ex.: 1029384756").
- Botão "Salvar e conectar" (habilitado quando há ID digitado).
- Lista de eventos, cada um em um card com checkbox à direita:
  - `Lead` — Lead chega pelo WhatsApp
  - `Schedule` — Reunião agendada
  - `Contact` — Ligação registrada
  - `Purchase` — Fechamento ganho

#### 6.5.4 Card "ÚLTIMOS EVENTOS PARA O META"

Tabela com colunas **QUANDO**, **LEAD**, **EVENTO**, **STATUS** (enviado / erro, com mensagem de erro no hover). Vazio: "Nenhum evento ainda." Exibe os 20 mais recentes.

### 6.6 Empresas (somente admin da plataforma)

**Título:** "Administração · empresas"
**Subtítulo:** "{N} dashboard(s) criado(s) · {N} leads no total. Cada empresa tem login, dados e webhook próprios."
**Ação:** "+ Nova empresa" (pede nome e slug; slug único, minúsculo, sem espaços; gera token do webhook).

**Card por empresa** (grade de cards; o card da empresa aberta tem borda amarela 3px):
- Cabeçalho: nome da empresa + pill amarela "dashboard aberto" quando é a empresa em uso (senão, botão "Abrir dashboard").
- Linhas em formato rótulo/valor, separadas por linhas finas:

| Rótulo | Conteúdo |
|---|---|
| SLUG | slug em monoespaçado com fundo cinza (ex.: `mirian`) |
| LEADS | "N no total · N em aberto · último DD/MM/AAAA, HH:MM:SS" ("em aberto" = leads que não estão em Fechamento) |
| WEBHOOK DE LEADS | input somente leitura com a URL + botão "Copiar" |
| GOOGLE | pill verde com o e-mail conectado ou pill vermelha "desconectado" |
| META ADS | pill verde "conectado" ou pill vermelha "desconectado" |
| USUÁRIOS | lista "Nome · e-mail · papel"; para outros usuários, botões "Nova senha" e "Remover" |

- Formulário de novo usuário no rodapé do card: "e-mail do usuário", "nome", select de papel (`admin` / `vendedor`) e botão "+ Criar login".
  - Cria o usuário no Supabase Auth e mostra a senha provisória uma única vez (ou envia convite por e-mail).
- "Nova senha" gera nova senha provisória; "Remover" pede confirmação e remove o vínculo do usuário com a empresa.
- Abaixo dos cards: bloco "COMO O ISOLAMENTO FUNCIONA" (seção 4).

**Abrir dashboard:** ao abrir uma empresa, todas as telas passam a mostrar os dados dela e o rodapé da sidebar mostra o nome dessa empresa.

### 6.7 Autenticação

- Login por e-mail e senha (Supabase Auth).
- Após login, o usuário cai em Métricas da sua empresa.
- "Sair" encerra a sessão e volta para a tela de login.
- Usuário sem empresa vinculada (e que não é admin da plataforma) vê mensagem de acesso pendente.

---

## 7. Modelo de dados (Supabase / Postgres)

```sql
empresas (
  id uuid pk, nome text, slug text unique,
  webhook_token text, webhook_ativo bool default true,
  url_envio text, ultimo_lead_em timestamptz,
  criado_em timestamptz
)

usuarios_empresa (
  user_id uuid fk auth.users, empresa_id uuid fk empresas,
  nome text, papel text check (papel in ('admin','vendedor')),
  pk (user_id, empresa_id)
)

admins_plataforma ( user_id uuid pk fk auth.users )

leads (
  id uuid pk, empresa_id uuid fk,
  nome text, telefone text, campanha text, anuncio text,
  etapa text check (etapa in ('novo','qualificacao','reuniao','ligacao','followup','fechamento')),
  qualificado bool null,            -- sim/não da qualificação
  etapa_max smallint,               -- maior etapa alcançada (funil cumulativo)
  compareceu bool null, ligacao_atendida bool null,
  resultado text null check (resultado in ('ganho','perdido')),
  valor numeric null, motivo_perda text null,
  agente_pausado bool default false,
  criado_em timestamptz, atualizado_em timestamptz,
  unique (empresa_id, telefone)
)

lead_etapas_historico ( id, lead_id, de, para, por_user_id, em )

mensagens (
  id uuid pk, empresa_id uuid fk, lead_id uuid fk,
  telefone text, autor text check (autor in ('cliente','agente','humano')),
  tipo text,          -- texto, audio, imagem, outro
  texto text, enviado_em timestamptz
)

reunioes (
  id uuid pk, empresa_id, lead_id,
  inicio timestamptz, fim timestamptz, titulo text,
  google_event_id text, status text   -- agendada, compareceu, faltou, cancelada
)

integracao_google (
  empresa_id pk, email text, calendario_id text, calendario_nome text,
  refresh_token text (criptografado), conectado_em timestamptz
)

integracao_meta (
  empresa_id pk, pixel_id text, access_token text (criptografado),
  evento_lead bool, evento_schedule bool, evento_contact bool, evento_purchase bool
)

eventos_meta (
  id, empresa_id, lead_id, evento text, status text, erro text, enviado_em timestamptz
)
```

**RLS:** todas as tabelas com `empresa_id` têm política `empresa_id in (select empresa_id from usuarios_empresa where user_id = auth.uid()) or auth.uid() in (select user_id from admins_plataforma)`. Tokens (webhook, Google, Meta) só são lidos por Edge Functions com service role; o frontend nunca recebe refresh tokens.

---

## 8. APIs e integrações

### 8.1 Webhook de leads (entrada)

`POST https://<projeto>.supabase.co/functions/v1/leads-webhook?empresa={slug}&token={token}`

Respostas: `200` ok · `401` token inválido · `403` webhook desativado · `404` slug inexistente · `422` payload inválido.

**Novo lead**
```json
{ "nome": "Fernando Assunção", "telefone": "556699853955",
  "campanha": "Teste chat n8n", "anuncio": "Criativo 01", "mensagem": "Olá!" }
```
Também aceita o payload bruto da WhatsApp Cloud API (extrai telefone, nome do perfil, texto e `referral` do anúncio: `source_id`, `headline`).

Regras: telefone normalizado só com dígitos; se o telefone já existe na empresa, atualiza o lead e registra a mensagem (não duplica). Atualiza `empresas.ultimo_lead_em`. Dispara `Lead` ao Meta se habilitado.

**Mensagem da conversa**
```json
{ "tipo": "mensagem", "telefone": "5542999418766",
  "texto": "Perfeito, registrado!", "autor": "cliente" }
```
`autor`: `"cliente"` ou `"agente"`. Cria o lead se não existir.

**Ações do fluxo (recomendado):** `{ "tipo": "etapa", "telefone", "etapa", "qualificado"? }` e `{ "tipo": "reuniao", "telefone", "inicio", "fim", "titulo" }` para o agente mover o lead e agendar.

**Consulta de pausa:** `GET .../leads-webhook/pausa?empresa=&token=&telefone=` → `{ "pausado": true|false }` — o N8N chama antes de o agente responder.

### 8.2 URL de envio (saída)

Ao responder pelo CRM: `POST {url_envio}` com
```json
{ "empresa": "mirian", "telefone": "5511947768212", "texto": "...", "autor": "humano", "usuario": "vendas@crm.com" }
```
Sucesso (2xx) grava a mensagem com `autor = humano` e marca `agente_pausado = true`. Falha mostra erro na conversa.

### 8.3 Google Calendar

- Conexão via OAuth (Supabase Auth, provedor Google) com acesso offline.
- Leitura: `events.list` da semana visível no calendário escolhido.
- Escrita: `events.insert` ao agendar; `events.patch`/`delete` ao remarcar/cancelar.
- Título padrão do evento: "{Empresa} | {tipo de reunião} - {nome do lead}".

### 8.4 Meta Conversions API

`POST https://graph.facebook.com/v{versão}/{pixel_id}/events`

| Gatilho no CRM | Evento |
|---|---|
| Lead criado | `Lead` |
| Lead movido para Reunião agendada | `Schedule` |
| Ligação registrada | `Contact` |
| Fechamento ganho | `Purchase` (`value`, `currency: BRL`) |

`user_data.ph` com o telefone em SHA-256; `action_source: "chat"` (ou `business_messaging` com `ctwa_clid` quando o anúncio for clique para WhatsApp); `event_id` = `{lead_id}:{evento}` para deduplicação. Cada envio é gravado em `eventos_meta` com status.

---

## 9. Requisitos não funcionais

- **Segurança:** RLS em todas as tabelas; tokens por empresa; segredos criptografados; HTTPS; tokens de webhook com no mínimo 32 caracteres aleatórios.
- **Tempo real:** novos leads e mensagens aparecem em até 2 s (Supabase Realtime).
- **Desempenho:** telas carregam em menos de 2 s com até 10 mil leads por empresa.
- **Responsividade:** funcional em desktop e iPad (layouts das telas de referência); em celular a sidebar vira menu recolhível.
- **Idioma e formatos:** pt-BR; datas `DD/MM/AAAA, HH:MM:SS`; moeda `R$`; fuso `America/Sao_Paulo`.
- **Auditoria:** histórico de mudanças de etapa com usuário e data.

---

## 10. Critérios de aceite

1. Um POST válido no webhook da empresa A cria o lead apenas na empresa A; o mesmo token não grava na empresa B.
2. Um usuário `vendedor` da empresa A não consegue ler dados da empresa B nem acessar Empresas.
3. Lead novo aparece em "Novo lead" e na lista de Mensagens sem recarregar a página.
4. Mover um lead até "Follow-up" faz o funil contar 1 em Qualificados.
5. Responder uma conversa pelo CRM chama a URL de envio e marca o agente como pausado para aquele telefone.
6. Agendar uma reunião cria o evento no Google Calendar conectado e ele aparece na Agenda como "Reunião do CRM".
7. Compromissos já existentes do Google aparecem em cinza e abrem o Google Calendar ao clicar.
8. Com Meta conectado e `Purchase` habilitado, um fechamento ganho gera um registro em "Últimos eventos para o Meta" com status.
9. "Regenerar token" invalida a URL anterior imediatamente.
10. O admin da plataforma cria uma empresa, cria um login e abre o dashboard dela; o card da empresa fica com borda amarela e pill "dashboard aberto".

---

## 11. Roadmap

- **v1.1:** métricas de "Métricas a definir" (tempo de primeira resposta, ciclo de venda, no-show por dia/horário, motivos de perda, follow-ups até fechar).
- **v1.2:** importação do gasto do Meta (Marketing API) para custo por lead e custo por fechamento.
- **v1.3:** filtros de período nas Métricas e exportação CSV.

## 12. Pontos em aberto

- Linha de campanha vazia na tabela de tracking: exibir rótulo "Sem campanha" em vez de célula em branco.
- O card do Meta pede só o Pixel/Dataset ID; a Conversions API exige também um access token — definir se será um token por empresa (campo adicional) ou um token de sistema da plataforma.
- Definir quem registra "compareceu" e "ligação atendida" (botões no card do lead ou ação do fluxo N8N).

---

Concebido por Romuald Członkowski — [www.aiadvisors.pl/en](https://www.aiadvisors.pl/en)
