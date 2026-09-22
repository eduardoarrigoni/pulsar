# 14 — Arquitetura Conceitual

> Fonte: `spec.txt` — seção "Visão geral do produto"

## Camadas

```
┌─────────────────────────────────────────────────────────────┐
│ FONTES DE DADOS                                             │
│ Strava → Garmin → COROS → Polar → RPE → Sono →              │
│ Avaliações → Vídeo → Imagem                                 │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ PULSAR CORE                                                 │
│ Atleta → Treino → Sessão → Grupo → Histórico                │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ ANALYTICS                                                   │
│ Performance → Carga → Aderência → Evolução → Biomecânica    │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ INTELLIGENCE                                                │
│ Machine Learning → Visão Computacional →                    │
│ Detecção de padrões → Recomendações                         │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ EXPERIÊNCIA                                                 │
│ Treinador → Dashboard → Alertas → Feedback → Atleta         │
└─────────────────────────────────────────────────────────────┘
```

## Descrição das camadas

| Camada | Conteúdo |
|---|---|
| **Fontes de dados** | Strava, Garmin, COROS, Polar, RPE, Sono, Avaliações, Vídeo, Imagem |
| **Pulsar Core** | Atleta, Treino, Sessão, Grupo, Histórico |
| **Analytics** | Performance, Carga, Aderência, Evolução, Biomecânica |
| **Intelligence** | Machine Learning, Visão Computacional, Detecção de padrões, Recomendações |
| **Experiência** | Treinador, Dashboard, Alertas, Feedback, Atleta |

## Princípio arquitetural declarado
A corrida é o **primeiro domínio de aplicação**; a arquitetura deve permitir a **expansão para outras modalidades esportivas e novas fontes de dados**.

Relacionado: a arquitetura deve ser **independente da fonte de dados** (ver [08-mvp.md](08-mvp.md)).
