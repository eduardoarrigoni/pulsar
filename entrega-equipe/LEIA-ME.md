# Pulsar — Pacote de Análise e Decisões

**Data:** 2026-08-25
**Para:** equipe do projeto
**O que é:** tudo que foi analisado e decidido antes de escrever a primeira linha de código

---

## O produto, em um parágrafo

Pulsar é uma plataforma para assessorias de corrida. Centraliza atletas e grupos, permite ao treinador prescrever treinos de forma estruturada, importa automaticamente o que o atleta correu (via Strava), **compara o planejado com o realizado** e apresenta isso em painéis individuais e de grupo. A comparação entre prescrito e executado é o que diferencia o Pulsar de uma planilha, e é onde está o risco técnico do projeto.

---

## Ordem de leitura

| Ordem | Documento | Tempo | Para quem |
|---|---|---|---|
| 1 | **[ANALYSIS.md](ANALYSIS.md)** | ~25 min | **todos** — o que foi investigado, o que foi decidido e por quê |
| 2 | **[MODEL.md](MODEL.md)** | ~20 min | **todos** — esquema de dados, 15 tabelas, com o raciocínio de cada escolha |
| 3 | **[UML.md](UML.md)** | ~20 min | **todos** — casos de uso e diagrama de classes |
| 4 | [PRESCRIPTION.md](PRESCRIPTION.md) | ~15 min | quem for mexer em prescrição de treino |
| 5 | [COMPARISON.md](COMPARISON.md) | ~15 min | quem for construir o motor de comparação |
| 6 | [OAUTH.md](OAUTH.md) | ~15 min | quem for construir a integração com o Strava |

Os diagramas renderizados estão em **[`diagramas/`](diagramas/)**, em SVG e PNG. São os mesmos que estão embutidos no `UML.md`.

> ⚠️ **Nota de idioma:** o `OAUTH.md` está em inglês; todo o resto está em português. Foi assim que os documentos nasceram e ainda não foi traduzido.

---

## O que foi decidido

| # | Decisão | Escolha | Onde está detalhado |
|---|---|---|---|
| **D1** | Fonte de dados | API do Strava como fonte principal, com armazenamento em banco próprio. **Risco contratual conhecido e aceito** | ANALYSIS §Constatação 1 |
| **D3** | Formato de prescrição | Formato fixo simples: aquecimento, série principal, volta à calma. Alvos exatos, formulário plano | PRESCRIPTION |
| **D4** | Fórmula de carga | Baseada em ritmo, com melhoria por frequência cardíaca quando houver sensor | MODEL §5 |
| **D5** | Treinador × atleta | Vários treinadores por assessoria, sem dono fixo | MODEL §1 |
| **D6** | Corte do produto mínimo | Núcleo mais painel de grupo. **Fora:** gráficos de evolução, periodização, provas, camada subjetiva | MODEL |
| **D7** | Saída do atleta | Marcar como inativo, não apagar — com ressalva | MODEL §D7 |
| **D8** | Carga do histórico | Uma vez, pela exportação em massa do Strava; API só para sincronização contínua | ANALYSIS §Constatação 9 |

**Ainda em aberto:** D2 (pedir esclarecimento por escrito ao Strava) e D9 (quais formatos de arquivo o leitor suporta no primeiro dia). Nenhuma das duas impede começar.

---

## As cinco coisas que a equipe precisa entender antes de codar

Estão espalhadas pelos documentos, mas são as que causam retrabalho caro se forem descobertas depois.

### 1. A política da API do Strava proíbe o que estamos fazendo

Não é ambiguidade nem interpretação. As cláusulas estão citadas literalmente no `ANALYSIS.md`: visibilidade do treinador, comparação entre atletas, retenção acima de 7 dias e uso em aprendizado de máquina são todos vedados. **A decisão de seguir mesmo assim foi tomada com essa informação na mesa.** Está documentado para que ninguém descubra por acidente daqui a seis meses.

Consequência prática no código: toda tabela de dado importado carrega uma coluna `source`. Ela existe porque, se a postura mudar, é preciso saber exatamente o que veio do Strava.

### 2. A falta é um dado, e alguém precisa criá-la

Quando o treinador atribui um treino, nasce uma sessão com situação `planned`. Se o atleta correr, a atividade chega e vira `completed`. **Se o atleta não correr, não acontece nada** — o Strava não avisa sobre corrida que não existiu.

Por isso existe uma rotina agendada que marca as sessões vencidas como `missed`. Sem ela, a aderência seria calculada só sobre o que foi feito e **daria perto de 100% para todo mundo**, inclusive para quem faltou metade do mês. A métrica central do produto estaria errada, e errada para cima — o pior tipo, porque parece plausível.

### 3. Aderência nula nunca é aderência zero

Quando o motor de comparação não consegue casar o treino planejado com o realizado, ele grava **nulo**, jamais zero.

- **Zero** significa "o atleta não fez o treino" — é um julgamento sobre a pessoa.
- **Nulo** significa "o software não conseguiu comparar" — é um julgamento sobre o sistema.

Confundir os dois faz o treinador cobrar um atleta que treinou. Nas agregações do painel de grupo, `avg` ignora nulos sozinho, que é o comportamento certo. **Se alguém "corrigir" isso para `coalesce(adherence_pct, 0)` em algum momento, quebra a regra inteira.**

### 4. Vínculo de grupo é historiado, e a consulta precisa da janela de datas

Atletas trocam de grupo no meio da temporada. `group_membership` tem `joined_at` e `left_at` justamente por isso.

Toda consulta do painel de grupo precisa perguntar "quem estava neste grupo **naquela data**", nunca "quem está neste grupo hoje". Sem isso, um atleta que mudou de grupo em abril reescreve retroativamente todos os números de março. A consulta correta está escrita em `MODEL.md` §9 — vale copiar dali em vez de reescrever.

### 5. O token de renovação do Strava gira, e precisa ser regravado

A cada renovação, o Strava devolve um `refresh_token` **novo** e invalida o antigo. Se o código gravar só o `access_token`, tudo funciona por 6 horas — depois a conexão morre e o atleta tem que autorizar de novo.

Esse erro **não aparece em desenvolvimento**. Aparece semanas depois, em produção, um atleta por vez. Detalhes em `OAUTH.md` §4, junto com a corrida de renovações concorrentes, que é o segundo jeito de quebrar a mesma coisa.

---

## Onde está o risco técnico

Um lugar só: **o motor de comparação** (`COMPARISON.md`).

Quando o atleta programa o treino no relógio, a API do Strava devolve as voltas com índices que apontam direto para dentro das séries temporais, e a comparação é praticamente exata e quase gratuita. Chamamos de Caminho A.

Quando o atleta corre com o celular no bolso e não marca voltas, é preciso **encontrar** as repetições dentro do sinal de ritmo. Chamamos de Caminho B, e é a única parte do projeto que pode não atingir precisão aceitável.

A mitigação é construir o Caminho A primeiro e **medir no piloto** quantos atletas caem em cada caminho antes de investir no B. Pode ser que quase ninguém precise dele.

---

## Ordem de construção sugerida

Detalhada em `MODEL.md` §11. Resumo:

1. Assessoria, treinador, atleta, autenticação e permissão por inquilino
2. Grupos, com o vínculo historiado desde o primeiro dia
3. Conexão com o Strava (autorização, tokens, avisos automáticos)
4. Importação de atividades, séries e voltas
5. Importação do histórico pela exportação em massa
6. Prescrição, atribuição e sessão
7. Motor de comparação — Caminho A
8. Carga por ritmo
9. Painel individual
10. Painel de grupo
11. Motor de comparação — Caminho B *(só depois de medir)*
12. Carga por frequência cardíaca *(só depois de medir)*

Do item 1 ao 9 já é um produto que uma assessoria consegue usar.

---

## Conteúdo do pacote

```
entrega-equipe/
├── LEIA-ME.md                    ← este arquivo
├── ANALYSIS.md                   análise e decisões
├── MODEL.md                      modelo de dados
├── UML.md                        casos de uso e classes
├── PRESCRIPTION.md               formato de prescrição
├── COMPARISON.md                 motor de comparação
├── OAUTH.md                      integração Strava (em inglês)
├── diagramas/                    SVG e PNG dos 7 diagramas
└── especificacao-original/       spec.txt e o recorte temático original
```

A pasta `especificacao-original/` é o material de origem, anterior a qualquer análise. Serve de contexto — **não reflete as decisões tomadas**, e em vários pontos contradiz o que foi decidido depois. Leia o `ANALYSIS.md` antes.
