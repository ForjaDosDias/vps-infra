# DevOps Conf — Esteira CI/CD (handoff para continuar)

> Documento de continuidade. Estado em **2026-06-18**. Local: `vps-infra/ci/devopsConf.md`.
> Plano original aprovado: `/root/.claude/plans/seguinte-claude-cansei-de-splendid-hinton.md`.

## Objetivo

Esteira DevOps por repositório, com gates crescentes e deploy automático na VPS (que é o ambiente de produção):

- `feature/*` → sem regra.
- `dev` → recebe merge **sem aprovação humana**, mas só com **testes verdes** (auto-merge ligado).
- `main` → PR **só a partir de `dev`** (check `guard`), **testes verdes** + **1 aprovação humana**.
- Merge na `main` → **deploy automático** na VPS via **self-hosted runner**.

## Decisões tomadas (não revisitar sem motivo)

1. **Gatilho de deploy:** self-hosted runner na VPS (pull/saída; não abre porta no UFW).
2. **Testes:** rodam no **GitHub-hosted** (ubuntu-latest). Só o **deploy** roda no self-hosted.
3. **Runner roda como root** (`RUNNER_ALLOW_RUNASROOT=1`) — tudo na VPS é root e os deploys já eram root.
4. **Runner é REPO-LEVEL (um por repo), não org-level.** O registro org-level na ForjaDosDias deixava
   os jobs presos em `queued` (o Broker do GitHub não roteava ao runner, mesmo com grupo `Default`
   aberto e labels corretos). Repo-level funcionou de imediato. → cada repo tem sua própria instância.
5. **Tamois mantém Kamal** (não migrar para docker compose).
6. **Repos privados na org free não impõem rulesets/auto-merge** → tornamos públicos (os 4 apps já são
   open-source). EstudeOAB foi tornado público (sem segredos no histórico; `.env` sempre foi gitignored).

## Arquitetura / artefatos

- **Templates de workflow:** `vps-infra/ci/templates/{ci.yml,guard-main-source.yml,deploy.yml}`.
- **Rulesets JSON:** `vps-infra/ci/rulesets/{main.json,dev.json}` (main com `strict=false` para não exigir
  atualizar a `dev` a cada release com squash).
- **Doc viva:** seção "Esteira CI/CD" no `vps-infra/CLAUDE.md`.
- **Binário do runner:** `/home/victor/actions-runner/actions-runner-linux-x64-2.335.1.tar.gz` (reusar).

### Workflows por repo (`.github/workflows/`)
- `ci.yml` — job **`test`**, em `pull_request`/`push` de `dev` e `main`, `runs-on: ubuntu-latest`.
- `guard-main-source.yml` — job **`guard`**, falha se `head_ref != dev` em PR para `main`.
- `deploy.yml` — job **`deploy`**, `runs-on: [self-hosted, vps-prod]`, em `push` na `main`.
  Faz `git config --global --add safe.directory $SRC_DIR` (runner é root, alguns `.git` são de outro
  dono) → `git fetch <token-url> main` → `git reset --hard FETCH_HEAD` → `docker compose up -d --build
  --wait`. Como faz fetch do HEAD da main, qualquer execução sobe sempre o estado atual (idempotente).
- Checks obrigatórios nos rulesets: `test` (main+dev) e `guard` (só main).

## Estado por repositório

| Repo | Visib. | Runner (dir / serviço) | Estado |
|---|---|---|---|
| **NaoEsqueci2** (ForjaDosDias) | público | `/home/victor/actions-runner` · `actions.runner.ForjaDosDias-NaoEsqueci2.vps-prod-runner` | ✅ esteira completa e validada |
| **EstudeOAB** (ForjaDosDias) | público | `/home/victor/actions-runner-estudeoab` · `actions.runner.ForjaDosDias-EstudeOAB.vps-prod-runner` | ✅ esteira completa, deploy validado |
| **Amanuense2** (ForjaDosDias) | público | — (a criar) | ⏳ pendente |
| **Tamois-Ia-Juridica** (VictorDG00) | público | — (a criar) | ⏳ pendente |

### NaoEsqueci2 — ✅ DONE (piloto)
- `deploy/dockerfile` mesclado na `main`; `dev` criada; workflows + rulesets ativos; auto-merge/auto-delete on.
- Deploy automático validado (container `naoesqueci` recriado e `healthy`).
- Fluxo validado ponta a ponta: feature→dev (auto-merge), guard barra branch≠dev, dev→main exige aprovação.
- **⚠️ PENDENTE:** **PR #16 (dev→main) está ABERTO com auto-merge LIGADO**, aguardando **1 aprovação
  humana**. Como foi aberto pela conta VictorDG00, ele não pode auto-aprovar — precisa de um sócio
  (Felipe/Pedro). Ao aprovar, funde e dispara deploy sozinho. (Sobe só o `docs/PIPELINE.md` de teste.)

### EstudeOAB — ✅ DONE
- WIP MercadoPago (Orders API) estava **vivo em produção mas nunca commitado** → commitado na `main`
  (git agora reflete a produção). Testes rodados antes (246 passando). PDFs/CSV de dados → `.gitignore`.
- Tornado público; runner repo-level; workflows; rulesets; auto-merge/auto-delete. Deploy validado.
- CI: `cd backend && npm ci && npm test` (jest **mocka o banco**, não precisa de Postgres no CI).
- Deploy: compose no próprio repo (`SRC_DIR == COMPOSE_DIR == Publicado/EstudeOAB`), container `estudeoab-backend-1`.
- **⚠️ FOLLOW-UP:** `EstudeOAB/CLAUDE.md` ainda manda "push sempre para `origin main`" — agora **bloqueado**
  pelo ruleset. Atualizar para o fluxo feature→dev→main (via PR, pois main está protegida).

## O que falta fazer

### Amanuense2 (próximo candidato)
- **Tree limpa, MAS não existe `main` no remoto.** Default é `claude/implementation-plan-challenges-XpawN`
  (o `main` local rastreia esse branch). Passo crítico: **estabelecer a `main`** a partir do código que
  está em produção (confirmar qual ref o container roda) e definir como default — provavelmente alinhar
  com o **Felipe** (dono do projeto).
- **Redeploy derruba o MCP** (o `amanuense-api` hospeda o MCP). → **agendar** a virada do deploy.
- Compose no próprio repo (`Publicado/Amanuense`), containers `amanuense` + `amanuense-api` + postgres.
- CI: `pip install -e ".[api]" && pytest`; precisa de **Postgres 16** (service container) e **tesseract-ocr**
  (`apt-get install tesseract-ocr`).
- Verificar ownership do `.git` (o `safe.directory` no deploy já cobre, mas confirmar).

### Tamois-Ia-Juridica
- **Muito WIP NÃO commitado** no checkout de produção: `Dockerfile`, `.kamal/`, `.github/` (a própria
  config de CI/Kamal!), `Gemfile`/`Gemfile.lock`, `README.md`, views, `public/*.html`, `storage/`.
  → **Revisar 1 a 1** o que commitar/descartar antes de qualquer `reset --hard` (igual fizemos no EstudeOAB).
  Confirmar se a produção roda esse WIP (provável) para não reverter código vivo.
- Deploy **via Kamal** (`kamal deploy`), registry local `localhost:5555` — NÃO usar docker compose.
  O `deploy.yml` dele é diferente do template (rodar `kamal deploy` no self-hosted).
- CI Minitest já existe no WIP (`.github/workflows/ci.yml`) — ajustar gatilhos para PRs de `dev`/`main`
  e adicionar `guard-main-source`.
- Conta **VictorDG00** (fora da org). Runner repo-level próprio.

## Receita para ligar a esteira num repo novo

```bash
# 1) Reconciliar: garantir que origin/main reflete a produção; criar dev a partir de main.
#    Revisar/commitar WIP vivo ANTES (deploy faz git reset --hard).

# 2) Runner repo-level (instância nova por repo):
DIR=/home/victor/actions-runner-<slug>; mkdir -p "$DIR"; cd "$DIR"
tar xzf /home/victor/actions-runner/actions-runner-linux-x64-2.335.1.tar.gz
export RUNNER_ALLOW_RUNASROOT=1
REG=$(gh api -X POST /repos/<org>/<repo>/actions/runners/registration-token --jq .token)
./config.sh --unattended --replace --url https://github.com/<org>/<repo> \
  --token "$REG" --name vps-prod-runner --labels vps-prod --work _work
./svc.sh install root && ./svc.sh start

# 3) Workflows: copiar templates de vps-infra/ci/templates/ e ajustar
#    (ci: stack do projeto; deploy: SRC_DIR/COMPOSE_DIR/REPO/concurrency). Commitar na main,
#    alinhar dev (git branch -f dev main), push main+dev. Registrar runner ANTES do push da main
#    (o push dispara o deploy).

# 4) Rulesets + settings (repo precisa ser público na org free):
gh api -X POST /repos/<org>/<repo>/rulesets --input vps-infra/ci/rulesets/main.json
gh api -X POST /repos/<org>/<repo>/rulesets --input vps-infra/ci/rulesets/dev.json
gh api -X PATCH /repos/<org>/<repo> -F allow_auto_merge=true -F delete_branch_on_merge=true
# (se acabou de tornar público: aguardar ~1min, "Repository has been locked" é temporário)
```

## Gotchas / aprendizados

- **Org-level runner não roteia** nesta org → use repo-level (um por repo).
- **`dubious ownership`**: runner é root, alguns `.git` são `victor:victor` → `deploy.yml` declara
  `safe.directory` (já incluído no template e nos repos feitos).
- **Repo privado + org free**: sem rulesets/auto-merge → tornar público (sem segredos no histórico) ou pagar Team.
- **`strict_required_status_checks_policy=false`** na main: evita exigir atualizar a `dev` a cada release (squash).
- Deploy é **idempotente**: sempre sobe o HEAD atual da `main` (fetch+reset), independente do commit gatilho.
- `docker compose up -d --build --wait` falha o job se o container não ficar healthy → bom sinal de deploy ruim.

## Operação do runner

```bash
# status / logs por repo
cd /home/victor/actions-runner            # NaoEsqueci2
cd /home/victor/actions-runner-estudeoab  # EstudeOAB
export RUNNER_ALLOW_RUNASROOT=1 && ./svc.sh status
# online via API:
gh api /repos/<org>/<repo>/actions/runners --jq '.runners[] | "\(.name) \(.status)"'
```

## Tarefas em aberto (resumo)

1. **NaoEsqueci2 PR #16** → pedir aprovação de um sócio (dispara deploy ao mesclar).
2. **EstudeOAB/CLAUDE.md** → atualizar para o fluxo feature→dev→main (via PR).
3. **Amanuense2** → estabelecer `main` (com Felipe), CI pytest+postgres+tesseract, agendar redeploy (MCP cai).
4. **Tamois** → revisar WIP grande, esteira com `kamal deploy`, runner repo-level.
