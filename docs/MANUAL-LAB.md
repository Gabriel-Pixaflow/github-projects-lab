# Manual do Laboratório — GitHub Projects + GitHub Actions

> Documento vivo, atualizado a cada etapa do laboratório. É a base para o manual de uso que será proposto para o projeto da empresa.
>
> - Plano e decisões gerais: [`PLANO-LAB.md`](../PLANO-LAB.md)
> - Dados fictícios de roadmap: [`ROADMAP.md`](../ROADMAP.md)
> - Repositório: <https://github.com/Gabriel-Pixaflow/github-projects-lab>
> - Project: <https://github.com/users/Gabriel-Pixaflow/projects/1>

## Sumário

1. [Visão geral](#1-visão-geral)
2. [Conceitos](#2-conceitos)
3. [Etapa 0 — Pré-requisitos](#3-etapa-0--pré-requisitos)
4. [Etapa 1 — Repositório e proteção da main](#4-etapa-1--repositório-e-proteção-da-main)
5. [Etapa 2 — Criar e configurar o Project](#5-etapa-2--criar-e-configurar-o-project)
6. [Etapa 3 — Labels, Milestones, Issues e ciclo manual](#6-etapa-3--labels-milestones-issues-e-ciclo-manual)
7. [Vínculo Issue ↔ Pull Request](#7-vínculo-issue--pull-request)
8. [Workflows nativos do Project](#8-workflows-nativos-do-project)
9. [Referência rápida de comandos `gh`](#9-referência-rápida-de-comandos-gh)
10. [Descobertas e pegadinhas](#10-descobertas-e-pegadinhas)
11. [Decisões](#11-decisões)
12. [Próximas etapas](#12-próximas-etapas)

---

## 1. Visão geral

### Problema

O roadmap da empresa vive em um arquivo markdown no repositório. Ele precisa ser atualizado manualmente via Pull Request e fica desatualizado com facilidade.

### Proposta

- **GitHub Projects (V2)** como Kanban.
- **Issues** como card principal de cada funcionalidade.
- **Pull Requests** vinculados às Issues, sem gerar cards duplicados.
- **Workflows nativos + GitHub Actions** movendo os cards automaticamente conforme o trabalho acontece.

### Fluxo do Kanban

```
Sugestão → Backlog → To Do → In Progress → Draft PR → In Review → Approved → Done
```

### Quem move cada transição (estado atual do laboratório)

| Transição | Responsável | Situação |
|---|---|---|
| Issue criada → status inicial | Workflow nativo *Item added to project* | ✅ funcionando (To Do; mudará para Backlog na Etapa 4) |
| PR vinculado → In Progress | Workflow nativo *Pull request linked to issue* | ✅ funcionando (será revisto na Etapa 4) |
| Draft PR → Draft PR | GitHub Actions | ⏳ Etapa 7a |
| Ready for review → In Review | GitHub Actions | ⏳ Etapa 7b |
| Aprovado → Approved | GitHub Actions | ⏳ Etapa 7c |
| Merge → Done | GitHub fecha a Issue + nativo *Item closed* | ✅ funcionando |

---

## 2. Conceitos

### 2.1 Objetos do GitHub

| Objeto | O que é | Onde vive |
|---|---|---|
| **Project (V2)** | Quadro/planilha que agrega itens de um ou mais repositórios | Usuário ou Organization (**não** pertence a um repositório) |
| **Issue** | Tarefa ou funcionalidade. É o card principal | Repositório |
| **Draft Issue** | Rascunho que existe **só no Project**. Sem número, sem repositório, sem notificações. Pode ser convertido em Issue | Project |
| **Pull Request** | Proposta de mudança de código, vinculada à Issue | Repositório |
| **Label** | Etiqueta livre. Uma Issue pode ter várias | Repositório |
| **Milestone** | Entrega com data. Uma Issue tem no máximo um. Mostra barra de progresso | Repositório |
| **Iteration** | Campo do Project com ciclos de tempo (sprints) | Project |

**Draft Issue × Issue**

| | Draft Issue | Issue |
|---|---|---|
| Número (#N) | ❌ | ✅ |
| Repositório | ❌ | ✅ |
| Labels / Milestone / Assignee no repo | ❌ | ✅ |
| Pode ser fechada por PR (`Closes #N`) | ❌ | ✅ |
| Campos do Project (Status, Priority...) | ✅ | ✅ |

Por isso a automação de PR só funciona com **Issues**. Draft Issues servem para ideias (coluna *Sugestão*).

### 2.2 Dois tipos de campo no Project

| Tipo | Exemplos | Onde o dado vive | Consequência |
|---|---|---|---|
| **Campos de sistema** | Title, Assignees, Labels, Milestone, Linked pull requests, Reviewers, Repository | Na própria Issue/PR | Editar no Project altera a Issue |
| **Campos do Project** | Status, Priority, Size, Iteration | Só no Project | A Issue não sabe em que coluna está; só a API de Projects altera |

Uma mesma Issue em dois Projects tem **um Status diferente em cada um**.

### 2.3 Hierarquia de IDs (Projects V2 / GraphQL)

```
Project        PVT_...     "qual quadro"         ex.: PVT_kwHOFGJHpc4Bm9yu
 ├─ Field      PVTSSF_...  "qual campo"          ex.: Status = PVTSSF_lAHOFGJHpc4Bm9yuzhlkj_o
 │   └─ Option 8 hex       "qual valor"          ex.: Draft PR = 067f21de
 └─ Item       PVTI_...    "qual card"           ex.: Issue #3 neste Project = PVTI_lAHOFGJHpc4Bm9yuzhAoNeQ
                └─ content → a Issue/PR/Draft Issue de verdade
```

- **Item ≠ Issue.** O Item é a Issue **dentro de um Project específico**. Para mover um card, é preciso descobrir o Item ID daquela Issue naquele Project.
- Prefixos: `PVTSSF` = single select field, `PVTIF` = iteration field, `PVTF` = campo comum.
- Os IDs acima são **exemplos deste laboratório**. Automações não devem tê-los fixos no código: devem resolver por nome em tempo de execução (ver [Decisões](#11-decisões)).

---

## 3. Etapa 0 — Pré-requisitos

### Ferramentas

```bash
git --version       # testado com 2.56
gh --version        # testado com 2.102
gh auth status      # conta e escopos do token
```

### Escopo `project` no `gh`

Sem ele, os comandos `gh project ...` falham com `missing required scopes [read:project]`.

```bash
gh auth refresh -h github.com -s project
```

- Precisa de **terminal interativo**: mostra um código de uso único e abre o navegador. Não funciona via `!` no Claude Code; use um terminal separado.
- O navegador precisa estar logado na **conta certa**.
- Para conferir os escopos reais do token:

  ```bash
  gh api -i user | grep -i x-oauth-scopes
  ```

### Decisões do laboratório

| Item | Escolha | Motivo |
|---|---|---|
| Dono | Conta pessoal | Simplicidade no lab (ver diferenças para Organization em `PLANO-LAB.md` §6) |
| Visibilidade do repo | Público | No plano Free, rulesets/branch protection só existem em repositórios públicos |
| Visibilidade do Project | Público | Demonstração sem precisar conceder acesso |

---

## 4. Etapa 1 — Repositório e proteção da main

### 4.1 Repositório local

```bash
git init -b main
# identidade só deste repositório (não altera o git global)
git config user.name  "Gabriel-Pixaflow"
git config user.email "341985189+Gabriel-Pixaflow@users.noreply.github.com"
git add . && git commit -m "chore: estrutura inicial"
```

**Por que o e-mail noreply?** Em repositório público, o e-mail dos commits fica visível para qualquer pessoa. O formato `ID+login@users.noreply.github.com` esconde o e-mail real e ainda atribui o commit à conta certa. Para descobrir o seu:

```bash
gh api user --jq '"\(.id)+\(.login)@users.noreply.github.com"'
```

### 4.2 Repositório remoto

```bash
gh repo create github-projects-lab --public --source . --remote origin --description "..."
git remote set-url origin https://github.com/Gabriel-Pixaflow/github-projects-lab.git
gh auth setup-git          # gh vira o credential helper do git (HTTPS sem senha)
git push -u origin main    # a primeira branch enviada vira a default branch
```

> O `set-url` para HTTPS foi necessário porque a chave SSH da máquina pertencia a outra conta (`Permission denied (publickey)`).

### 4.3 Ruleset na `main`

*Settings → Rules → Rulesets*, ou via API:

```bash
gh api -X POST repos/OWNER/REPO/rulesets --input ruleset.json
```

```json
{
  "name": "Proteção da main",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "bypass_actors": [
    { "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "pull_request" }
  ],
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "pull_request", "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false
    } }
  ]
}
```

| Regra | Efeito |
|---|---|
| `pull_request` com 1 aprovação | Merge bloqueado sem aprovação. Também faz o `reviewDecision` do PR ter valor (sem a regra, vem `null`) |
| `dismiss_stale_reviews_on_push` | Commit novo após a aprovação a descarta, gerando o evento `pull_request_review: dismissed` |
| `non_fast_forward` | Bloqueia `push --force` |
| `deletion` | Impede apagar a `main` |
| Bypass admin, modo `pull_request` | Admin não faz push direto, mas pode forçar o merge **de um PR** |

### Como testar

```bash
git commit --allow-empty -m "teste: push direto"
git push                      # esperado: GH013 Repository rule violations
git reset --hard origin/main  # desfaz o commit de teste
```

---

## 5. Etapa 2 — Criar e configurar o Project

### 5.1 Criar, tornar público e vincular ao repositório

```bash
gh project create --owner Gabriel-Pixaflow --title "Lab Kanban"
gh project edit 1 --owner Gabriel-Pixaflow --visibility PUBLIC --description "..."
gh project link 1 --owner Gabriel-Pixaflow --repo Gabriel-Pixaflow/github-projects-lab
```

- O **vínculo** faz o Project aparecer na aba *Projects* do repositório.
- O vínculo **não** adiciona Issues automaticamente. Isso é papel do workflow *Auto-add*.

### 5.2 Opções do Status (8 colunas)

Via GraphQL, reaproveitando os IDs das opções padrão (`Todo`, `In Progress`, `Done`):

```graphql
mutation($field: ID!) {
  updateProjectV2Field(input: {
    fieldId: $field
    singleSelectOptions: [
      { name: "Sugestão",    color: PINK,   description: "Ideia ainda não avaliada/priorizada" }
      { name: "Backlog",     color: GRAY,   description: "Aceita, aguardando priorização" }
      { id: "<id do Todo>",        name: "To Do",       color: BLUE,   description: "Priorizada, pronta para começar" }
      { id: "<id do In Progress>", name: "In Progress", color: YELLOW, description: "Em desenvolvimento" }
      { name: "Draft PR",    color: ORANGE, description: "PR em rascunho vinculado" }
      { name: "In Review",   color: PURPLE, description: "PR pronto, aguardando review" }
      { name: "Approved",    color: GREEN,  description: "PR aprovado, aguardando merge" }
      { id: "<id do Done>",        name: "Done",        color: GREEN,  description: "PR mergeado / Issue concluída" }
    ]
  }) { projectV2Field { ... on ProjectV2SingleSelectField { options { id name } } } }
}
```

> **Por que passar o `id`?** Sem ele, o GitHub cria opções novas com IDs novos. Os cards perdem o status e os workflows nativos (que apontam para o ID antigo do *Done*) param de funcionar.

Para descobrir os IDs atuais:

```bash
gh project field-list 1 --owner Gabriel-Pixaflow
```

### 5.3 Campos personalizados

| Campo | Tipo | Valores |
|---|---|---|
| Priority | Single select | 🔴 Alta, 🟡 Média, 🟢 Baixa |
| Size | Single select | P, M, G |
| Iteration | Iteration | Sprint 1 a 6, 14 dias, a partir de 2026-10-12 |

- Criados com `createProjectV2Field` (GraphQL), porque o `gh project field-create` não aceita cores, descrições nem o tipo Iteration.
- Criar um campo Iteration com `iterations: []` gera um campo **sem sprints**. Elas precisam ser informadas explicitamente com `updateProjectV2Field`, passando `iterationConfiguration.iterations: [{ title, startDate, duration }]`.

### 5.4 Views

| View | Layout | Filtro | Para quê |
|---|---|---|---|
| Kanban | Board | `-is:pr` | Quadro principal; exclui PRs e mantém Draft Issues |
| Sprint atual | Board | `-is:pr iteration:@current` | Só o sprint corrente (`@current` é dinâmico) |
| Tabela | Table | — | Visão completa e edição em massa |
| Roadmap | Roadmap | `-is:pr` | Linha do tempo; substituto visual do ROADMAP.md |

- A API (`createProjectV2View` / `updateProjectV2View`) aceita apenas nome, layout, filtro e campos visíveis.
- `createProjectV2View` **não aceita `filter`**: é preciso criar e depois aplicar o filtro com `updateProjectV2View`.
- Views **Roadmap não aceitam `visibleFieldIds`** (erro `Roadmap views do not support visible fields`).

**Ajustes somente pela interface** (seta ▾ ao lado do nome da view; depois clicar em **Save**):

1. Kanban → *Column by*: **Status**.
2. Roadmap → *Date fields*: **Iteration**; *Zoom*: **Quarter**; *Markers*: **Iteration** e **Milestones**.
3. Tabela → *Group by*: **Status**; *Sort by*: **Priority**.

> Alterações em views pela interface só valem depois de clicar em **Save**. Sem isso, ficam só na sua sessão.

### Como testar

1. Repositório → aba *Projects* → "Lab Kanban" aparece.
2. Kanban mostra as 8 colunas e mais *No Status*.
3. Criar um card, definir Iteration = sprint atual → aparece em *Sprint atual*.
   - Item **sem** Iteration não aparece nessa view (comportamento esperado do filtro).

---

## 6. Etapa 3 — Labels, Milestones, Issues e ciclo manual

### 6.1 Labels e Milestones

```bash
gh label create "area:auth" -R OWNER/REPO --color 1D76DB --description "Autenticação e contas"
# ... area:agenda, area:pacientes, area:notificacoes, area:dashboard, area:pagamentos, tech-debt

# gh não tem comando de milestone → REST API
gh api -X POST repos/OWNER/REPO/milestones \
  -f title="Q4 2026 — MVP" -f due_on="2026-12-31T23:59:59Z" -f description="..."
```

| Origem no ROADMAP.md | Vira |
|---|---|
| Trimestre (`## Q4 2026 — MVP`) | **Milestone** |
| Área (`### Autenticação`) | **Label** `area:*` |
| Prioridade (🔴🟡🟢) | Campo **Priority** |
| Tamanho (`P/M/G`) | Campo **Size** |
| Sprint | Campo **Iteration** |

### 6.2 Criar Issues já no Project e preencher campos

```bash
gh issue create -R OWNER/REPO --title "..." --label area:auth --milestone "Q4 2026 — MVP" \
  --assignee @me --project "Lab Kanban" --body "..."
```

`--project` adiciona a Issue ao Project (cria o **Item**), mas os campos do Project ficam vazios ou recebem o valor do workflow *Item added*.

**Preencher campos por nome** (mais legível; o `gh` resolve os IDs):

```bash
gh project item-edit 1 --owner OWNER --url https://github.com/OWNER/REPO/issues/4 --field Status   --value "Backlog"
gh project item-edit 1 --owner OWNER --url https://github.com/OWNER/REPO/issues/4 --field Priority --value "🔴 Alta"
```

**Preencher campos por ID** (obrigatório para Iteration; é o caminho que automações usam):

```bash
gh project item-edit --project-id PVT_... --id PVTI_... --field-id PVTIF_... --iteration-id 82ee3b9e
```

- Um campo por chamada. A API GraphQL tem a mesma regra: um `updateProjectV2ItemFieldValue` por campo.

### 6.3 Draft Issue

```bash
gh project item-create 1 --owner OWNER --title "..." --body "..."
```

Cria um item **só no Project**, sem número e sem repositório. Usado para a coluna *Sugestão*.

### 6.4 Ciclo manual completo (Issue #3 → PR #5)

```bash
# 1. branch vinculada à Issue (aparece em "Development" da Issue)
gh issue develop 3 --name docs/3-registro-progresso --base main --checkout

# 2. trabalho + commit (Refs apenas menciona; quem fecha é o PR)
git commit -m "docs: ...

Refs #3"
git push

# 3. Draft PR com palavra-chave de fechamento na descrição
gh pr create --draft --base main --title "..." --body "... Closes #3"

# 4. pronto para review
gh pr ready 5

# 5. merge (aqui com bypass de admin, pois não havia segundo revisor)
#    Interface: marcar "Merge without waiting for requirements to be met" → Squash and merge
gh pr merge 5 --squash --admin --delete-branch

# 6. atualizar o local (squash cria um commit novo → branch -d recusa; conferir e usar -D)
git checkout main && git pull --prune
git diff main docs/3-registro-progresso --stat   # vazio = idêntico
git branch -D docs/3-registro-progresso
```

**O que aconteceu com o card #3 em cada passo**

| # | Ação | Evento no GitHub | Card #3 | Quem moveu |
|---|---|---|---|---|
| 1 | Issue criada com `--project` | `issues: opened` | **To Do** | nativo *Item added to project* |
| 2 | Branch vinculada | `create` | To Do | ninguém |
| 3 | Commit + push | `push` | To Do | ninguém |
| 4 | Draft PR com `Closes #3` | `pull_request: opened` | **In Progress** | nativo *Pull request linked to issue* |
| 5 | Ready for review | `pull_request: ready_for_review` | In Progress | ❌ lacuna → Actions |
| 6 | Aprovação | `pull_request_review: submitted` | — | ❌ lacuna → Actions |
| 7 | Merge | `pull_request: closed` (merged) | **Done** | GitHub fecha a Issue → nativo *Item closed* |

**Conclusão:** os workflows nativos cobrem o **início** e o **fim** do fluxo. O **meio** (Draft PR, In Review, Approved) exige GitHub Actions.

---

## 7. Vínculo Issue ↔ Pull Request

| Onde / como | Vincula (aparece em *Development*) | Fecha a Issue no merge |
|---|---|---|
| `Closes`/`Fixes`/`Resolves #N` na **descrição** do PR, base = **branch padrão** | ✅ | ✅ |
| Mesma palavra-chave, mas PR com base em **outra branch** | ❌ | ❌ |
| Vínculo manual pela barra lateral *Development* | ✅ | ✅ |
| Palavra-chave na **mensagem de commit** | ❌ | ✅ quando o commit chega à branch padrão |
| `Refs #N`, `#N` ou palavra-chave em **comentário** | ❌ (só menção na timeline) | ❌ |

- Palavras aceitas: `close/closes/closed`, `fix/fixes/fixed`, `resolve/resolves/resolved`.
- Várias Issues: `Closes #3, closes #4`. A palavra-chave deve se repetir para cada uma.
- Outro repositório: `Closes owner/repo#3`.

### Descobrir o vínculo pela API (sem regex)

```bash
# PR → Issues que ele fecha (keyword + vínculo manual)
gh pr view 5 --json closingIssuesReferences

# Issue → PRs que a fecham
gh api graphql -f query='{ repository(owner:"OWNER", name:"REPO"){ issue(number:3){
  closedByPullRequestsReferences(first:5, includeClosedPrs:true){ nodes{ number isDraft state } } } } }'

# Issue → branches vinculadas (criadas por "gh issue develop" / "Create a branch")
gh api graphql -f query='{ repository(owner:"OWNER", name:"REPO"){ issue(number:3){
  linkedBranches(first:5){ nodes{ ref{ name } } } } } }'
```

> ⚠️ O vínculo é processado **de forma assíncrona**. Logo após criar o PR, `closingIssuesReferences` pode vir **vazio**; alguns segundos depois, aparece. Automações precisam de retry.

---

## 8. Workflows nativos do Project

Menu **⋯** (canto superior direito do Project) → **Workflows**. A API permite **ler** o estado dos workflows, mas não ligá-los:

```bash
gh api graphql -f query='{ user(login:"OWNER"){ projectV2(number:1){ workflows(first:20){ nodes{ name enabled } } } } }'
```

| Workflow | Padrão em projeto novo | O que faz | Avaliação |
|---|---|---|---|
| Item added to project | 🟢 ligado → *Todo* | Item novo recebe um Status | Mudar para **Backlog** (Etapa 4) |
| Item closed | 🟢 ligado → *Done* | Issue fechada → Done | ⚠️ Inclui *not planned* (testado com a #2) |
| Auto-close issue | 🟢 ligado | Card movido para Done → fecha a Issue | ⚠️ Arrastar para Done fecha a Issue |
| Pull request linked to issue | 🟢 ligado → *In Progress* | PR vinculado → move a **Issue** | Não diferencia draft de pronto |
| Pull request merged | 🟢 ligado → *Done* | Card **do PR** → Done | Sem efeito aqui (PR não vira card) |
| Auto-add sub-issues to project | 🟢 ligado | Sub-issues entram no Project | Útil para tarefas com subtarefas |
| Auto-add to project | a configurar | Adiciona Issues do repo automaticamente | Etapa 4 (filtro `is:issue`) |
| Item reopened | a configurar | Issue reaberta → Status | Etapa 4 |
| Auto-archive items | a configurar | Arquiva itens fechados após X dias | Etapa 4 |
| Code review approved / changes requested | a configurar | Movem o card **do PR** | Não servem quando o card principal é a Issue |

---

## 9. Referência rápida de comandos `gh`

| Comando | Para quê |
|---|---|
| `gh auth status` | Conta ativa e escopos |
| `gh auth refresh -h github.com -s project` | Adicionar escopo (terminal interativo) |
| `gh auth setup-git` | Usar o `gh` como credencial do git em HTTPS |
| `gh repo create NOME --public --source . --remote origin` | Criar o repo a partir da pasta local |
| `gh label create` / `gh label list` | Labels |
| `gh api -X POST repos/O/R/milestones -f title=...` | Milestones |
| `gh issue create ... --project "Nome"` | Criar Issue já no Project |
| `gh issue develop N --name BRANCH --checkout` | Branch vinculada à Issue |
| `gh issue close N --reason "not planned"` | Fechar sem concluir |
| `gh pr create --draft` / `gh pr ready N` | Draft PR / pronto para review |
| `gh pr view N --json isDraft,reviewDecision,closingIssuesReferences` | Estado do PR para automação |
| `gh pr merge N --squash --delete-branch` | Merge |
| `gh project create` / `edit` / `link` | Gerenciar o Project |
| `gh project field-list N --owner O` | Campos e IDs das opções |
| `gh project item-list N --owner O --format json` | Itens e seus campos |
| `gh project item-create N --owner O --title ...` | Draft Issue |
| `gh project item-edit ... --field X --value Y` | Alterar campo por nome |
| `gh api graphql -f query='...'` | Qualquer operação GraphQL |

**Dicas**

- Use `--jq` do próprio `gh` para filtrar JSON (não depende do `jq` instalado).
- Introspecção do schema GraphQL, para descobrir o que a API suporta:

  ```bash
  gh api graphql -f query='{ __type(name:"UpdateProjectV2FieldInput"){ inputFields{ name } } }'
  ```

  A API limita cada campo de introspecção a 2 usos por requisição (`INTROSPECTION_LIMIT_EXCEEDED`).

---

## 10. Descobertas e pegadinhas

| # | Descoberta | Impacto |
|---|---|---|
| 1 | `gh auth refresh` exige terminal interativo | Rodar fora do Claude Code |
| 2 | E-mail do git global fica público em repo público | Usar noreply com config local |
| 3 | `GITHUB_TOKEN` não acessa Projects V2 | Actions precisará de PAT classic (conta pessoal) ou GitHub App (org) |
| 4 | Editar opções do Status sem `id` recria as opções | Sempre reenviar os IDs existentes |
| 5 | Iteration criado via API nasce sem sprints | Informar `iterations` explicitamente |
| 6 | Roadmap view não aceita campos visíveis; `createProjectV2View` não aceita filtro | Criar e depois atualizar |
| 7 | Ajustes de view na interface exigem **Save** | Sem isso, não persistem |
| 8 | Projeto novo já vem com 6 workflows nativos ligados | Revisar antes de testar automações |
| 9 | *Item closed* manda *not planned* para Done | Decisão pendente |
| 10 | *Pull request linked to issue* não diferencia draft | Decisão pendente |
| 11 | `closingIssuesReferences` é preenchido de forma assíncrona | Retry na Action do evento `opened` |
| 12 | Sem regra de review obrigatório, `reviewDecision` = `null` | Ruleset é pré-requisito do status *Approved* |
| 13 | `item-list` pode não refletir um item recém-criado | Considerar consistência eventual em scripts |
| 14 | Squash merge faz `git branch -d` recusar a branch local | Conferir com `git diff` e usar `-D` |
| 15 | Claude Code (auto mode) bloqueia `gh pr merge --admin` | Merge com bypass é decisão humana |
| 16 | Autor do PR não pode aprovar o próprio PR | Segunda conta necessária para a Etapa 7c |

---

## 11. Decisões

### Tomadas

| Decisão | Escolha | Motivo |
|---|---|---|
| Card principal | Issue | PR vinculado, sem card duplicado |
| PRs no Project | Não adicionar | Views filtram `-is:pr`; auto-add usará `is:issue` |
| Estratégia de IDs | Resolver por nome em tempo de execução; apenas `PROJECT_OWNER`/`PROJECT_NUMBER` em Variables | Renomear/recriar opções não quebra a automação |
| Merge | Squash | Um commit por Issue na `main` |
| Branches | `gh issue develop` | Vínculo detectável por API (`linkedBranches`) |

### Pendentes (Etapa 4)

| # | Decisão | Opções | Recomendação |
|---|---|---|---|
| 1 | Status inicial de Issue nova | Backlog / Sugestão | Backlog |
| 2 | *Pull request linked to issue* | Reconfigurar para Draft PR / desligar e deixar a Action decidir pelo `isDraft` | Desligar |
| 3 | *Not planned* indo para Done | Aceitar / coluna "Descartado" + Action / aceitar + Auto-archive | Aceitar + Auto-archive |

---

## 12. Próximas etapas

- [ ] Etapa 4 — Workflows nativos
- [ ] Etapa 5 — GraphQL na mão
- [ ] Etapa 6 — Credencial (PAT/App) e Secrets
- [ ] Etapa 7 — Actions: Draft PR, In Review, Approved, Merge/Closed
- [ ] Etapa 8 — Casos de borda
- [ ] Etapa 9 — Detectar início de desenvolvimento
- [ ] Etapa 10 — Migração do ROADMAP.md
- [ ] Etapa 11 — Demo
