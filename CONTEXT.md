# Bolão Kebolets — Contexto do Projeto

## URLs
- **App:** https://melatop.github.io/bolao-kebolets
- **Admin:** https://melatop.github.io/bolao-kebolets/admin.html (senha: `kebolets2026`)
- **GitHub:** github.com/melatop/bolao-kebolets
- **GitHub Token:** *(não armazenar aqui)*

## Stack
- **Frontend:** HTML/JS estático no GitHub Pages (`index.html`, `admin.html`)
- **Backend:** Supabase (`yciqdafnvcxrxgkwltxy.supabase.co`)
- **Anon Key:** *(ver no Supabase → Settings → API)*

## Banco de Dados (Supabase)

### Tabelas
- **`jogos`** — 72 jogos fase de grupos. Campos: `id` (ex: `bra-mar-0613`), `kickoff_utc`, `kickoff_brt` (texto `YYYY-MM-DD HH:MM`), `home_name`, `away_name`, `home_flag`, `away_flag`, `grupo`
- **`palpites`** — `nome`, `game_id`, `home_score`, `away_score`. Unique por `(nome, game_id)`
- **`resultados`** — `game_id`, `home_score`, `away_score`, `finalizado` (bool), `atualizado_em`
- **`usuarios`** — `nome` (PK), `senha`, `criado_em`

### Trigger importante
```sql
-- Recalcula kickoff_brt automaticamente ao inserir/atualizar kickoff_utc
create trigger trg_set_kickoff_brt
before insert or update of kickoff_utc on jogos
for each row execute function set_kickoff_brt();
```

### Secrets
- `SERVICE_ROLE_KEY`, `APIFOOTBALL_KEY` *(ver no Supabase → Edge Functions → Secrets)*

## Edge Functions
- **`sync-resultados`** — busca `worldcup26.ir/get/games` e salva placares. Roda via pg_cron a cada minuto.
- **`sync-apifootball`** — backup usando api-football.com (engatilhada, não ativa)

### Trocar para API-Football (quando necessário)
```sql
select cron.unschedule('sync-resultados');
select cron.schedule('sync-resultados', '* * * * *', $$
  select net.http_post(
    url := 'https://yciqdafnvcxrxgkwltxy.supabase.co/functions/v1/sync-apifootball',
    headers := '{"Content-Type": "application/json"}'::jsonb,
    body := '{}'::jsonb
  );
$$);
```

## Lógica Principal do App

### Datas e Horários
- **`kickoff_brt`** é a fonte de verdade para datas — sempre em formato `YYYY-MM-DD HH:MM` (texto, sem timezone)
- Jogos de hoje: `kickoff_brt.substring(0,10) === todayStr` onde `todayStr` é a data BRT atual
- Jogos de ontem sem resultado finalizado também aparecem em "hoje"

### Autenticação
- Nome sempre em UPPERCASE, só letras e números
- Senha salva no Supabase (`usuarios`) e no localStorage (`bolao_kebolets_pw`)
- Auto-login via localStorage
- Usuários com nome antigo (com espaço) passam por tela de rename inline

### Pontuação
- Placar exato = **3 pts**
- Resultado certo (só vencedor/empate) = **1 pt**
- Errou = **0 pts**

### Seções do App
1. **Em andamento** — jogos `isLocked` com resultado na API e `finalizado: false`
2. **Palpites do dia** — jogos de hoje que ainda não começaram
3. **Jogos do dia** — todos os jogos de hoje com placares
4. **Últimos jogos** — jogos de ontem finalizados
5. **Ranking** — pontos, ✅ (1pt), 🎯 (3pts), forma dos últimos 5 jogos
6. **Meu histórico** — palpites do usuário com resultado e pontuação

### Regras importantes
- `isLocked(game)`: `new Date() >= new Date(game.kickoff_brt || game.kickoff_utc)`
- "Em andamento" exige `resultados[g.id]` existente e `finalizado: false`
- Palpites deduplicados por `NOME.toUpperCase() + game_id`
- Usuário `TEST` oculto do ranking
- Auto-refresh a cada 60s quando há jogo ao vivo

## Arquivos
- `index.html` — app principal (~1000 linhas)
- `admin.html` — painel admin (inserir resultados, migrar nomes)
- `logo-cubic.png` — logo do rodapé
- `jogos-copa2026.sql` — SQL dos 72 jogos (já executado)
- `CONTEXT.md` — este arquivo

## Problemas Conhecidos / Histórico
- `kickoff_brt` deve ser texto `YYYY-MM-DD HH:MM` — tipo `timestamptz` causa conversão automática indesejada
- Jogos com horário UTC que cai em madrugada BRT podem aparecer no dia errado — corrigir manualmente: `update jogos set kickoff_brt = 'YYYY-MM-DD HH:MM' where id = '...'`
- pg_net precisa estar ativado para o cron funcionar (Database → Extensions)
- `renderGames` é síncrono mas usa `window._resultadosCache` que deve ser carregado antes

## Como fazer deploy
```bash
cd /home/claude/repo-bolao
git add .
git commit -m "mensagem"
git push
```
