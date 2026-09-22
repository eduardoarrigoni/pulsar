# Pulsar — Índice da Especificação

> ℹ️ **Este é o material de origem**, anterior a qualquer análise. Serve de contexto e **não reflete as decisões tomadas** —
> em vários pontos contradiz o que foi decidido depois. Comece pelo [LEIA-ME](../LEIA-ME.md).

Documentação derivada de [`../spec.txt`](spec.txt), fatiada por tema.
Nesta etapa o conteúdo foi **apenas reorganizado**, sem acréscimos ou decisões novas.

> ⚠️ **Leia antes de construir:** [ANALYSIS.md](../ANALYSIS.md) — análise pré-build (em inglês).
> Conclusão crítica: a política atual da API do Strava **proíbe** os quatro usos centrais previstos no MVP.
> Os documentos abaixo refletem a spec original; a análise indica o que precisa ser revisto.
>
> 🔧 **Referência técnica:** [OAUTH.md](../OAUTH.md) — fluxo OAuth, rotação de tokens, webhooks e obrigações de exclusão.
>
> 📐 **Formato de prescrição:** [PRESCRIPTION.md](../PRESCRIPTION.md) — estrutura do treino planejado, entidades `Session` e `Assessoria`, e o modelo de tolerância. Resolve a decisão D3.
>
> ⚙️ **Motor de comparação:** [COMPARISON.md](../COMPARISON.md) — como o planejado é casado com o realizado a partir das séries da API. A única peça com risco técnico real.
>
> 🗄️ **Modelo de dados:** [MODEL.md](../MODEL.md) — esquema completo do produto mínimo viável, 15 tabelas. Consolida as decisões D4, D5, D6 e D7.
>
> 📊 **Modelagem UML:** [UML.md](../UML.md) — modelo de casos de uso (24 casos, 5 expandidos) e diagrama de classes em três partes.

## Contexto
| # | Documento | Conteúdo |
|---|---|---|
| 01 | [Visão e Objetivo](01-visao-e-objetivo.md) | O que é o Pulsar e o que ele se propõe a fazer |
| 02 | [Problema](02-problema.md) | Fragmentação de ferramentas e dados |
| 03 | [Solução Atual](03-solucao-atual.md) | Como o treinador resolve isso hoje |
| 04 | [Melhorias Propostas](04-melhorias-propostas.md) | Gestão, prescrição, análise, relacionamento |
| 05 | [Pitch Comercial](05-pitch-comercial.md) | Como apresentar ao cliente |
| 06 | [Ganhos Operacionais](06-ganhos-operacionais.md) | Redução de tempo e trabalho manual |

## Produto
| # | Documento | Conteúdo |
|---|---|---|
| 07 | [Níveis de Visualização](07-niveis-de-visualizacao.md) | Assessoria → Grupo → Atleta → Sessão |
| 08 | [MVP](08-mvp.md) | Escopo, checklist e o que fica de fora |
| 09 | [Análises](09-analises.md) | Desempenho, percepção, futuras, biomecânica |
| 11 | [Apresentação dos Dados](11-apresentacao-dos-dados.md) | Pós-sessão, periódico e alertas |

## Dados e regras
| # | Documento | Conteúdo |
|---|---|---|
| 10 | [Armazenamento e Retenção](10-armazenamento-e-retencao.md) | Histórico de sessões, ciclo de vida, retenção |
| 12 | [Permissões e Privacidade](12-permissoes-e-privacidade.md) | Papéis: atleta, treinador, admin |

## Arquitetura e futuro
| # | Documento | Conteúdo |
|---|---|---|
| 13 | [Visão de Longo Prazo](13-visao-longo-prazo.md) | Ecossistema multimodal de inteligência esportiva |
| 14 | [Arquitetura Conceitual](14-arquitetura-conceitual.md) | Camadas: Fontes → Core → Analytics → Intelligence → Experiência |

---

## Mapa spec.txt → documentos

| Pergunta em `spec.txt` | Documento |
|---|---|
| 1. Objetivo do projeto | 01 |
| 2. Problema a resolver | 02 |
| 3. Como é resolvido hoje | 03 |
| 4. O que melhorar | 04 |
| 5. Apresentação ao cliente | 05 |
| 6. Reduzir custos/tempo/manual | 06 |
| 7. Grupo ou individual | 07 |
| 8. Expectativa do MVP | 08 |
| 9. Análises pretendidas | 09 |
| 10. Armazenamento e descarte | 10 |
| 11. Pós-sessão ou periódico | 11 |
| 12. Consulta pelo próprio indivíduo | 12 |
| 13. Visão de longo prazo | 13 |
| Visão geral do produto | 14 |

---

## Recortes transversais

**MVP (o que é para agora)** → 07, 08, 09-A, 09-B, 10, 11 (itens 1 e 2), 12
**Produto final (visão)** → 09-C, 09-D, 11 (item 3), 13, 14 (camada Intelligence)
