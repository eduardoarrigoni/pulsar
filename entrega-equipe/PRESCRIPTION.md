# Pulsar — Formato de Prescrição de Treino

**Situação:** Rascunho para revisão
**Data:** 2026-08-25
**Resolve:** D3 (formato de prescrição) e a parte mínima de D5 (entidades) necessária para anexá-lo
**Idioma:** Português. Termos escritos por extenso, sem siglas.

---

## O que é este documento

O Strava informa ao Pulsar o que o atleta **realmente fez**. Nada informa ao Pulsar o que o treinador **pediu** — isso o treinador digita diretamente no Pulsar. Este documento define o formato dessa prescrição, para que um computador consiga comparar as duas coisas.

É a raiz do esquema de dados. O motor de comparação lê daqui, a aderência é calculada a partir da comparação, e todas as análises leem a partir disso. Vale acertar antes da primeira migração.

---

## Decisões que este rascunho implementa

| # | Pergunta | Decisão |
|---|---|---|
| 1 | Quão expressivo? | **Formato fixo simples** — aquecimento, série principal, volta à calma. Sem aninhamento. |
| 2 | O que pode ser alvo? | Distância, duração, ritmo, frequência cardíaca. *(Nível de esforço fica definido, mas sem uso até a camada subjetiva entrar.)* |
| 3 | Valores exatos ou faixas? | **Valores exatos.** A tolerância fica no sistema, não na prescrição. |
| 4 | Como o treinador preenche? | **Um formulário estruturado grande, com campos anuláveis.** Sem formulário dinâmico ou adaptativo. |

---

## 1. O formato

Todo treino prescrito tem exatamente três blocos. Dois deles são opcionais.

```
TREINO
├── AQUECIMENTO        opcional — pode ficar vazio
├── SÉRIE PRINCIPAL    obrigatório
│     repetições × ( trecho de trabalho + recuperação )
└── VOLTA À CALMA      opcional — pode ficar vazio
```

Essa é toda a gramática. Uma rodagem leve de dez quilômetros é uma série principal de uma repetição, sem recuperação. Um treino intervalado é uma série principal de seis repetições com recuperação entre elas.

### Exemplo resolvido

O treinador escreve, no papel:

> 15 minutos leve, depois 6 × 800 metros a 3:50 por quilômetro com 90 segundos de recuperação, depois 10 minutos leve

O Pulsar armazena:

| Bloco | Campo | Valor |
|---|---|---|
| Aquecimento | duração | 900 segundos |
| Aquecimento | ritmo alvo | *(vazio)* |
| Série principal | repetições | 6 |
| Série principal | distância | 800 metros |
| Série principal | ritmo alvo | 230 segundos por quilômetro |
| Série principal | recuperação (duração) | 90 segundos |
| Volta à calma | duração | 600 segundos |

Agora "o atleta acertou o alvo na repetição 4?" é uma pergunta que tem resposta.

---

## 2. Unidades — decididas uma vez, nunca mais discutidas

Todos os valores são armazenados como **números inteiros em unidades base**. Formatar para exibição é trabalho da interface, nunca do banco de dados.

| Grandeza | Armazenada como | Exemplo |
|---|---|---|
| Distância | metros | `800` |
| Duração | segundos | `900` |
| Ritmo | **segundos por quilômetro** | `230` = 3:50 por quilômetro |
| Frequência cardíaca | batimentos por minuto | `165` |
| Nível de esforço | número inteiro de 1 a 10 | `7` |

**Por que o ritmo é armazenado em segundos por quilômetro e não como velocidade:** o treinador pensa e fala em minutos por quilômetro. Armazenar o inverso obriga toda leitura e toda escrita a fazer uma divisão, e os erros de arredondamento se acumulam nos dois sentidos. Armazene o que o treinador disse.

**Nunca armazene ritmo como texto.** `"3:50"` não pode ser comparado, somado nem ordenado sem antes ser interpretado, e abre a porta para `"3:50/km"`, `"3.50"` e `"03:50"` convivendo na mesma coluna.

---

## 3. O formulário que o treinador vê

Uma tela. Todos os campos visíveis. A maioria vazia em qualquer treino específico.

```
┌─ TREINO ──────────────────────────────────────────────┐
│ Nome          [ Intervalado 6x800                    ] │
│ Tipo          [ Intervalado       ▾ ]                  │
│ Data prevista [ 2026-09-02        ]                    │
│ Destinado a   [ ○ Atleta  ● Grupo ]  [ Grupo A    ▾ ] │
│ Observações   [                                      ] │
├─ AQUECIMENTO ─────────────── (deixe vazio para pular) ─┤
│ Distância [      ] m     Duração   [ 15:00 ]           │
│ Ritmo     [      ] /km   Freq. card. [   ] bpm         │
│ Observações [ trote leve                             ] │
├─ SÉRIE PRINCIPAL ──────────────────── (obrigatório) ───┤
│ Repetições [ 6 ]                                       │
│ Distância [ 800  ] m     Duração   [      ]            │
│ Ritmo     [ 3:50 ] /km   Freq. card. [   ] bpm         │
│ Esforço   [      ] 1-10        ← desabilitado por ora  │
│ ── recuperação entre as repetições ──                  │
│ Duração   [ 1:30 ]       Distância [      ] m          │
│ Observações [                                        ] │
├─ VOLTA À CALMA ───────────── (deixe vazio para pular) ─┤
│ Distância [      ] m     Duração   [ 10:00 ]           │
│ Ritmo     [      ] /km   Freq. card. [   ] bpm         │
│ Observações [                                        ] │
└────────────────────────────────────────────────────────┘
```

A interface aceita `3:50` e `15:00` do treinador e converte para números inteiros ao salvar. O treinador nunca digita segundos.

---

## 4. Regras de validação

O formulário é permissivo, mas não aceita qualquer coisa. Estas regras são o mínimo que impede o motor de comparação de receber dados sem sentido.

**Por bloco, quando o bloco está sendo usado:**

1. Um bloco que tenha *qualquer* campo preenchido precisa ter **distância ou duração**. "Corra a 3:50 por quilômetro" sem distância e sem duração não é um treino — não tem fim.
2. Distância e duração podem estar **ambas** preenchidas. Isso significa "corra 800 metros, mas pare aos 4 minutos se não tiver terminado". A comparação usa o que ocorrer primeiro.
3. Um alvo (ritmo ou frequência cardíaca) sem distância nem duração é inválido, pela regra 1.
4. Um bloco com **todos os campos vazios é um bloco pulado**, e isso é permitido apenas para aquecimento e volta à calma.

**Específico da série principal:**

5. Repetições precisa ser no mínimo 1. Padrão: 1.
6. Recuperação só pode ser preenchida quando repetições for maior que 1. A recuperação após a última repetição é ignorada.
7. A série principal não pode ser pulada.

**Limites de sanidade** — rejeite fora destas faixas, pois indicam erro de digitação, não intenção:

| Campo | Faixa aceita |
|---|---|
| Distância | 1 metro – 500.000 metros |
| Duração | 1 segundo – 86.400 segundos |
| Ritmo | 120 – 1.200 segundos por quilômetro (2:00 – 20:00 por quilômetro) |
| Frequência cardíaca | 30 – 230 batimentos por minuto |
| Repetições | 1 – 100 |

---

## 5. Armazenamento

O formulário é plano, conforme decidido. O armazenamento **não é**, e a razão merece dois parágrafos.

Se os três blocos virarem umas trinta colunas em uma única tabela (`warmup_distance_meters`, `main_target_pace…`, `cooldown_notes`), então no dia em que um treinador precisar de uma pirâmide — 400, 800, 1200, 800, 400 — a correção será uma migração de esquema que toca toda prescrição já escrita. Se, em vez disso, cada bloco for uma **linha em uma tabela filha**, esse mesmo dia custa uma linha a mais.

O formulário do treinador continua exatamente tão plano quanto o especificado. Isto trata apenas de onde os valores são gravados.

```sql
workout_prescription
  id                    uuid primary key
  assessoria_id         uuid not null              -- fronteira do inquilino
  created_by_coach_id   uuid not null
  name                  text not null
  workout_type          text not null              -- ver vocabulário abaixo
  scheduled_date        date not null
  notes                 text                       -- texto livre para o atleta
  is_structured         boolean not null default true
  free_text             text                       -- ver a válvula de escape, §7
  created_at            timestamptz not null
  updated_at            timestamptz not null

workout_block
  id                    uuid primary key
  prescription_id       uuid not null references workout_prescription
  block_type            text not null              -- 'warmup' | 'main' | 'cooldown'
  position              int  not null              -- 0, 1, 2 hoje; ordenação para depois

  repetitions           int  not null default 1

  -- o trecho de trabalho; ao menos um entre distância/duração é obrigatório
  distance_meters       int                        -- anulável
  duration_seconds      int                        -- anulável

  -- alvos; todos anuláveis
  target_pace_spk       int                        -- segundos por quilômetro
  target_heart_rate_bpm int
  target_effort         int                        -- 1-10, sem uso até a camada subjetiva

  -- recuperação, só faz sentido quando repetitions > 1
  recovery_duration_seconds  int
  recovery_distance_meters   int

  notes                 text

  unique (prescription_id, block_type, position)
```

**Hoje**, uma prescrição tem uma, duas ou três linhas em `workout_block`. **Depois**, uma pirâmide é cinco linhas `main` com `position` de 0 a 4, e nada mais muda.

### Atribuição

Uma prescrição é escrita uma vez e entregue a um atleta ou a um grupo inteiro. Mantenha isso separado da prescrição em si:

```sql
prescription_assignment
  id                    uuid primary key
  prescription_id       uuid not null references workout_prescription
  athlete_id            uuid                       -- um destes dois é preenchido
  group_id              uuid
  assigned_at           timestamptz not null
  check (num_nonnulls(athlete_id, group_id) = 1)
```

Atribuir a um grupo cria uma `session` por integrante (ver §8), de modo que um atleta que entrar no grupo na semana que vem não herde silenciosamente os treinos da semana passada.

---

## 6. Vocabulário de tipos de treino

Uma lista fechada. Ela alimenta a tabela de tolerância da §9, e é por ela que o treinador filtra.

| Valor | Rótulo exibido | Formato típico |
|---|---|---|
| `easy` | Rodagem leve | uma repetição, sem alvos ou com ritmo folgado |
| `long` | Longão | uma repetição, distância grande |
| `interval` | Intervalado | muitas repetições, ritmo alvo apertado, recuperação |
| `tempo` | Tempo / Limiar | uma repetição, ritmo alvo apertado |
| `fartlek` | Fartlek | repetições por duração em vez de distância |
| `hills` | Subidas | repetições, recuperação, ritmo alvo geralmente ausente |
| `race` | Prova | uma repetição, alvo pode ser um tempo final |
| `rest` | Descanso | nenhum bloco |
| `strength` | Força | não é comparado com o Strava; registrado apenas para carga |
| `other` | Outro | usa a tolerância padrão |

`rest` é o único tipo que legitimamente não tem série principal. Trate-o como uma prescrição com zero blocos — ele ainda importa, porque "o atleta descansou quando foi mandado descansar?" é uma pergunta de aderência real.

---

## 7. A válvula de escape — e ser honesto sobre o limite

O formato fixo simples não expressa tudo. Ele não consegue representar:

- **Pirâmides** — 400, 800, 1200, 800, 400
- **Séries dentro de séries** — 3 séries de (4 × 400 metros), 3 minutos entre as séries
- **Alvos que mudam ao longo das repetições** — "6 × 1 quilômetro, acelerando a cada um"
- **Blocos alternados** — 5 × (1 minuto forte / 1 minuto leve)

Essa foi uma troca deliberada em favor da viabilidade de construção. A válvula de escape é:

> Marque `is_structured = false`, coloque o treino em `free_text`, e a sessão é criada normalmente — mas a comparação entre planejado e realizado é **pulada** para ela. A atividade continua sendo importada, continua contando para a carga e continua aparecendo no histórico. Só a comparação fica ausente.

Duas regras que impedem isso de corroer o produto em silêncio:

1. **A interface precisa dizer.** Uma sessão com `is_structured = false` mostra "não comparado" em vez de mostrar aderência zero. Zeros silenciosos corromperiam todas as médias de grupo.
2. **Conte quantas são.** Se mais de um treino em cada cinco, aproximadamente, estiver caindo em texto livre durante o piloto, o formato simples foi a escolha errada — e a correção é a coluna `position`, que já está no lugar. Meça isso em vez de supor.

---

## 8. A que isso se conecta

A prescrição precisa que duas entidades existam. Esta é a fatia mínima de D5 — o modelo de domínio completo continua em aberto.

```
Assessoria  (a agência — a fronteira do inquilino)
   ├── Treinador
   ├── Grupo
   │      └── vínculo → Atleta   (historiado, ver abaixo)
   └── Atleta
             └── Sessão
```

**`Session`** é a junção no centro do sistema. É criada quando uma prescrição é atribuída a um atleta, e vai acumulando ao longo do tempo:

```sql
session
  id                    uuid primary key
  athlete_id            uuid not null
  assessoria_id         uuid not null
  prescription_id       uuid                       -- anulável! ver abaixo
  activity_id           uuid                       -- anulável! ver abaixo
  scheduled_date        date
  status                text not null              -- ver abaixo
  comparison_result     jsonb                      -- preenchido pelo motor de comparação
  coach_feedback        text
  created_at, updated_at
```

As duas chaves estrangeiras são anuláveis, e é justamente esse o ponto:

| `prescription_id` | `activity_id` | Significado | Situação |
|---|---|---|---|
| preenchido | nulo | prescrito, não realizado | `missed` |
| preenchido | preenchido | prescrito e realizado | `completed` |
| nulo | preenchido | corrida não planejada que o atleta fez mesmo assim | `unplanned` |

A linha `missed` não é um caso excepcional — ela é **como a aderência é medida**. Uma sessão sem atividade é o dado. É por isso que as sessões são criadas no momento da atribuição, e não no momento da importação.

> **Em aberto, e afeta o motor de comparação:** uma prescrição pode corresponder a duas atividades (um atleta que divide o treino entre manhã e noite), e duas prescrições podem corresponder a uma atividade? A recomendação é proibir ambos no produto mínimo viável e reavaliar com dados reais do piloto.

**O vínculo com o grupo precisa ser historiado.** Armazene como linhas com `joined_at` e `left_at`, e não como uma coluna `group_id` no atleta. Um atleta que muda do Grupo B para o Grupo A em março, caso contrário, reescreve retroativamente todas as médias do grupo desde janeiro. Isso não custa nada agora e é uma migração dolorosa depois.

---

## 9. Como a comparação usa isso — e onde mora a tolerância

Como os alvos são valores exatos, o motor de comparação precisa de uma tolerância para decidir se acertou ou errou. Ela **não** fica armazenada na prescrição. É uma configuração por assessoria, com padrão por tipo de treino:

| Tipo de treino | Tolerância de ritmo | Tolerância de distância | Razão |
|---|---|---|---|
| `interval` | ±3% | ±5% | precisão é o objetivo do treino |
| `tempo` | ±3% | ±5% | idem |
| `easy` | ±15% | ±10% | correr mais devagar que um alvo leve não é falha |
| `long` | ±10% | ±5% | a distância importa, o ritmo não |
| `hills` | não comparado | ±10% | ritmo em subida não significa nada sem a inclinação |
| `race` | ±2% | ±1% | — |
| padrão | ±10% | ±10% | — |

Três observações sobre esta tabela:

1. **Rodagens leves deveriam ser assimétricas.** Correr uma rodagem leve *mais devagar* que o prescrito é cumprir o treino; correr *mais rápido* é o erro de treinamento mais comum no corredor amador. Uma única porcentagem simétrica não consegue expressar isso. A recomendação é ter `pace_tolerance_slower` e `pace_tolerance_faster` como configurações separadas — um acréscimo pequeno agora, que torna possível depois um dos alertas mais úteis para o treinador.
2. **Estes números são um ponto de partida, não uma conclusão.** Ajuste-os com dados reais do piloto.
3. **Armazene, junto de cada resultado de comparação, a tolerância que foi usada.** Quando a tabela for reajustada, os números históricos de aderência não podem mudar em silêncio.

### O que o motor faz de fato

Para cada bloco `main` com `repetitions = N`:

1. Encontrar `N` esforços na atividade que correspondam à distância ou duração do bloco.
   - Se o arquivo tiver voltas marcadas pelo relógio (arquivos `.fit`), confie nelas primeiro.
   - Se não tiver — arquivos `.gpx` do aplicativo de celular do Strava **não têm voltas** — detecte os esforços a partir do sinal de ritmo.
2. Para cada esforço, comparar o ritmo obtido com `target_pace_spk` dentro da tolerância.
3. Registrar acerto ou erro por repetição, mais um resultado no nível da sessão.
4. Repetições faltando contam como erro. Repetições a mais são registradas, mas não elevam a aderência acima de completa.

Os passos 1 e 2 são a parte genuinamente difícil, e merecem documento próprio — este aqui só garante que o lado planejado seja inequívoco.

---

## 10. Pontos em aberto que este rascunho não resolve

| # | Item | Onde se resolve |
|---|---|---|
| 1 | Nível de esforço como alvo está definido, mas sem uso | ativa junto com a camada subjetiva |
| 2 | Uma prescrição ↔ duas atividades, e o inverso | recomendação: proibir por ora (§8) |
| 3 | Tolerância de ritmo assimétrica para rodagens leves | recomendação: adotar (§9) |
| 4 | Detecção de esforços pelo sinal de ritmo em arquivos `.gpx` | documento próprio — o risco técnico real |
| 5 | Biblioteca de modelos para o treinador reaproveitar treinos | não é necessária para começar; é necessária para o treinador gostar |
| 6 | Se `race` deveria ter um campo de tempo final alvo | uma coluna anulável, no dia em que for desejada |

---

## Resumo do que construir

1. Tabelas `workout_prescription` e `workout_block`, como na §5
2. `prescription_assignment`, criando uma `session` por atleta
3. `session` com as duas chaves estrangeiras anuláveis, como na §8
4. Vínculo de grupo historiado
5. O formulário plano da §3, com as validações da §4
6. Configurações de tolerância por assessoria, com os padrões da §9, gravadas junto de cada resultado de comparação

O motor de comparação em si é o próximo documento, e é a única peça com risco técnico real.
