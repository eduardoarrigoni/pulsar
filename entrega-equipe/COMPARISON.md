# Pulsar — Motor de Comparação (Planejado × Realizado)

**Situação:** Rascunho para revisão
**Data:** 2026-08-25
**Resolve:** os itens 2 e 3 da Constatação 5 — segmentação e casamento
**Depende de:** [PRESCRIPTION.md](PRESCRIPTION.md), que define o lado planejado
**Idioma:** Português. Termos escritos por extenso, sem siglas.

---

## O que é este documento

A prescrição diz o que o treinador pediu. A API do Strava diz o que o atleta fez. Este documento define como o computador liga uma coisa à outra e responde: **"o atleta acertou a repetição 4?"**

É a única peça do produto mínimo viável com risco técnico real. Todo o resto é cadastro, formulário e gráfico.

---

## Correção de premissa

Um rascunho anterior tratava a diferença entre arquivos `.fit` e `.gpx` como o problema central. **Isso estava errado para a via principal.**

Os dados entram principalmente pela **API**, não pela importação de arquivo. E a API do Strava **normaliza tudo**: seja o atleta um usuário de Garmin ou alguém que gravou pelo celular, os dados voltam na mesma estrutura de séries temporais. O formato original de gravação some no meio do caminho.

O que **de fato** varia entre atletas não é o formato. São duas outras coisas, e o motor precisa ser desenhado em torno delas:

| Varia | Por quê | Consequência |
|---|---|---|
| **Quais séries existem** | é uma questão de *sensor*, não de formato — sem cinta ou sensor óptico, não há frequência cardíaca em fonte nenhuma | alvos de frequência cardíaca podem ser inverificáveis |
| **Se as voltas são úteis** | é uma questão de *comportamento* — depende de o atleta ter marcado voltas ou programado o treino no relógio | decide entre o caminho fácil e o caminho difícil |

A importação de arquivo continua existindo para a carga histórica, mas o modelo interno deve ser desenhado a partir do que a API entrega. O leitor de arquivo se adapta a ele, e não o contrário.

---

## 1. O que a API entrega

### 1.1 As séries temporais

`GET /activities/{id}/streams` devolve vetores paralelos, todos do mesmo comprimento, um ponto por registro:

| Chave | Unidade | Observação |
|---|---|---|
| `time` | segundos desde o início | não é necessariamente 1 por segundo — pode ter intervalos irregulares |
| `distance` | metros, **acumulados** | distância de um trecho = `distance[fim] - distance[início]` |
| `velocity_smooth` | metros por segundo | já suavizado pelo Strava |
| `altitude` | metros | |
| `heartrate` | batimentos por minuto | **ausente** se não houve sensor |
| `cadence` | passos por minuto **de uma perna** | ver a armadilha abaixo |
| `grade_smooth` | porcentagem | inclinação, permite calcular ritmo ajustado |
| `moving` | verdadeiro ou falso | permite descontar tempo parado |
| `latlng` | par de coordenadas | para mapa, não para a comparação |
| `watts`, `temp` | — | irrelevantes para corrida na maioria dos casos |

**Três coisas que precisam estar claras antes de escrever código:**

1. **Uma série indisponível é simplesmente omitida da resposta**, não vem preenchida com nulos. O código precisa checar a *existência* da chave, não o conteúdo dela. Um atleta sem sensor de frequência cardíaca não devolve `heartrate` de jeito nenhum.
2. **A cadência é por perna.** O Strava devolve algo em torno de 85, não os 170 que o corredor conhece. Multiplique por 2 antes de exibir, ou você vai passar meses explicando isso para treinadores.
3. **Nunca peça resolução reduzida.** O parâmetro `resolution` aceita `low` (100 pontos), `medium` (1.000), `high` (10.000) e `all`. Reduzir a resolução **destrói as fronteiras dos intervalos** — é exatamente o detalhe que o motor precisa. Peça sempre a resolução cheia, e guarde ela.

### 1.2 As voltas — e o detalhe que resolve metade do problema

`GET /activities/{id}` devolve, junto do detalhe, um vetor `laps`. Cada volta traz `elapsed_time`, `moving_time`, `distance`, `average_speed`, `average_heartrate`, `lap_index` — e, o que interessa de verdade:

> **`start_index` e `end_index` são índices dentro das séries temporais.**

Isso é o presente. Se as voltas forem úteis, a segmentação não precisa ser calculada: a repetição 4 é literalmente a fatia `streams[lap[3].start_index : lap[3].end_index]`. Sem detecção, sem heurística, sem erro.

Metade do trabalho deste documento é aproveitar isso quando dá, e a outra metade é o que fazer quando não dá.

---

## 2. Os dois caminhos

```
                 a atividade tem voltas úteis?
                              │
                ┌─────────────┴─────────────┐
               sim                          não
                │                            │
        CAMINHO A — fatiar               CAMINHO B — buscar
        pelos índices das voltas         guiado pela prescrição
        barato, exato                    caro, aproximado, com confiança
```

### 2.1 Decidindo se as voltas são úteis

Uma atividade **sempre** traz pelo menos uma volta. Um atleta que nunca apertou o botão de volta traz exatamente uma, cobrindo a corrida inteira — o que é inútil para comparar seis repetições.

O teste, para uma série principal de `N` repetições:

```
voltas_uteis =
      número de voltas >= N + 1
  E   existem N voltas cujo tamanho bate com o do trecho prescrito,
      dentro de uma tolerância folgada (±25%)
```

A tolerância aqui é deliberadamente folgada. Não é o julgamento de acerto ou erro — é só a pergunta "essas voltas parecem ser este treino?". O julgamento vem depois, com a tolerância real da prescrição.

Se `voltas_uteis` for verdadeiro, siga o Caminho A. Se não, Caminho B.

> **Nota de realidade:** atletas de assessoria que programam o treino no relógio caem quase sempre no Caminho A, e o resultado é praticamente perfeito. Atletas que correm com o celular no bolso caem quase sempre no Caminho B. Os dois vão existir no mesmo grupo, e a proporção entre eles é a coisa mais útil que o piloto pode medir na primeira semana.

### 2.2 Caminho A — fatiar pelas voltas

1. Descartar as voltas que correspondem a aquecimento, recuperação e volta à calma. Regra prática: entre as voltas candidatas, as `N` cujo tamanho mais se aproxima do trecho prescrito são as repetições; as demais são intervalo.
2. Para cada repetição, fatiar as séries por `start_index` e `end_index`.
3. Medir (§3) e julgar contra a tolerância da prescrição.

Custo: desprezível. Precisão: alta.

---

## 3. Caminho B — busca guiada pela prescrição

Aqui está o risco técnico do projeto, e vale explicar por que ele é menor do que parece.

**Não construa um detector genérico de intervalos.** Detectar intervalos "às cegas", sem saber o que se procura, é um problema difícil e mal definido. Mas o Pulsar **não está às cegas**: a prescrição diz exatamente o que procurar — seis trechos de aproximadamente 800 metros a aproximadamente 230 segundos por quilômetro, separados por aproximadamente 90 segundos.

Isso transforma o problema de *segmentação* em *busca por um padrão conhecido*, que é muito mais tratável.

### 3.1 O algoritmo

Dado um bloco `main` com `N` repetições, distância `D` e ritmo alvo `P`:

**Passo 1 — janela deslizante por distância.**
Para cada índice `i` da série, encontrar o menor `j` tal que `distance[j] - distance[i] >= D`. Isso dá "o trecho de `D` metros que começa em `i`".

Como `distance` é acumulada e monotônica, isso é feito com dois ponteiros em uma passada só — custo linear, não quadrático.

**Passo 2 — ritmo de cada janela.**
```
ritmo(i) = (time[j] - time[i]) / (distance[j] - distance[i]) × 1000
```
em segundos por quilômetro.

**Passo 3 — filtrar candidatos.**
Manter as janelas cujo ritmo esteja dentro de uma banda folgada em torno de `P` — sugestão: ±20%, novamente mais folgada que a tolerância de julgamento. Estamos achando as repetições, não avaliando ainda.

**Passo 4 — escolher `N` janelas sem sobreposição.**
Ordenar os candidatos pelo quão perto estão de `P`. Guloso: pegar o melhor, descartar tudo que sobrepõe, repetir até ter `N` ou até acabarem os candidatos.

**Passo 5 — validar o espaçamento.**
As `N` janelas escolhidas deveriam estar separadas por aproximadamente a recuperação prescrita. Se os intervalos entre elas forem coerentes, a confiança sobe. Se forem caóticos — três repetições coladas e uma vinte minutos depois — a confiança cai, e provavelmente o casamento está errado.

**Passo 6 — medir e julgar** (§4).

### 3.2 Blocos por duração

Fartlek e treinos por tempo usam o mesmo algoritmo, trocando a janela por distância pela janela por tempo: menor `j` tal que `time[j] - time[i] >= duração`. Todo o resto é idêntico.

### 3.3 Casos que precisam de tratamento explícito

| Caso | O que acontece | Tratamento |
|---|---|---|
| Aquecimento atinge o ritmo alvo por acaso | vira candidato falso | a validação de espaçamento (passo 5) normalmente elimina; além disso, ignore candidatos antes do fim do aquecimento prescrito |
| O atleta fez 4 das 6 repetições | só 4 janelas passam no filtro | 4 acertos e 2 faltas — **isto é um resultado válido**, não uma falha do motor |
| O atleta fez 7 repetições | 7 candidatos | pegue as 6 melhores; registre a extra sem elevar a aderência acima de completa |
| Parada no meio da repetição | tempo decorrido infla o ritmo | use a série `moving` para descontar; ver §4 |
| Falha de sinal do relógio | buraco na distância | um salto anormal em `distance` entre pontos consecutivos invalida janelas que o atravessam |
| Treino em subida | o ritmo não significa nada | tipo `hills` não compara ritmo, por decisão já tomada em [PRESCRIPTION.md](PRESCRIPTION.md) §9 |

---

## 4. Medindo o trecho encontrado

Achada a fatia `[i, j]`, os números finais:

```
distância        = distance[j] - distance[i]
tempo decorrido  = time[j] - time[i]
tempo em movimento = soma de Δtime onde moving == verdadeiro
ritmo            = tempo / distância × 1000
frequência cardíaca média = média de heartrate[i..j]     (se a série existir)
```

Duas decisões dentro disso:

**Ritmo medido a partir de `distance` e `time`, não de `velocity_smooth`.** A velocidade suavizada é ótima para *encontrar* os trechos, porque o alisamento tira o ruído. Mas ela atrasa nas transições, e para *medir* o resultado o cálculo direto a partir da distância acumulada é mais fiel ao que o relógio marcou.

**Dentro de uma repetição, use tempo decorrido.** Parar no meio de um tiro de 800 metros é parte do desempenho, não é uma pausa a ser desconsiderada. Já *entre* as repetições, o tempo em movimento é o que importa. A série `moving` distingue os dois casos.

O julgamento — acerto ou erro — usa a tabela de tolerância de [PRESCRIPTION.md](PRESCRIPTION.md) §9, e a tolerância aplicada fica gravada junto do resultado.

---

## 5. Confiança — e o que mostrar quando não se sabe

O Caminho B às vezes vai errar. Um motor que erra em silêncio é pior que um motor que admite dúvida, porque um número errado de aderência contamina toda média de grupo que passar por ele.

Cada comparação devolve uma **confiança**:

| Nível | Quando | O que a interface mostra |
|---|---|---|
| **Alta** | Caminho A, ou Caminho B com `N` janelas achadas e espaçamento coerente | o resultado, direto |
| **Média** | Caminho B com menos de `N` janelas, ou espaçamento irregular | o resultado, com marca de "conferir" |
| **Baixa** | quase nada casou, ou a atividade não parece o treino prescrito | **não mostra número** — mostra "não foi possível comparar" e oferece revisão manual |

**A regra que não pode ser quebrada:** confiança baixa nunca vira aderência zero. Aderência zero significa "o atleta não fez o treino", e é um julgamento sobre o atleta. "Não consegui comparar" é um julgamento sobre o software. Confundir os dois faz o treinador cobrar o atleta errado, e é o tipo de erro que faz uma assessoria abandonar a ferramenta.

Isso vale igualmente para as sessões marcadas como `is_structured = false` em [PRESCRIPTION.md](PRESCRIPTION.md) §7.

---

## 6. Aquecimento e volta à calma

Não vale comparar com rigor. Ninguém trota exatamente 15 minutos, e nenhum treinador se importa.

Compare apenas o total: distância ou duração do bloco contra o trecho da atividade antes da primeira repetição (aquecimento) e depois da última (volta à calma). Sem alvo de ritmo, a menos que o treinador tenha preenchido um — e, mesmo assim, com a tolerância folgada de `easy`.

Isso também dá de graça o limite inferior de busca do Caminho B: candidatos antes do fim esperado do aquecimento são descartados.

---

## 7. Armazenamento das séries

Preocupação legítima, que se resolve com uma conta.

Uma corrida de uma hora tem cerca de 3.600 pontos. Com seis séries úteis a 4 bytes cada, dá cerca de 86 kilobytes por atividade, sem compressão. Para uma assessoria de 30 atletas com 3 anos de histórico — cerca de 18.000 atividades — são cerca de **1,5 gigabyte**, e bem menos que isso comprimido.

Não é um problema. Recomendações:

1. **Guarde as séries cruas em resolução cheia**, comprimidas, fora da tabela principal. Elas nunca mudam depois de importadas.
2. **Guarde o resultado da comparação separado** (`session.comparison_result`), porque ele *muda* quando a tolerância é reajustada ou o motor melhora.
3. **Poder recalcular sem reimportar é o ponto.** É por isso que as séries cruas ficam guardadas em vez de só os números derivados: quando o Caminho B melhorar, você reprocessa o histórico inteiro sem gastar uma requisição sequer.

---

## 8. Ordem de construção

1. Buscar e guardar séries e voltas na importação — sem nenhuma comparação ainda
2. Caminho A completo, com medição e julgamento
3. Medir, no piloto, a proporção Caminho A × Caminho B
4. Caminho B, passos 1 a 4
5. Validação de espaçamento e níveis de confiança
6. Aquecimento e volta à calma

Os passos 1 e 2 já entregam comparação funcionando para os atletas que programam o treino no relógio, que é provavelmente a maioria dos atletas sérios de uma assessoria. O passo 3 diz quanto o Caminho B realmente importa antes de você investir nele.

---

## 9. Riscos e itens em aberto

| # | Item | Avaliação |
|---|---|---|
| 1 | O Caminho B pode não atingir precisão aceitável em treinos irregulares | risco real; mitigado por confiança explícita em vez de número errado |
| 2 | Ritmo ajustado pela inclinação não está definido | `grade_smooth` existe; adiar até o piloto mostrar se faz falta |
| 3 | Sessões divididas em duas atividades | proibido por ora, por [PRESCRIPTION.md](PRESCRIPTION.md) §8 |
| 4 | Alvos de frequência cardíaca para atletas sem sensor | inverificáveis; a interface precisa dizer isso na hora da prescrição, não depois |
| 5 | Esteira e corrida indoor | distância vem do relógio, não do sinal de satélite, e pode ser bem imprecisa — decidir se entra na comparação |
| 6 | Sem acesso às séries de atividades de terceiros | a API só entrega séries das atividades do próprio atleta autenticado; cada atleta precisa ter autorizado |

---

## Resumo

O medo de que formatos diferentes de arquivo quebrassem a comparação não se aplica à via principal: a API normaliza tudo. O que separa o caso fácil do difícil é se o atleta marcou voltas.

Quando marcou, `start_index` e `end_index` entregam a segmentação pronta e a comparação é exata. Quando não marcou, a prescrição funciona como gabarito de busca, o que é bem mais fácil que detectar intervalos do zero — e o motor devolve confiança junto com o resultado, para nunca transformar dúvida do software em cobrança sobre o atleta.
