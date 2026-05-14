# Bolão Copa do Mundo 2026 ⚽🏆

Bolão privado dos amigos para a FIFA World Cup 2026 (USA · Canadá · México · 11/jun a 19/jul de 2026).

**Como funciona:**
- 5 pontos pelo placar exato
- 3 pontos por acertar só o resultado (V/E/D)
- 0 pontos errando
- **Desempate:** saldo de gols (Σ |gols mandante − gols visitante|) dos palpites que pontuaram
- Admin cadastra cada amigo (sem cadastro público)
- Palpites travam automaticamente no apito inicial
- Resultados puxados automaticamente da API-Football (fallback: openfootball/worldcup.json)

## Stack

- Next.js 16 (App Router) + TypeScript
- shadcn/ui + Tailwind CSS + framer-motion
- Supabase (Postgres + Auth + RLS + Storage + Edge Functions + pg_cron)
- Vercel (free tier) para deploy

## Setup local

### 1. Pré-requisitos
- Node 20+
- [Supabase CLI](https://supabase.com/docs/guides/cli) instalado
- Docker Desktop (necessário pelo Supabase local)

### 2. Instalar dependências
```bash
npm install
```

### 3. Subir Supabase local
```bash
npx supabase start
```
Isso roda Postgres, Auth, Storage, Edge Functions e Studio (http://localhost:54323) localmente.

### 4. Aplicar migrations e seeds
```bash
npx supabase db reset
```
Esse comando roda todos os arquivos em `supabase/migrations/` em ordem:
- `0001_init.sql` — schema, funções de scoring, materialized view
- `0002_rls.sql` — Row Level Security
- `0003_seed_reference.sql` — grupos, estádios, 48 times
- `0004_seed_matches.sql` — 72 jogos de grupo + 32 placeholders de mata-mata
- `0005_cron.sql` — agendamento do sync-results (rodar manualmente no prod com `<PROJECT_REF>` substituído)

### 5. Configurar variáveis de ambiente
Copie `.env.example` para `.env.local`:
```bash
cp .env.example .env.local
```

Edite com os valores que aparecem no output do `supabase start`:
```
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon do supabase status>
SUPABASE_SERVICE_ROLE_KEY=<service_role do supabase status>
API_FOOTBALL_KEY=<sua chave free em api-football.com>
WORLD_CUP_LEAGUE_ID=1
WORLD_CUP_SEASON=2026
```

### 6. Criar o primeiro admin
No Supabase Studio (http://localhost:54323), vá em **Authentication → Add User**, crie sua conta com `email_confirm=true`, e depois execute no SQL Editor:
```sql
update profiles set is_admin = true where id = (select id from auth.users where email = 'seu@email.com');
```

### 7. Rodar dev server
```bash
npm run dev
```
Acesse http://localhost:3000.

## Deploy

### Banco e Edge Functions (Supabase Cloud)
1. Crie um projeto em https://supabase.com
2. `supabase link --project-ref <project-ref>`
3. `supabase db push` — sobe as migrations
4. `supabase functions deploy sync-results --no-verify-jwt` — deploya a Edge Function
5. Configure os secrets:
   ```bash
   supabase secrets set API_FOOTBALL_KEY=xxx WORLD_CUP_LEAGUE_ID=1 WORLD_CUP_SEASON=2026
   ```
6. Edite `supabase/migrations/0005_cron.sql` substituindo `<PROJECT_REF>` e `<SERVICE_ROLE_JWT>` e execute manualmente via SQL Editor.

### Frontend (Vercel)
1. Faça push do repositório no GitHub
2. Conecte na [Vercel](https://vercel.com), importe o projeto
3. Configure as 4 env vars (`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `API_FOOTBALL_KEY`)
4. Deploy

## Estrutura

```
app/
├─ (auth)/login          # tela de login (sem signup público)
├─ (app)/                # área autenticada
│  ├─ dashboard          # próximo jogo, meu ranking, últimos resultados
│  ├─ jogos              # lista + detalhe + formulário de palpite
│  ├─ ranking            # pódio + tabela completa
│  └─ regras             # explicação das regras
├─ (admin)/admin         # área restrita
│  ├─ usuarios           # CRUD de jogadores
│  ├─ jogos              # lançar placar + definir mata-mata
│  └─ seed               # operações de manutenção

components/
├─ ui/                   # primitives shadcn
└─ bolao/                # componentes específicos (MatchCard, Podium, etc)

lib/
├─ supabase/             # clients (browser/server/admin) + types
├─ scoring.ts            # mirror em TS da função SQL de pontos
├─ time.ts               # helpers de timezone (America/Sao_Paulo)
└─ validators.ts         # schemas zod

supabase/
├─ migrations/           # SQL do schema, RLS, seeds, cron
└─ functions/
   └─ sync-results/      # Edge Function que puxa resultados a cada 15min
```

## Trocar/atualizar dados da Copa

- **Calendário oficial publicado:** edite `supabase/migrations/0004_seed_matches.sql` ajustando horários reais ou rode o script `scripts/build-seed.ts` (a partir do openfootball/worldcup.json).
- **Sortear mata-mata:** use `/admin/jogos` → botão "Definir" em cada partida placeholder.
- **Lançar placar manualmente:** `/admin/jogos` → botão "Lançar" (também recalcula pontos).

## Decisões de projeto

- **Sem signup público** — só admin cadastra
- **Palpite travado no servidor** — RLS valida `kickoff_utc > now()`, não dá pra burlar via JS
- **Pontuação autoritativa em SQL** — função `recompute_match_points` é a fonte da verdade
- **Tudo em timestamptz UTC** — formatação só na borda (`lib/time.ts`)
- **Free tier estrito** — Vercel hobby + Supabase free, sem APIs pagas

Diversão entre amigos 🏆 Boa sorte na chave!
