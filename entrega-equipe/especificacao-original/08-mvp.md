# 08 — Escopo do MVP

> Fonte: `spec.txt` — pergunta 8

## Objetivo do MVP
Validar a capacidade da plataforma de **centralizar e analisar os dados de treinamento de uma assessoria**.

**Integração externa inicial:** API do **Strava**, para importar as atividades realizadas pelos atletas.

## Escopo — checklist

### Cadastro e organização
- [ ] Cadastro de treinador
- [ ] Cadastro de atletas
- [ ] Criação e gerenciamento de grupos
- [ ] Controle de permissões

### Treinos
- [ ] Planejamento de treinos
- [ ] Distribuição individual e em grupo

### Integração de dados
- [ ] Integração com Strava
- [ ] Importação das atividades
- [ ] Armazenamento histórico

### Análise
- [ ] Comparação entre treino planejado e realizado
- [ ] Dashboard individual
- [ ] Dashboard de grupo
- [ ] Gráficos de evolução
- [ ] Métricas básicas de desempenho

### Percepção do atleta
- [ ] Registro de RPE
- [ ] Registro de sono
- [ ] Registro de fadiga
- [ ] Registro de dores

## Requisito arquitetural do MVP
A arquitetura deve ser **independente da fonte de dados**, permitindo incorporar futuramente **Garmin, COROS, Polar** e outras plataformas **sem reconstruir o núcleo do sistema**.

## Explicitamente FORA do MVP
Fazem parte da visão do produto final, não do MVP:

- Machine Learning
- Análise automática de biomecânica
- Visão computacional
- Análise automatizada de vídeo
- Treinos guiados por áudio
- Gamificação avançada
- Comunidade
- Sistema financeiro completo
