# Pulsar — Análise Pré-Construção

**Situação:** Rascunho para decisão
**Data:** 2026-08-24 (revisado em 2026-08-25)
**Escopo:** Validar as premissas do `spec.txt` antes de qualquer código ser escrito.
**Idioma:** Português. Termos escritos por extenso, sem siglas.

---

## Resumo executivo

A visão do produto é coerente e o problema é real. **O plano de execução não era viável como estava escrito**, por um motivo:

> O produto mínimo viável indica o Strava como única fonte externa de dados. A política atual da API do Strava proíbe — de forma explícita e por escrito — as quatro coisas para as quais o produto mínimo viável precisa do Strava.

Isso não é um obstáculo técnico que se contorna com engenharia. É contratual, e construir sobre isso colocaria o produto em descumprimento desde o primeiro dia.

A boa notícia: a *arquitetura* que a especificação já exige — núcleo independente de fonte — é exatamente a resposta certa. A correção é mudar qual fonte chega primeiro, não redesenhar o produto.

| # | Constatação | Gravidade | Impede construir? |
|---|---|---|---|
| 1 | A política da API do Strava proíbe o uso central que o produto mínimo viável faz do Strava | 🔴 Crítica | Contratual — ver nota de situação |
| 2 | O programa de desenvolvedores da Garmin está fechado para novos inscritos | 🟠 Alta | Restringe o plano B |
| 3 | *(não é um problema — é a estratégia de dados que funciona)* | — | — |
| 4 | Dados de saúde sob a Lei Geral de Proteção de Dados são dados *sensíveis* (dor, sono, fadiga) | 🟠 Alta | Impede a entrada de atletas, não o início |
| 5 | A comparação entre planejado e realizado não está especificada e é a parte difícil | 🟡 Média | Não |
| 6 | "Carga", "aderência" e "evolução" não estão definidas | 🟡 Média | Não |
| 7 | `Session` e `Assessoria` são usadas, mas nunca modeladas | 🟡 Média | Impede o esquema de dados |
| 8 | O produto mínimo viável não é mínimo (17 itens) | 🟡 Média | Não |
| 9 | A carga do histórico não cabe no limite de requisições do Strava | 🟠 Alta | Molda o desenho da entrada de atletas |

> **Nota de situação (2026-08-25).** Decisões de escopo tomadas nesta data:
>
> 1. **Risco de retenção aceito.** Os dados do Strava ficam armazenados no banco do próprio Pulsar. Constatação 1 → risco aceito, não é mais um impedimento aberto.
> 2. **Entrada de dados dividida em duas vias.** O histórico é carregado uma única vez a partir da exportação em massa do Strava; a API cuida apenas da sincronização contínua. Constatação 9 → encerrada, mas isso traz o **leitor de arquivos para dentro do escopo do produto mínimo viável**.
> 3. **Dados subjetivos adiados.** Percepção subjetiva de esforço, sono, fadiga e dor são escopo do projeto final. O produto mínimo viável usa apenas dados do Strava. Isso rebaixa a Constatação 4 e **reabre a decisão sobre a métrica de carga (D4)** — ver as notas em cada uma.
> 4. **A API é a fonte principal, não o arquivo.** A importação de arquivo existe e é suportada, mas o caminho normal dos dados é a API. Ver a Constatação 9.
>
> Continuam abertas: **5, 6, 7, 8**, e uma **4** reduzida.

---

## Método

As constatações 1 e 2 foram verificadas contra fontes primárias em 2026-08-24, não de memória. As cláusulas estão citadas literalmente abaixo. O resto vem da leitura do `spec.txt` contra ele mesmo.

**Confiança:** Alta nas constatações 1 e 2 (fontes primárias, citadas). Média na 4 (leitura jurídica — precisa da chancela de um advogado, não da minha). Alta nas 5 a 9 (são propriedades do próprio texto da especificação, ou aritmética verificada).

---

## Constatação 1 — O Strava não pode ser a espinha dorsal dos dados 🔴

### O que a especificação exige do Strava

O produto mínimo viável (`08-mvp.md`) exige, tendo o Strava como fonte única:

1. Um **treinador vendo as atividades de um atleta** (`07`, `12`)
2. **Comparação entre atletas** dentro de um grupo (`04`, `07`)
3. **Armazenamento histórico permanente** — a especificação diz que os dados "não devem ser descartados" (`10`)
4. **Aprendizado de máquina e inteligência artificial** sobre esse histórico (`01`, `09`, `13`)

### O que a política da API do Strava diz

Citado de <https://www.strava.com/legal/api_policy> (versão 2026), verificado em 2026-08-24. As cláusulas estão no original em inglês, com tradução livre logo abaixo de cada uma:

> **§2.3** — "Strava Data provided by a specific Strava user may be displayed or disclosed in your Developer Application only to that user."
>
> *Dados do Strava fornecidos por um usuário específico só podem ser exibidos ou divulgados na sua aplicação para aquele mesmo usuário.*

> **§6.1** — "You may not display or disclose Strava Data related to other users, even if such data is publicly viewable."
>
> *Você não pode exibir ou divulgar dados do Strava relativos a outros usuários, ainda que esses dados sejam publicamente visíveis.*

> **§6.2** — "You may not retain Strava Data in your cache for longer than seven (7) days."
>
> *Você não pode reter dados do Strava em cache por mais de sete dias.*

> **§5.5** — "You may not retain Strava Data in any Persistent Index."
>
> *Você não pode reter dados do Strava em nenhum índice persistente.*

> **§5.3** — "You may not use the Strava API Materials or Strava Data, directly or indirectly, in connection with the development, training, evaluation, or operation of any AI Application."
>
> *Você não pode usar os materiais da API nem os dados do Strava, direta ou indiretamente, no desenvolvimento, treinamento, avaliação ou operação de qualquer aplicação de inteligência artificial.* A definição se estende a "treinamento, ajuste fino, aprendizado por reforço, alinhamento, fundamentação, avaliação, comparação de desempenho, geração de vetores de representação e geração aumentada por recuperação".

> **§7.4** — o desenvolvedor deve "promptly and permanently delete … all Strava Data and all Personal Data derived from Strava Data" no prazo de trinta dias após o usuário revogar o acesso.
>
> *Apagar pronta e permanentemente todos os dados do Strava e todos os dados pessoais derivados deles.*

> **§5.16** — proíbe operar "any MCP Server, agent-mediated interface, or analogous mechanism that exposes the Strava API Materials."
>
> *Qualquer servidor de contexto para modelos, interface mediada por agente ou mecanismo análogo que exponha os materiais da API do Strava.*

### A colisão, item por item

| Exigência da especificação | Cláusula do Strava | Veredito |
|---|---|---|
| Treinador vê as atividades do atleta | §2.3, §6.1 | ❌ Proibido |
| Comparação entre atletas de um grupo | §6.1 | ❌ Proibido |
| Histórico permanente para análise longitudinal | §6.2 (cache de 7 dias), §5.5 | ❌ Proibido |
| Aprendizado de máquina, visão computacional, detecção de padrões | §5.3 | ❌ Proibido |
| Atleta vê os próprios dados importados | §2.3 | ✅ Permitido |

**Quatro das cinco capacidades centrais são proibidas. A única que sobrevive é exatamente a que o próprio aplicativo do Strava já faz.**

O limite de cache de sete dias merece destaque: não é apenas uma regra de privacidade, é incompatível com a premissa central do produto. O documento `10-armazenamento-e-retencao.md` é inteiramente construído sobre histórico de vários anos.

### A ambiguidade sobre treinadores — não resolvida

Em novembro de 2024, depois da primeira onda de restrições, o Strava disse ao desenvolvedor do Intervals.icu que "a intenção não é proibir relações de treinamento", e o acesso de treinadores por lá continuou. O próprio comunicado de imprensa do Strava também dizia que plataformas de treinamento poderiam continuar operando.

**Porém:** o texto atual da política (2026) não contém nenhuma exceção para treinadores, equipes ou clubes. Um esclarecimento informal dado a um desenvolvedor em 2024 não é uma licença, e não sobrevive a uma reescrita da política.

> **Recomendação:** não construa sobre uma exceção verbal. Se você quer visibilidade de treinador via Strava, obtenha isso por escrito do Strava como parte de um pedido de acesso estendido — *antes* de construir, não depois.

### Níveis de acesso

Em vigor desde 1º de junho de 2026:

- **Nível padrão** — autoatendimento, sem fila. Dois patamares: 10 usuários registrados, ou 9.999. **Exige que o desenvolvedor tenha uma assinatura paga do Strava** (cerca de 11,99 dólares por mês). A cobrança desse requisito começou em 30 de junho de 2026.
- **Nível de acesso estendido** — mais de 10.000 usuários, aprovação caso a caso, sem exigência de assinatura, elegível às APIs de parceiro.
- Limites de requisição no nível padrão (reverificados em 2026-08-25 em <https://developers.strava.com/docs/rate-limits/>):
  - **Geral:** 200 requisições por 15 minutos, 2.000 por dia
  - **Leitura (fora de envio):** 100 requisições por 15 minutos, **1.000 por dia**
  - Os limites são **por aplicação, não por atleta** — todos os atletas de todas as assessorias dividem o mesmo bolso.
  - Os valores ao vivo chegam nos cabeçalhos `X-RateLimit-Limit` / `X-RateLimit-Usage` e `X-ReadRateLimit-Limit` / `X-ReadRateLimit-Usage`; cada um traz `15min,dia`. Leia esses cabeçalhos em vez de fixar números no código.
  - Aplicações promovidas recebem tetos maiores (por exemplo, 400 por 15 minutos no geral), mas isso é concedido, não é autoatendimento.
- Também já agendado: endpoints de clube e de exploração de segmentos descontinuados em 1º de setembro de 2026; nova URL base e mudanças de autenticação em 1º de junho de 2027.

A capacidade, portanto, *não* é o impedimento — uma assessoria do tamanho de um piloto cabe no nível padrão. **O impedimento são os termos da política.**

---

## Constatação 2 — O plano B óbvio está fechado no momento 🟠

**O programa de desenvolvedores do Garmin Connect está suspenso.** Não é possível criar novas contas de desenvolvedor e não há data divulgada de reabertura. A API de saúde exige, além disso, aprovação como parceiro e uma pessoa jurídica; não é autoatendimento.

Isso importa porque a Garmin era o substituto natural: o programa dela é explicitamente desenhado para "plataformas de condicionamento, treinamento, bem-estar ou acompanhamento de saúde", envia os dados completos da atividade (`.fit`, `.gpx`, `.tcx`) para uma URL de retorno assim que o relógio sincroniza, e os termos dela são feitos exatamente para este caso de uso.

**COROS e Polar** têm programas de parceria e valem avaliação, mas cada um exige inscrição e aprovação, e nenhum dos dois foi verificado aqui. Trate-os como tarefas de pesquisa, não como premissas.

---

## Constatação 3 — A estratégia de dados que funciona

O caminho que é juridicamente limpo, tecnicamente suficiente e está disponível hoje:

### Importação de arquivo iniciada pelo atleta (`.fit`, `.tcx`, `.gpx`)

O atleta exporta o próprio arquivo de atividade e envia ao Pulsar. Isso é materialmente diferente de acesso via API:

- O arquivo pertence ao atleta, não à API do Strava.
- Pela **Lei Geral de Proteção de Dados, artigo 18**, a portabilidade dos dados é um direito do atleta.
- O formato `.fit` carrega tudo o que a especificação pede — ritmo, frequência cardíaca, cadência, altimetria, voltas, e as séries por segundo necessárias para detectar intervalos.
- Sem limites de requisição, sem regra de cache de sete dias, sem teto de atletas por nível, sem risco de revogação.

⚠️ **Ressalva:** um ser humano baixando o próprio arquivo é portabilidade. *Automatizar* esse download seria burlar os termos e configuraria descumprimento. Mantenha o ser humano no meio do caminho, e peça a um advogado que confirme a postura antes do lançamento.

### Entrada manual
Necessária de qualquer forma — nem todo atleta tem relógio, e é a única maneira de capturar os dados subjetivos (percepção subjetiva de esforço, sono, fadiga, dor) que são o diferencial real do Pulsar.

### Strava, limitado ao que é permitido
O Strava ainda pode servir a visão do próprio atleta. Trate-o como uma conveniência que pode ser retirada, nunca como o sistema de registro.

### Uma API de parceiro, buscada em paralelo
Garmin quando reabrir, ou COROS e Polar. Comece os pedidos cedo — a aprovação é lenta e discricionária.

> **O lado bom:** a especificação já exige um núcleo independente de fonte (`08`, `14`). Esta constatação transforma isso de "seria bom ter" na decisão que sustenta tudo.

---

## Constatação 4 — Lei Geral de Proteção de Dados: isso é dado pessoal sensível 🟠

A especificação coleta dor, local da dor, qualidade do sono e fadiga (`09-B`). Pelo **artigo 5º, inciso II**, dado referente à saúde é *dado pessoal sensível*, o que traz obrigações mais rígidas que as de dado pessoal comum:

- Exige **consentimento específico e destacado** para finalidades definidas (artigo 11, inciso I) — aceitar um termo de uso genérico não basta.
- A finalidade precisa ser declarada de antemão; "podemos treinar aprendizado de máquina com isso mais adiante" precisa ser consentido explicitamente, não acrescentado depois.
- A política de retenção permanente descrita em `10` precisa de base legal e de uma justificativa de retenção documentada. "Guardamos tudo para sempre porque o aprendizado de máquina precisa" é uma finalidade que precisa ser informada e consentida.
- Os atletas precisam de um caminho real de exclusão, o que conflita com o desenho de histórico imutável de `10`.

Há também uma **fronteira de produto** já corretamente identificada em `11`: os alertas são apoio à decisão, **não diagnóstico médico**. Como o sistema registra dor e sua localização, essa fronteira precisa estar visível na interface e nos termos, não apenas neste documento.

### Rebaixada em 2026-08-25 — mas não a zero

Com percepção subjetiva de esforço, sono, fadiga e dor adiados para fora do produto mínimo viável, a parte aguda desta constatação some: o Pulsar deixa de registrar dor e sua localização, então a questão do dado pessoal sensível perde a maior parte da força, e a preocupação com a fronteira médica é adiada junto com os alertas que a levantaram.

O que permanece, e não deve passar batido:

- **Frequência cardíaca ainda é plausivelmente dado de saúde.** O artigo 5º, inciso II, cobre "dado referente à saúde". Se a frequência cardíaca contínua de um monitor de atividade se enquadra é algo genuinamente indefinido na prática brasileira — a Autoridade Nacional de Proteção de Dados não se pronunciou. É um caso muito mais fraco que o de dados de dor, mas não está claramente fora.
- **O consentimento continua necessário para a relação com o treinador**, independentemente de sensibilidade: o atleta está autorizando um terceiro a ver seus dados de treino.
- **A finalidade continua tendo de ser declarada de antemão.** Se o projeto final vai treinar aprendizado de máquina com esse histórico, essa finalidade precisa estar informada e consentida **no momento da coleta, dentro do produto mínimo viável** — encaixar consentimento depois, sobre dados já coletados, é o modo caro de falhar, e é exatamente o que adiar a funcionalidade convida a acontecer.
- **O caminho de exclusão** continua valendo, e continua conflitando com o histórico imutável de `10`.

> Veredito revisado: 🟡 Média para o produto mínimo viável, voltando a 🟠 Alta quando a camada subjetiva entrar. A única coisa a acertar *agora* é a finalidade declarada no texto de consentimento.

---

## Constatação 5 — Planejado contra realizado é o problema real de engenharia 🟡

O documento `08-mvp.md` lista "comparação entre treino planejado e realizado" como um item entre dezessete. Não é um item. É a funcionalidade que faz do Pulsar outra coisa que não uma planilha, e é o único item da lista com risco técnico genuíno.

Ela exige quatro coisas que a especificação nunca define:

1. **Um formato estruturado de prescrição.** "6 × 800 m a 3:45–3:55/km, 90 s de recuperação" precisa ser dado, não texto livre. Nada na especificação define isso, e tudo depois depende disso.
2. **Segmentação dos intervalos** da atividade realizada a partir das séries brutas — ou confiando nas voltas do relógio, ou detectando os esforços a partir do sinal de ritmo. Confiar nas voltas é mais fácil, e frequentemente errado.
3. **Casamento** entre os trechos prescritos e os executados, tolerando voltas perdidas, ordem trocada e treinos abandonados.
4. **Um modelo de tolerância.** 3:58 contra um alvo de 3:55 é acerto ou erro? A resposta é uma decisão de produto sem padrão técnico, e é ela que determina a métrica de aderência.

> **Encaminhamento:** os itens 1 e 4 foram resolvidos em [PRESCRIPTION.md](PRESCRIPTION.md). Os itens 2 e 3 são o motor de comparação, e continuam sendo o único risco técnico real do projeto.

---

## Constatação 6 — Métricas centrais indefinidas 🟡

"Carga de treinamento", "aderência" e "evolução" aparecem em `04`, `07`, `09` e `11` como métricas de destaque, e nunca são definidas. Não são lacunas cosméticas — cada uma tem várias formulações consagradas, exigindo entradas diferentes.

| Métrica | Opções | Consequência da escolha |
|---|---|---|
| **Carga** | percepção subjetiva de esforço × duração; índice baseado em frequência cardíaca; índices baseados em ritmo ou potência | Determina se a carga funciona sem cinta de frequência cardíaca |
| **Aderência** | sessões cumpridas ÷ prescritas; razão de volume; taxa de alvos atingidos | Depende inteiramente do modelo de tolerância da Constatação 5 |
| **Evolução** | ritmo a uma frequência cardíaca fixa; previsão de prova; melhores marcas; tendência de carga | Determina o histórico mínimo antes que algo possa ser mostrado |

### Revisada em 2026-08-25 — a recomendação baseada em esforço percebido está anulada

A recomendação anterior era usar percepção subjetiva de esforço multiplicada pela duração, e ela dependia inteiramente de essa percepção estar sendo coletada. Com a camada subjetiva adiada para fora do produto mínimo viável, **essa métrica não está disponível** e a escolha se inverte:

| Opção | Precisa de | Falha quando |
|---|---|---|
| **Índice por frequência cardíaca** | frequência cardíaca contínua durante toda a sessão | o atleta gravou pelo celular sem cinta — e aí o dado simplesmente não existe |
| **Carga baseada em ritmo** | distância, tempo, altimetria | nunca falha — todo registro tem isso |
| **Duração × zona de intensidade** | frequência cardíaca *ou* zonas de ritmo | degrada para ritmo quando a frequência cardíaca falta |

> **Recomendação revisada:** faça a carga **baseada em ritmo, com caminho de melhoria por frequência cardíaca**. Calcule a partir do ritmo ajustado pela inclinação e da duração, de modo que funcione para todo atleta, inclusive quem só usa o celular; e trate o índice por frequência cardíaca como uma estimativa melhor, usada *quando o dado permitir*. Armazene as entradas, não só o resultado, para que a fórmula possa ser revista sem reimportar nada.
>
> Qualquer que seja a escolha, registre **qual fórmula produziu cada valor** — quando a percepção subjetiva de esforço chegar no projeto final, os valores históricos não podem mudar de significado em silêncio.

---

## Constatação 7 — Lacunas no modelo de domínio 🟡

Duas entidades sustentam o desenho e nunca são definidas.

**`Session`** — o documento `10` a descreve como uma corrente: *treino planejado → atividade → dados objetivos → percepção do atleta → retorno do treinador → análise*. Isso faz da sessão a tabela de junção no coração do sistema, e levanta perguntas que a especificação não responde:

- Existe uma sessão quando um treino é prescrito mas nunca realizado? (Precisa existir — é assim que a aderência é medida.)
- E uma atividade sem prescrição — uma corrida não planejada do atleta?
- Uma prescrição pode corresponder a duas atividades, ou duas prescrições a uma?

**`Assessoria`** — é o topo da hierarquia em `07` e a fronteira do inquilino em `12`, mas `08` não oferece nenhuma maneira de criar uma. O produto mínimo viável cadastra treinadores e atletas, e nada acima deles. Toda regra de permissão de `12` depende dessa entidade existir.

Relacionado: a especificação assume um treinador por atleta, mas assessorias reais têm vários treinadores, e atletas trocam de grupo no meio da temporada. Se o vínculo com o grupo é historiado ou sobrescrito determina se comparações entre temporadas passadas serão possíveis depois — uma decisão pequena agora, uma migração cara mais tarde.

> **Encaminhamento:** a fatia mínima está resolvida em [PRESCRIPTION.md](PRESCRIPTION.md) §8. O modelo completo continua em aberto.

---

## Constatação 8 — O produto mínimo viável não é mínimo 🟡

Dezessete itens de checklist, cobrindo autenticação, múltiplos inquilinos, planejamento de treino, integração externa, dois painéis, gráficos e quatro formulários de dados subjetivos. Isso é um produto, não um teste de hipótese.

O objetivo declarado do produto mínimo viável é *validar que a plataforma consegue centralizar e analisar os dados de treino de uma assessoria*. Medido contra esse objetivo:

- **Testa a hipótese:** importação de dados, planejado contra realizado, painel individual.
- **Estrutura necessária:** autenticação, cadastro de atletas, permissões.
- **Adiável:** painéis de grupo, gráficos de evolução ao longo do tempo, periodização, gestão de provas.

Os gráficos de evolução merecem nota específica: não mostram nada de útil até o atleta ter semanas de dados. Construí-los na primeira semana significa construir uma funcionalidade que só pode ser avaliada no terceiro mês.

---

## Constatação 9 — A carga do histórico não cabe no limite de requisições 🟠

*Acrescentada em 2026-08-25, depois que o risco de retenção da Constatação 1 foi aceito e o Strava passou a ser a fonte principal.*

Com o Strava como espinha dorsal, a restrição que aperta deixa de ser a política e passa a ser **aritmética**.

### Por que uma atividade custa mais de uma requisição

O objeto resumido de `GET /athlete/activities` **não** traz os dados por segundo. A comparação entre planejado e realizado (Constatação 5) precisa das séries. Então, por atividade:

| Chamada | Entrega | Necessária para |
|---|---|---|
| `GET /athlete/activities` | 200 resumos por página | descoberta (barato, diluído) |
| `GET /activities/{id}` | voltas, `splits_metric`, detalhe completo | segmentação de intervalos |
| `GET /activities/{id}/streams` | séries por segundo de tempo, distância, frequência cardíaca, velocidade, cadência, altimetria | análise de ritmo, detecção de esforços |

São **cerca de 2 requisições de leitura por atividade** — as voltas vêm dentro da chamada de detalhe, então o endpoint separado `/laps` não é necessário.

### A aritmética

O orçamento de leitura é de **1.000 requisições por dia, compartilhadas por toda a aplicação**.

```
1.000 req/dia ÷ 2 req/atividade ≈ 500 atividades/dia  — para TODOS os atletas somados
```

Um atleta que corre 4 vezes por semana há 3 anos tem cerca de 600 atividades, cerca de 1.200 requisições — **mais que um dia inteiro do orçamento da aplicação toda, para um único atleta.**

| Tamanho da assessoria | Histórico | Atividades | Tempo de carga |
|---|---|---|---|
| 10 atletas | 2 anos | ~4.000 | ~8 dias |
| 30 atletas | 3 anos | ~18.000 | ~36 dias |
| 50 atletas | 3 anos | ~30.000 | ~60 dias |

### O que isso *não* afeta

A operação em regime é trivial: 30 atletas × cerca de 1 atividade por dia × 2 requisições = 60 requisições por dia, 6% do orçamento. **O problema é exclusivamente a carga inicial.**

### Resolvida em 2026-08-25 — entrada de dados em duas vias

**Decisão:** o histórico é carregado **uma única vez** a partir da exportação em massa do Strava; a API e os avisos automáticos cuidam apenas da atividade contínua. A Constatação 9 está encerrada — o regime permanente usa cerca de 6% do orçamento de leitura.

| Via | Cobre | Mecanismo |
|---|---|---|
| **Importação da exportação em massa** | todo o histórico, uma vez por atleta | atleta pede a exportação → envia o arquivo → o Pulsar lê |
| **API + avisos automáticos** | tudo a partir da conexão em diante | OAuth mais assinatura de notificações |

> **Esclarecimento do usuário, 2026-08-25:** a API é a via principal. A importação de arquivo existe e é suportada, mas não é por onde os dados normalmente entram. Isso importa para o desenho do leitor: ele precisa existir para a carga histórica, mas o formato interno deve ser modelado a partir do que a API entrega, não a partir do que o arquivo entrega.

**Consequência de escopo — o leitor de arquivos é trabalho do produto mínimo viável, não adiável.** Conteúdo verificado de uma exportação em massa do Strava:

- `activities/` — um arquivo por atividade, **no formato original de gravação**: `.fit`, `.gpx` ou `.tcx`, e **compactado de forma inconsistente** (nomes reais parecem `971607640.gpx`, `83514080.gpx.gz`, `1243401459.fit.gz`).
- `activities.csv` — uma linha de resumo por atividade: nome, data, tipo, distância, tempo em movimento, altimetria, frequência cardíaca, calorias.

Notas práticas:

1. **O formato segue o aparelho de gravação.** Atletas de Garmin ou COROS produzem `.fit` (rico: frequência cardíaca por segundo, cadência, voltas). Atletas que gravaram pelo aplicativo de celular do Strava produzem `.gpx` — **só o traçado, sem frequência cardíaca, sem voltas.** A mesma assessoria vai produzir os dois.
2. **As duas vias precisam convergir para um único modelo interno.** É o núcleo independente de fonte que a especificação já exige (`08`, `14`).
3. **Remoção de duplicatas contra a API.** Exportação e avisos automáticos vão entregar atividades em torno da data de conexão. Use como chave o identificador da atividade no Strava (o nome do arquivo exportado *é* o identificador).
4. A entrega da exportação leva horas e é iniciada pelo atleta — não pode ser automatizada. A entrada de atletas precisa lidar com essa espera.

---

## Decisões necessárias antes da implementação

Estas são suas, não minhas. Ordenadas por quanto depende delas.

| # | Decisão | Situação |
|---|---|---|
| ~~**D1**~~ | ~~Confirmar a estratégia de fontes de dados.~~ **Decidido em 2026-08-25:** a API do Strava é a fonte principal; os dados ficam no banco do Pulsar; o risco de retenção da Constatação 1 é aceito. | *encerrada* |
| **D2** | Buscar esclarecimento por escrito do Strava e/ou acesso estendido? Agora também é o caminho para um **aumento do limite de requisições** (Constatação 9). | aberta |
| ~~**D3**~~ | ~~Definir o formato estruturado de prescrição.~~ **Rascunhado em 2026-08-25** → [PRESCRIPTION.md](PRESCRIPTION.md). Formato fixo simples, alvos exatos, formulário plano. | *aguardando revisão* |
| ~~**D4**~~ | ~~Escolher a fórmula de carga.~~ **Decidido em 2026-08-25:** baseada em ritmo, com melhoria por frequência cardíaca quando o sensor existir → [MODEL.md](MODEL.md) §5. | *encerrada* |
| ~~**D5**~~ | ~~Definir `Session` e `Assessoria`.~~ **Decidido em 2026-08-25:** vários treinadores por assessoria, sem dono fixo; esquema completo → [MODEL.md](MODEL.md). | *encerrada* |
| ~~**D6**~~ | ~~Cortar o produto mínimo viável.~~ **Decidido em 2026-08-25:** núcleo mais painel de grupo. Fora: gráficos de evolução, periodização, provas, camada subjetiva. | *encerrada* |
| ~~**D7**~~ | ~~Postura de exclusão.~~ **Decidido em 2026-08-25:** marcar como inativo, não apagar. Ver a ressalva em [MODEL.md](MODEL.md) — não atende pedido formal de exclusão; o esquema isola dados pessoais para que isso fique barato depois. | *encerrada com ressalva* |
| ~~**D8**~~ | ~~Política de carga histórica.~~ **Decidido em 2026-08-25:** histórico por importação única da exportação em massa; API para sincronização contínua. | *encerrada* |
| **D9** | Quais formatos o leitor precisa suportar no primeiro dia — `.fit` mais `.gpx` mais `.tcx`, ou um subconjunto | aberta |

---

## Perguntas em aberto

1. **Já existe uma assessoria piloto?** O tamanho dela, e se os atletas usam Garmin ou Strava, muda materialmente as decisões.
2. **Quem vai construir isso?** Sozinho ou em equipe — muda o que "mínimo" pode significar.
3. **Existe data alvo ou compromisso externo** puxando o produto mínimo viável?
4. **O Pulsar é comercial desde o início?** Isso eleva bastante o que está em jogo nos termos do Strava.
5. **Os atletas já gravam com relógio,** ou a entrada manual vai ser o caminho dominante na prática?

---

## Fontes

Verificadas em 2026-08-24, com os limites de requisição reverificados em 2026-08-25:

- [Política da API do Strava (2026)](https://www.strava.com/legal/api_policy) — as cláusulas que vinculam
- [Acordo da API do Strava (2026)](https://www.strava.com/legal/api)
- [Limites de requisição](https://developers.strava.com/docs/rate-limits/)
- [Atualizações no acordo da API do Strava](https://press.strava.com/articles/updates-to-stravas-api-agreement) — a declaração sobre plataformas de treinamento
- [Como os dados aparecem em aplicativos de terceiros](https://support.strava.com/hc/en-us/articles/31798729397773-API-Agreement-Update-How-Data-Appears-on-3rd-Party-Apps)
- [Uma atualização no nosso programa de desenvolvedores](https://communityhub.strava.com/insider-journal-9/an-update-to-our-developer-program-13428) — níveis e datas
- [Exportação de dados e exportação em massa](https://support.strava.com/hc/en-us/articles/216918437-Exporting-your-Data-and-Bulk-Export)
- [Intervals.icu — "Strava visibility update: Coaching is ok!"](https://forum.intervals.icu/t/strava-visibility-update-coaching-is-ok/80331) — o esclarecimento de 2024
- [Programa de desenvolvedores do Garmin Connect](https://developer.garmin.com/gc-developer-program/activity-api/)
