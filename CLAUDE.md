# Instagram Dashboard — Contexto do Projeto

## Visão Geral

Dashboard web para análise e criação de conteúdo no Instagram.  
Stack: **Vercel** (frontend) + **n8n Cloud** (backend/OAuth) + **Supabase** (banco de dados) + **Meta Graph API** (dados do Instagram).

URL de produção: `https://instagram-alpha-self.vercel.app`  
Repositório GitHub: `https://github.com/gugamelo98-maker/instagram`

---

## Arquitetura

```
Browser (index.html)
  │
  ├── Meta OAuth → n8n webhook /instagram-auth → troca code por token
  ├── n8n webhook /instagram-data → busca perfil + métricas via Graph API
  ├── n8n webhook /instagram-posts → busca últimos posts
  └── Supabase (anon key) → lê dados salvos do usuário
```

O n8n salva tudo no Supabase via upsert. O frontend só lê do Supabase (anon key).  
**O App Secret NUNCA aparece no frontend.**

---

## Configuração do Frontend (index.html)

```js
const CONFIG = {
  META_APP_ID: '1123465306700530',
  N8N_WEBHOOK_BASE: 'https://rarelookingorca-n8n.cloudfy.live/webhook',
  SUPABASE_URL: 'https://sfjxgbkvnjfvzbguwwhf.supabase.co',
  SUPABASE_ANON_KEY: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InNmanhnYmt2bmpmdnpiZ3V3d2hmIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Njg5NjM3MzAsImV4cCI6MjA4NDUzOTczMH0.JtlgLxwYEkvIZEqGqDUOi3vng2HAf9M34XayW4w6orA',
  REDIRECT_URI: window.location.origin + '/callback.html',
};
```

---

## Segurança — REGRA OBRIGATÓRIA

> **O App Secret `1291ff3370b7fe375096b368538e38ac` NUNCA deve estar no frontend.**  
> Ele fica EXCLUSIVAMENTE em nodes do n8n no lado servidor.  
> Nunca colocar em index.html, callback.html, ou qualquer arquivo estático.

---

## n8n

- **Instância**: `https://rarelookingorca-n8n.cloudfy.live`
- **Workflow ID**: `Yjadcc52fNhLYrtJ` (Instagram Dashboard)
- **Versão atual**: `a0007431`
- **Status**: Ativo em produção

### Webhooks

| Path | Função |
|------|--------|
| `/instagram-auth` | Recebe code OAuth, troca por token, salva no Supabase |
| `/instagram-data` | Busca perfil + métricas da conta |
| `/instagram-posts` | Busca últimos posts com insights |

### Node de Supabase (upsert)

O node `💾 Salvar no Supabase` usa:
```json
{
  "contentType": "raw",
  "rawContentType": "application/json",
  "url": "https://sfjxgbkvnjfvzbguwwhf.supabase.co/rest/v1/ig_users?on_conflict=user_id"
}
```
Headers obrigatórios: `apikey`, `Authorization: Bearer <anon_key>`, `Prefer: resolution=merge-duplicates`.

---

## Supabase

- **Projeto ID**: `sfjxgbkvnjfvzbguwwhf`
- **Tabela principal**: `ig_users`
- **RLS**: Política `"Anon can upsert ig_users"` — `ALL` com `USING(true) WITH CHECK(true)`

### Estrutura da tabela `ig_users`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| user_id | text (PK) | Instagram user ID |
| username | text | @handle |
| name | text | Nome do perfil |
| biography | text | Bio atual |
| followers_count | int | Seguidores |
| following_count | int | Seguindo |
| media_count | int | Total de posts |
| profile_picture_url | text | URL da foto |
| account_type | text | BUSINESS / CREATOR / PERSONAL |
| access_token | text | Token de acesso (curto prazo) |
| updated_at | timestamptz | Última atualização |

> **Senhas não são salvas.** Só o access_token temporário do Meta. Cada token expira em 60 dias.

---

## Meta Graph API

- **App ID**: `1123465306700530`
- **Tipo de conta suportada**: Business ou Creator (contas pessoais não têm acesso à API de insights)
- **Redirect URI**: `https://instagram-alpha-self.vercel.app/callback.html`
- **Permissões usadas**: `instagram_basic`, `instagram_manage_insights`, `pages_show_list`, `business_management`

---

## Ferramentas de Conteúdo (8 Skills)

Baseadas no repositório de referência: `github.com/sergebulaev/instagram-skills`

Implementadas como tabs na seção "Ferramentas de Conteúdo" do dashboard:

| Tab | Skill | Função |
|-----|-------|--------|
| Legenda | `ig-caption-writer` | Gera caption com hook IG1-IG4, body, CTA, hashtags |
| Carrossel | `ig-carousel-planner` | Plano de 5-7 slides com cover + CTA final |
| Hook | `ig-hook-extractor` | Analisa hooks com 10 fórmulas IG1-IG10 |
| Hashtags | `ig-hashtag-strategist` | 3-5 tags sized (niche/mid/broad) |
| Humanizar | `ig-humanizer` | Audita e reescreve texto com AI tells |
| Plano | `ig-content-planner` | Plano semanal 4-post (Seg/Qua/Sex/Dom) |
| Repurpose | `ig-repurposer` | Transforma um post em 4 formatos |
| Perfil | `ig-profile-optimizer` | Auditoria 9 critérios do perfil |

### Regras de voz (de skills/root SKILL.md)

- Sem vocabulário AI: "significant", "crucial", "notably", "comprehensive", "insights", "robust", "leverage", "foster", "nuanced", "streamline", "elevate", "empower", "fundamentally", "essentially", "ultimately"
- Em dashes: máximo ~1 por 100 palavras (1-2 por caption)
- Hook: primeiros 125 chars devem ser autossuficientes (antes do "mais")
- Hashtags: 3-5 total (nunca 30), sempre relacionadas ao conteúdo

### 10 Fórmulas de Hook (IG1-IG10)

```
IG1: Número + Resultado Específico  → "7 técnicas que dobraram meu engajamento"
IG2: Antes / Depois                 → "Antes: zero vendas. Depois: R$12k em 30 dias"
IG3: Erro Comum                     → "O erro que 90% comete no Instagram"
IG4: Segredo / Bastidores           → "O que ninguém te conta sobre o algoritmo"
IG5: Pergunta Direta                → "Você ainda posta todo dia sem resultado?"
IG6: Afirmação Ousada               → "Stories não servem para vender. Até agora."
IG7: Micro-história                 → "Às 23h, com 312 seguidores, postei..."
IG8: Promessa de Transformação      → "Em 21 dias você vai parar de depender de trends"
IG9: Custo da Inação                → "Cada dia sem consistência é uma semana de atraso"
IG10: Lista Inesperada              → "3 coisas que o algoritmo pune (e você faz todos os dias)"
```

---

## Algoritmo Instagram 2026

Sinais em ordem de peso:
1. **Sends/Compartilhamentos** (DM) — sinal #1
2. **Saves** — sinal #2  
3. **Comentários** — sinal #3
4. **Likes** — sinal #4

Melhores horários para postar (Brasil):
- Terça–Sexta: 7h–9h, 12h–13h, 19h–21h
- Segunda e fins de semana: menor alcance

---

## Deploy

- **Vercel Project**: vinculado a `gugamelo98-maker/instagram` no GitHub
- **Auto-deploy**: qualquer push na branch `main` atualiza produção em ~30s
- **Domínio**: `instagram-alpha-self.vercel.app`

Para fazer deploy: commit + push para `main` → Vercel detecta e deploya automaticamente.

---

## Status de Implementação

- [x] OAuth Instagram (Meta Business Login)
- [x] Dashboard com métricas do perfil
- [x] Score do perfil (9 critérios)
- [x] Sinais do algoritmo
- [x] Estratégia de hashtags
- [x] Melhores horários de post
- [x] Recomendações personalizadas
- [x] Grid de posts recentes
- [x] Gerador de caption
- [x] CSS para todas as 8 ferramentas
- [x] HTML para todos os 8 painéis de ferramentas
- [x] JavaScript para todos os 8 skills
- [ ] Commit e push das ferramentas para GitHub (pendente)

---

## Arquivos do Projeto

```
/home/claude/instagram/
├── index.html      — Dashboard principal (toda a lógica aqui)
├── callback.html   — Página de retorno OAuth do Meta
├── CLAUDE.md       — Este arquivo (contexto e instruções)
└── .git/
```

---

## Como Continuar o Desenvolvimento

1. **Para alterar o dashboard**: editar `index.html`
2. **Para mudar fluxo OAuth**: editar workflow n8n `Yjadcc52fNhLYrtJ`
3. **Para ver dados no banco**: Supabase → tabela `ig_users`
4. **Para fazer deploy**: `git add . && git commit -m "mensagem" && git push`
5. **Para adicionar nova ferramenta**: adicionar tab no HTML + função JS + seguir padrão dos 8 existentes

### Skill de referência adicional não implementada
- `ig-audience-insights`: usa Apify para escanear hashtags e retornar audience insights — requer Actor do Apify

---

## Contexto de Desenvolvimento

Este projeto foi construído em sessão de Claude Code.  
Referência de skills: `github.com/sergebulaev/instagram-skills`  
Usuário: Gustavo Melo (`gugamelo98@gmail.com`)
