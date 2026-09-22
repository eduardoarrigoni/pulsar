# Pulsar — Modelo de Dados

**Situação:** Rascunho para revisão
**Data:** 2026-08-25
**Resolve:** D4, D5, D6, D7 — e consolida o esquema do produto mínimo viável
**Depende de:** [PRESCRIPTION.md](PRESCRIPTION.md), [COMPARISON.md](COMPARISON.md), [OAUTH.md](OAUTH.md)
**Idioma:** Português. Termos escritos por extenso, sem siglas. Identificadores de tabela e coluna ficam em inglês, como é costume em código.

---

## Decisões aplicadas

| # | Decisão | Escolha |
|---|---|---|
| **D4** | Fórmula de carga | Baseada em ritmo, com melhoria por frequência cardíaca quando o sensor existir |
| **D5** | Relação treinador–atleta | Vários treinadores por assessoria, sem dono fixo — qualquer treinador vê e prescreve para qualquer atleta da assessoria |
| **D6** | Corte do produto mínimo viável | Núcleo mais painel de grupo. Fora: gráficos de evolução, periodização, provas, camada subjetiva |
| **D7** | Saída do atleta | Marcar como inativo, não apagar |

### Nota sobre D7

Marcar como inativo resolve o caso comum — o atleta saiu da assessoria — e é o que está modelado aqui.

O que ele **não** resolve é um pedido formal de exclusão sob a Lei Geral de Proteção de Dados, que é um direito do titular e não depende de o produto ter previsto. Isso não muda a decisão, e não estou reabrindo: o que faço aqui é modelar de modo que atender esse pedido depois custe uma linha de comando em vez de uma migração.

**A regra que garante isso:** dados pessoais existem em **um lugar só**, a tabela `athlete`. Nada mais no esquema guarda nome, email ou telefone — nem denormalizado "para facilitar a consulta", nem copiado dentro de resultado de comparação, nem em log de aplicação. Com essa disciplina, anonimizar um atleta é um `UPDATE` em uma linha, e o histórico de treino da assessoria continua de pé.

Se essa regra for quebrada em algum ponto, ela deixa de valer inteira. Vale tratá-la como invariante de revisão de código.

---

## Visão geral

```
assessoria ─┬─ coach
            ├─ training_group ── group_membership ─┐
            └─ athlete ──────────────────────────┬─┘
                  │                              │
                  ├─ provider_connection         │
                  │                              │
                  ├─ activity ─┬─ activity_stream│
                  │            └─ activity_lap   │
                  │                              │
                  └─ session ────────────────────┘
                        ↑
     workout_prescription ─┬─ workout_block
                           └─ prescription_assignment

     tolerance_setting  (por assessoria)
```

Quinze tabelas. Toda tabela de dados carrega `assessoria_id` — a fronteira do inquilino — mesmo quando poderia ser deduzida por junção. Isso é redundante de propósito: torna todo filtro de segurança uma condição direta, e um esquecimento vira erro de consulta em vez de vazamento entre assessorias.

---

## 1. Inquilino e pessoas

```sql
create table assessoria (
  id            uuid primary key,
  name          text not null,
  created_at    timestamptz not null default now(),
  is_active     boolean not null default true
);

create table coach (
  id            uuid primary key,
  assessoria_id uuid not null references assessoria,
  name          text not null,           -- DADO PESSOAL
  email         citext not null,         -- DADO PESSOAL
  password_hash text not null,
  role          text not null,           -- 'coach' | 'admin'
  is_active     boolean not null default true,
  created_at    timestamptz not null default now(),
  unique (assessoria_id, email)
);

create table athlete (
  id            uuid primary key,
  assessoria_id uuid not null references assessoria,

  -- DADOS PESSOAIS — este é o único lugar onde eles existem
  name          text not null,
  email         citext,
  phone         text,
  birth_date    date,

  -- parâmetros de treino, preenchidos pelo treinador
  threshold_pace_spk    int,   -- ritmo de limiar, segundos por quilômetro
  max_heart_rate        int,   -- batimentos por minuto
  resting_heart_rate    int,   -- batimentos por minuto

  is_active     boolean not null default true,
  inactivated_at timestamptz,
  created_at    timestamptz not null default now(),
  unique (assessoria_id, email)
);
```

**Sobre D5 — vários treinadores, sem dono fixo.** Não existe tabela ligando treinador a atleta. A regra de acesso é: *um treinador vê e prescreve para qualquer atleta da mesma assessoria*. É uma condição só (`coach.assessoria_id = athlete.assessoria_id`), e cobre férias, substituição e treino compartilhado sem nenhuma estrutura extra.

Se um dia a assessoria crescer a ponto de precisar restringir, o caminho é vincular treinador a `training_group` — uma tabela nova, sem mexer em nada existente.

**Sobre os parâmetros de treino.** `threshold_pace_spk` é o que torna a carga calculável (§5). Ele muda quando o atleta evolui, e é por isso que o valor usado fica gravado junto de cada cálculo, e não é lido da tabela na hora de exibir.

---

## 2. Grupos, com vínculo historiado

```sql
create table training_group (
  id            uuid primary key,
  assessoria_id uuid not null references assessoria,
  name          text not null,
  is_active     boolean not null default true,
  created_at    timestamptz not null default now()
);

create table group_membership (
  id            uuid primary key,
  assessoria_id uuid not null references assessoria,
  group_id      uuid not null references training_group,
  athlete_id    uuid not null references athlete,
  joined_at     date not null,
  left_at       date,                    -- nulo = ainda no grupo
  check (left_at is null or left_at >= joined_at)
);

create index on group_membership (group_id, joined_at, left_at);
create index on group_membership (athlete_id);
```

**Isto é o que sustenta o painel de grupo, e é a parte sutil da escolha do escopo.**

O painel de grupo pergunta coisas como "qual foi a carga média do Grupo A em março". A resposta correta considera quem estava no Grupo A *em março* — não quem está hoje. Sem o vínculo historiado, um atleta que mudou de grupo em abril reescreve retroativamente todos os números de março, e ninguém percebe.

O padrão de consulta, que vale escrever uma vez e reusar:

```sql
-- atletas do grupo em uma data específica
select athlete_id
from group_membership
where group_id = $1
  and joined_at <= $2
  and (left_at is null or left_at > $2);
```

Toda agregação de grupo passa por aqui. Se alguma consulta do painel fizer `join` direto de atleta com grupo sem a janela de data, ela está errada — e errada de um jeito silencioso.

---

## 3. Conexão com o Strava

Vem de [OAUTH.md](OAUTH.md), sem alterações de conteúdo.

```sql
create table provider_connection (
  id                uuid primary key,
  assessoria_id     uuid not null references assessoria,
  athlete_id        uuid not null references athlete,
  provider          text not null,          -- 'strava' | 'garmin' | 'coros' | 'polar'
  provider_user_id  text not null,          -- chave de roteamento dos avisos automáticos
  access_token      bytea not null,         -- CRIPTOGRAFADO EM REPOUSO
  refresh_token     bytea not null,         -- CRIPTOGRAFADO EM REPOUSO
  expires_at        timestamptz not null,
  scopes            text[] not null,
  connected_at      timestamptz not null default now(),
  revoked_at        timestamptz,
  unique (provider, provider_user_id)
);
```

Três lembretes que já custaram caro em outros projetos:

1. **Os dois tokens são criptografados em repouso.** Eles dão acesso a dados pessoais.
2. **O `refresh_token` gira a cada renovação e precisa ser regravado.** Não regravar é o erro que quebra o "autorizar uma vez só", e ele falha de forma intermitente — semanas depois, sob carga.
3. **`provider_user_id` é o identificador do atleta no Strava**, e é por ele que o aviso automático chega. Não é dado pessoal do seu banco, mas é identificador — se um dia houver exclusão real, ele sai junto.

---

## 4. Atividades

```sql
create table activity (
  id                uuid primary key,
  assessoria_id     uuid not null references assessoria,
  athlete_id        uuid not null references athlete,

  -- procedência, exigida pela obrigação de exclusão do Strava (§7.4)
  source            text not null,      -- 'strava' | 'file_upload' | 'manual'
  source_ref        text,               -- identificador da atividade no provedor
  ingested_at       timestamptz not null default now(),

  started_at        timestamptz not null,
  activity_type     text not null,      -- 'Run' | 'Ride' | ...
  name              text,

  distance_meters   int not null,
  elapsed_seconds   int not null,
  moving_seconds    int not null,
  elevation_gain_m  int,
  average_hr        int,                -- nulo se não houve sensor
  max_hr            int,

  -- carga, ver §5
  training_load     numeric(8,2),
  load_formula      text,               -- 'pace_v1' | 'hr_v1'
  load_inputs       jsonb,              -- as entradas usadas, para poder recalcular

  has_hr_stream     boolean not null default false,
  has_usable_laps   boolean not null default false,

  created_at        timestamptz not null default now(),
  unique (source, source_ref)
);

create index on activity (athlete_id, started_at desc);
create index on activity (assessoria_id, started_at desc);

create table activity_stream (
  activity_id   uuid primary key references activity on delete cascade,
  point_count   int not null,
  data          bytea not null,      -- séries comprimidas, resolução cheia
  keys          text[] not null      -- quais séries existem de fato
);

create table activity_lap (
  id            uuid primary key,
  activity_id   uuid not null references activity on delete cascade,
  lap_index     int not null,
  start_index   int not null,        -- índice DENTRO das séries temporais
  end_index     int not null,
  distance_meters   int not null,
  elapsed_seconds   int not null,
  moving_seconds    int not null,
  average_hr        int,
  unique (activity_id, lap_index)
);
```

**`unique (source, source_ref)`** é o que impede duplicata quando a exportação em massa e o aviso automático entregarem a mesma atividade — o que vai acontecer em torno da data de conexão de todo atleta.

**`has_usable_laps`** é gravado na importação, não calculado na hora de comparar. É o que permite ao painel dizer quantos atletas caem no Caminho A e quantos no Caminho B, que é a medição mais útil da primeira semana de piloto ([COMPARISON.md](COMPARISON.md) §2.1).

**As séries ficam em tabela separada e comprimidas.** Elas nunca mudam depois de importadas, são grandes e raramente lidas — cerca de 1,5 gigabyte para 30 atletas com 3 anos. Guardá-las cruas em resolução cheia é o que permite reprocessar o histórico inteiro quando o motor de comparação melhorar, sem gastar uma requisição da API.

> ⚠️ **Privacidade das coordenadas.** A série `latlng` mostra onde o atleta mora — o ponto de partida da maioria das corridas é a porta de casa. Isso é dado pessoal identificante de verdade, guardado dentro de `activity_stream.data`. Duas consequências: o painel de grupo não deve exibir mapas de outros atletas, e se um dia houver exclusão real, as séries saem junto (o `on delete cascade` já garante isso).

---

## 5. Carga de treinamento — D4

Escolha: **baseada em ritmo, com melhoria por frequência cardíaca.**

### 5.1 Fórmula base, por ritmo

Funciona para todo atleta, inclusive quem grava só pelo celular sem nenhum sensor.

```
intensidade = ritmo_limiar / ritmo_realizado
carga       = (duração_em_segundos / 3600) × intensidade² × 100
```

Com os ritmos em segundos por quilômetro. Como ritmo menor significa mais rápido, a razão fica acima de 1 quando o atleta correu mais forte que o limiar, e o expoente 2 faz o esforço intenso pesar mais que o volume leve — que é o comportamento que treinador espera.

Exemplo: uma hora a 5:00 por quilômetro, com limiar em 4:30.
```
intensidade = 270 / 300 = 0,9
carga       = 1 × 0,81 × 100 = 81
```

### 5.2 Melhoria por frequência cardíaca

Quando `has_hr_stream` for verdadeiro e o atleta tiver `max_heart_rate` e `resting_heart_rate` preenchidos, uma estimativa melhor:

```
reserva     = (frequência_média - frequência_repouso) / (frequência_máxima - frequência_repouso)
carga       = (duração_em_minutos) × reserva × fator
```

O `fator` calibra as duas escalas para a mesma faixa de números, para que o treinador não veja a carga saltar quando o atleta esquece a cinta. Calibre com dados reais do piloto — não invente um número agora.

### 5.3 As três regras que não podem ser quebradas

1. **Grave qual fórmula produziu cada valor** (`load_formula`). Quando a percepção subjetiva de esforço chegar no projeto final e virar uma terceira fórmula, os valores históricos não podem mudar de significado em silêncio.
2. **Grave as entradas** (`load_inputs`), incluindo o `threshold_pace_spk` que estava valendo naquele dia. O limiar do atleta melhora ao longo do ano; sem isso, recalcular o histórico com o limiar de hoje faz março inteiro parecer mais fácil do que foi.
3. **Sem `threshold_pace_spk`, não há carga.** Deixe `training_load` nulo e mostre "limiar não definido" em vez de chutar um valor. Um número inventado atravessa o painel de grupo e contamina a média de todo mundo.

---

## 6. Prescrição

Sem alterações em relação a [PRESCRIPTION.md](PRESCRIPTION.md) §5, apenas com `assessoria_id` acrescentado onde faltava.

```sql
create table workout_prescription (
  id                  uuid primary key,
  assessoria_id       uuid not null references assessoria,
  created_by_coach_id uuid not null references coach,
  name                text not null,
  workout_type        text not null,
  scheduled_date      date not null,
  notes               text,
  is_structured       boolean not null default true,
  free_text           text,
  created_at          timestamptz not null default now(),
  updated_at          timestamptz not null default now()
);

create table workout_block (
  id                  uuid primary key,
  prescription_id     uuid not null references workout_prescription on delete cascade,
  block_type          text not null,      -- 'warmup' | 'main' | 'cooldown'
  position            int  not null,
  repetitions         int  not null default 1,
  distance_meters     int,
  duration_seconds    int,
  target_pace_spk     int,
  target_heart_rate_bpm int,
  target_effort       int,                -- sem uso até a camada subjetiva
  recovery_duration_seconds int,
  recovery_distance_meters  int,
  notes               text,
  unique (prescription_id, block_type, position)
);

create table prescription_assignment (
  id              uuid primary key,
  assessoria_id   uuid not null references assessoria,
  prescription_id uuid not null references workout_prescription,
  athlete_id      uuid,
  group_id        uuid,
  assigned_at     timestamptz not null default now(),
  check (num_nonnulls(athlete_id, group_id) = 1)
);
```

---

## 7. Sessão — a junção central

```sql
create table session (
  id                uuid primary key,
  assessoria_id     uuid not null references assessoria,
  athlete_id        uuid not null references athlete,
  prescription_id   uuid references workout_prescription,   -- anulável
  activity_id       uuid references activity,               -- anulável
  scheduled_date    date,
  status            text not null,     -- 'planned' | 'completed' | 'missed' | 'unplanned'

  -- resultado agregado, o que o painel de grupo consulta
  adherence_pct     numeric(5,2),      -- nulo quando não foi possível comparar
  confidence        text,              -- 'high' | 'medium' | 'low' | 'not_compared'
  tolerance_used    jsonb,             -- a tolerância vigente no cálculo

  -- detalhe por repetição, o que a tela do atleta abre
  comparison_result jsonb,

  coach_feedback    text,
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now(),
  unique (athlete_id, prescription_id)
);

create index on session (athlete_id, scheduled_date desc);
create index on session (assessoria_id, scheduled_date desc);
create index on session (status);
```

As duas chaves estrangeiras são anuláveis de propósito:

| `prescription_id` | `activity_id` | Significado | `status` |
|---|---|---|---|
| preenchido | nulo, data futura | treino prescrito, ainda não chegou | `planned` |
| preenchido | nulo, data passada | prescrito e não realizado | `missed` |
| preenchido | preenchido | prescrito e realizado | `completed` |
| nulo | preenchido | corrida não planejada | `unplanned` |

A linha `missed` **é o dado**, não uma ausência de dado — é assim que a aderência é medida. Por isso a sessão nasce no momento da atribuição, e não quando a atividade chega.

**`adherence_pct` e `confidence` são colunas, e não campos do `jsonb`.** O painel de grupo agrega esses dois números para dezenas de atletas por semana; agregar dentro de `jsonb` funciona, mas fica lento e ilegível rápido. O detalhe por repetição continua em `jsonb`, porque só é lido uma sessão por vez.

**Quando `confidence` for `low` ou `not_compared`, `adherence_pct` é nulo — nunca zero.** Zero significa "o atleta não fez o treino", e é um julgamento sobre a pessoa. Nulo significa "o software não conseguiu comparar". Toda agregação do painel de grupo precisa ignorar os nulos, não tratá-los como zero, ou uma falha de software vira cobrança em cima de um atleta que treinou direito.

---

## 8. Tolerância

```sql
create table tolerance_setting (
  id                     uuid primary key,
  assessoria_id          uuid not null references assessoria,
  workout_type           text not null,
  pace_tolerance_faster  numeric(4,2),   -- porcentagem; nulo = não compara ritmo
  pace_tolerance_slower  numeric(4,2),
  distance_tolerance     numeric(4,2),
  unique (assessoria_id, workout_type)
);
```

Duas tolerâncias de ritmo separadas, e não uma. Correr uma rodagem leve mais devagar que o alvo é cumprir o treino; correr mais rápido é o erro mais comum do corredor amador. Uma porcentagem simétrica não consegue dizer isso, e é o que torna possível depois um dos alertas mais úteis para o treinador.

Padrões em [PRESCRIPTION.md](PRESCRIPTION.md) §9. A tolerância vigente no momento do cálculo fica copiada em `session.tolerance_used`, para que reajustar a tabela não reescreva a história.

---

## 9. O painel de grupo — a consulta que importa

O escopo escolhido inclui o painel de grupo, e ele tem exatamente uma consulta difícil. Vale deixá-la escrita:

```sql
-- carga e aderência médias de um grupo, por semana, respeitando o vínculo historiado
select
  date_trunc('week', s.scheduled_date) as semana,
  count(distinct s.athlete_id)         as atletas,
  avg(a.training_load)                 as carga_media,
  avg(s.adherence_pct)                 as aderencia_media,   -- ignora nulos automaticamente
  count(*) filter (where s.status = 'missed') as faltas
from session s
join group_membership gm
  on  gm.athlete_id = s.athlete_id
  and gm.group_id   = $1
  and gm.joined_at <= s.scheduled_date
  and (gm.left_at is null or gm.left_at > s.scheduled_date)
left join activity a on a.id = s.activity_id
where s.assessoria_id = $2
  and s.scheduled_date between $3 and $4
group by 1
order by 1;
```

Três coisas nela que são fáceis de errar:

1. **A junção com `group_membership` usa a janela de datas.** Sem isso, quem mudou de grupo em abril reescreve março.
2. **`avg` ignora nulos sozinho**, que é exatamente o comportamento desejado para `adherence_pct` — sessões não comparadas somem da média em vez de virar zero.
3. **`left join` na atividade**, porque sessão `missed` não tem atividade e precisa continuar aparecendo na contagem de faltas.

> **Aviso de expectativa, que já estava na escolha do escopo:** este painel não mostra nada útil enquanto vários atletas não tiverem semanas de dados. Na primeira semana de piloto ele vai estar quase vazio, e isso é o esperado, não um defeito.

---

## 10. O que não está aqui

Fora do corte, de propósito:

| Item | Onde entra |
|---|---|
| Gráficos de evolução | precisa de meses de histórico para mostrar algo |
| Periodização e mesociclos | projeto final |
| Provas e resultados | projeto final |
| Camada subjetiva — esforço percebido, sono, fadiga, dor | projeto final; a coluna `target_effort` já está reservada |
| Vínculo de treinador com grupo | só se a assessoria crescer a ponto de precisar restringir |
| Exclusão real de dados pessoais | D7; a disciplina de dados pessoais em um lugar só deixa isso barato |

---

## 11. O que construir, em ordem

1. `assessoria`, `coach`, `athlete`, autenticação e permissão por inquilino
2. `training_group`, `group_membership` com a janela de datas desde o primeiro dia
3. `provider_connection` mais o fluxo de autorização de [OAUTH.md](OAUTH.md)
4. `activity`, `activity_stream`, `activity_lap` e a importação por aviso automático
5. Importação da exportação em massa para o histórico
6. `workout_prescription`, `workout_block`, `prescription_assignment`, `session`
7. Caminho A do motor de comparação ([COMPARISON.md](COMPARISON.md) §2.2)
8. Carga por ritmo (§5.1)
9. Painel individual
10. Painel de grupo (§9)
11. Caminho B do motor, depois de medir quanto ele importa
12. Melhoria da carga por frequência cardíaca (§5.2)

Do 1 ao 9 já é um produto que uma assessoria consegue usar. O 10 é o que o dono da assessoria quer ver. O 11 e o 12 dependem do que o piloto mostrar, e não deveriam ser construídos antes disso.
