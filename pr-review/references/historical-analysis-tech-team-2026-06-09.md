---
type: analysis
title: "Analise historica de PRs tech-team e propostas de melhoria na skill pr-review"
source_agent: hermes
source_origin: "Analise de 12 PRs recentes do repo griinstitute/tech-team, sessao 2026-06-09"
created: 2026-06-09
updated: 2026-06-09
tags: [pr-review, tech-team, processo, skill-improvement]
confidence: high
signal: alto
status: raw
---

# Analise historica: PRs tech-team vs skill pr-review

## Dados brutos

Analisados 12 PRs do repo griinstitute/tech-team (#71-#86, periodo 2026-06-04 a 2026-06-09).

**Padrao unanime:**
- reviewDecision: REVIEW_REQUIRED (todos)
- reviews: [] (todos)
- mergedBy: guifav (todos)
- Nenhum reviewer atribuido, nenhum comentario de review, nenhuma aprovacao formal.

**Causas observadas nas sessoes de desenvolvimento:**
1. Codex CLI timeout sistematico (>120s, >180s, >300s) em reviews
2. Kimi API erro 401 (autenticacao expirada)
3. Claude Code timeout em reviews complexos
4. Pressao de velocidade: "merge agora, review depois" se tornou padrao implicito
5. Processo multi-model (3 reviewers obrigatorios) e irreversivelmente lento para PRs pequenos

## Falhas do processo atual (skill vs pratica)

| Aspecto | Skill propoe | Pratica real | Gap |
|---|---|---|---|
| Numero de reviewers | 3 (Claude + Codex + Kimi) | 0 | Processo impossivel de executar |
| Tempo de review | ~30-60s wall-clock (skill) | >5min por reviewer, com timeout | Expectativa irreal |
| Veredito strict | ZERO findings em TODOS | Nenhum veredito emitido | Nao aplicavel |
| Gating | Manual (humano decide) | Bypass total | Nenhuma barreira fisica |
| Escopo de review | Universal gates + 15 catalogs | Nada executado | Overwhelming |

## Propostas de melhoria para a skill

### 1. Simplificar para 1 reviewer obrigatorio + checklist leve

O processo de 3 reviewers (Claude Code + Codex + Kimi) e teoricamente rigoroso mas praticamente inviavel. Cada ferramenta falha ~30-50% das vezes (timeout, auth, capacity).

**Proposta:** 1 reviewer automatizado (preferencia: Claude Code via `claude -p` com contexto do PR) + 1 checklist manual de 5 itens para o autor preencher no PR body.

Checklist (autor preenche antes de solicitar review):
- [ ] Testes passam localmente (`pytest -q`)
- [ ] Nao ha literais estaticos/snapshot no HTML output
- [ ] Nao ha credenciais hardcoded
- [ ] Mudanca foi testada no workflow do proprio branch (se aplicavel)
- [ ] Issue referenciada no titulo (Refs #N ou Closes #N)

### 2. Adicionar secao "Quando NAO fazer review"

A skill atual nao discrimina escopo. Tudo passa pelo mesmo processo pesado.

**Proposta:** Categorias de PR com tratamento diferenciado:

| Tipo | Exemplo | Review necessario? |
|---|---|---|
| Hotfix emergencial | Correcao de bug em producao | Post-merge review (dentro de 24h) |
| Trivial (<10 linhas) | Fix de typo, rename de variavel | Self-review do autor + checklist |
| Refactor mecanico | Extracao de funcao, renomeacao | 1 reviewer, foco em regressao |
| Feature nova | Nova funcionalidade, novo modulo | Review completo (1 reviewer + checklist) |
| Mudanca de infra | Workflow, deploy, secrets | Review completo + teste em staging |

### 3. Criar fallback quando ferramentas falham

O historico mostra que quando Codex/Kimi/Claude falham, o PR e mergeado sem review. Isso e pior do que um review manual rapido.

**Proposta:** Quando o reviewer automatizado falhar (timeout, auth error), o agente deve:
1. Tentar 1 retry apos 20s
2. Se falhar novamente, executar um review manual leve (diff estatistico + 5 perguntas de checklist)
3. Documentar no PR: "Review automatizado indisponivel (Codex timeout). Review manual executado."
4. NUNCA mergear sem nenhum tipo de review

### 4. Adicionar gating no GitHub (branch protection)

O fato de que todos os PRs tem `reviewDecision: REVIEW_REQUIRED` mas foram mergeados mesmo assim indica que nao ha branch protection ativa.

**Proposta:** A skill deve incluir instrucoes para configurar branch protection:
- Require 1 approving review antes de merge
- Require status checks to pass (pytest, build)
- Restrict pushes to main (force push, delete branch)
- Include administrators (ate o dono do repo precisa passar pelo processo)

### 5. Integrar review com GitHub Actions

Em vez de depender de agentes externos (Codex, Kimi, Claude CLI), o review pode ser parte do CI.

**Proposta:** Adicionar um workflow de GitHub Actions que:
1. Roda `pytest` no PR
2. Roda `flake8` / `black --check`
3. Verifica literais estaticos no HTML (smoke test)
4. Comenta no PR com o resultado
5. Bloqueia merge se falhar

Isso elimina a dependencia de ferramentas externas instaveis.

### 6. Adicionar template de PR

Muitos PRs do historico tem bodies muito curtos ou sem contexto suficiente para review.

**Proposta:** Template de PR que inclui:
```
## O que
(breve descricao da mudanca)

## Por que
(motivacao / issue relacionada)

## Como testar
(passos para validar)

## Checklist do autor
- [ ] Testes passam
- [ ] Nao ha literais estaticos
- [ ] Nao ha credenciais hardcoded
- [ ] Issue referenciada
```

### 7. Documentar "Review de excecao" (quando tudo falha)

O historico mostra incidentes onde PRs foram mergeados apesar de findings (PR #75/#76 revertido depois). A skill precisa de um protocolo para situacoes de excecao.

**Proposta:** Protocolo de merge de excecao:
1. PR e marcado com label `exception-merge`
2. Documentar no body: razao da excecao, risco aceito, plano de mitigacao
3. Criar issue follow-up para revisao pos-merge
4. Notificar no canal #engineering (ou equivalente)

## Conclusao

A skill pr-review atual e teoricamente excelente mas operacionalmente inviavel para o ritmo do tech-team. As melhorias propostas visam:
1. **Reduzir atrito:** 1 reviewer em vez de 3
2. **Aumentar confiabilidade:** fallback manual em vez de zero review
3. **Automatizar gating:** GitHub Actions + branch protection
4. **Contextualizar por escopo:** nem todo PR precisa do mesmo rigor
5. **Documentar excecoes:** quando tudo falha, ha um protocolo

A skill deve ser reescrita como um processo que FUNCIONA 100% das vezes, mesmo que com menos rigor teorico, em vez de um processo que FUNCIONA 0% das vezes por ser irrealizavel.
