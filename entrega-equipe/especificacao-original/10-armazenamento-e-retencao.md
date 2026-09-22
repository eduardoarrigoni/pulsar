# 10 — Armazenamento de Sessões e Retenção de Dados

> Fonte: `spec.txt` — pergunta 10

## Regra
Cada sessão deve ser **armazenada historicamente**.

**Motivo:** permitir análises longitudinais e acompanhar a evolução do atleta ao longo de **semanas, meses e anos**.

## Ciclo de vida de uma sessão
```
Treino planejado
  → Atividade realizada
    → Dados objetivos
      → Percepção do atleta
        → Feedback do treinador
          → Análises
            → Ajustes futuros
```

No produto final, uma sessão poderá também estar relacionada a **vídeos, imagens e análises biomecânicas**.

## Retenção
- Os dados **não devem ser descartados** simplesmente após determinado período.
- O histórico é **essencial** para identificar tendências e construir modelos de análise e Machine Learning.
- O armazenamento deve respeitar políticas de **privacidade, segurança, consentimento e retenção de dados**.
