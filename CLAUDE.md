# vps-infra — Infraestrutura da VPS

Repositório central de configurações de infra e deploy de todos os projetos publicados na VPS. Criado para facilitar recuperação rápida do ambiente após atualizações do sistema (ex: `sudo apt upgrade` pode reconfiguar o nginx do sistema).

## Arquitetura

Todos os projetos rodam em containers Docker. Existem duas redes externas:
- `vps-proxy` — projetos de produção (proxy + apps)
- `coolify` — stack de monitoring (Grafana, Loki, Promtail)

```
Internet → proxy (nginx:80) ──► tamois:80       (tamois.com.br)
                            ──► estude-nginx:80  (forjadosdias.com.br, aprovadonaoab.com.br)
                            ──► amanuense:80     (felipe.forjadosdias.tech)
                            ──► naoesqueci:3001  (naoesqueci.forjadosdias.tech)

Monitoring (interno):
  promtail → loki → grafana:3001
```

Redes Docker devem existir antes de subir qualquer serviço:
```bash
docker network create vps-proxy
docker network create coolify
```

## Estrutura do Repositório

```
vps-infra/
├── proxy/              # nginx reverse proxy — porta 80 pública
│   ├── docker-compose.yml
│   └── nginx.conf
├── tamois/             # infra do Tamois IA Jurídica (Rails)
│   ├── docker-compose.yml
│   └── .env.example
├── estudeoab/          # infra do EstudeOAB (Node + React)
│   ├── docker-compose.yml
│   ├── nginx/nginx.conf
│   └── .env.example
├── amanuense/          # infra do Amanuense (Python + D3.js)
│   ├── docker-compose.yml
│   └── .env.example
├── naoesqueci/         # infra do Não Esqueci (Vite + React + Node)
│   ├── docker-compose.yml
│   └── .env.example
└── monitoring/         # Grafana + Loki + Promtail
    ├── docker-compose.yml
    ├── .env.example
    └── config/
        └── promtail.yml
```

## Como Recuperar o Ambiente

### 1. Criar a rede Docker
```bash
docker network create vps-proxy
```

### 2. Subir o proxy reverso
O docker-compose do proxy foi iniciado a partir de `/home/victor/codigo/nginx-proxy/`.
O container monta o nginx.conf desse path — copiar o arquivo do repo antes de recarregar:
```bash
cp /home/victor/Projetos/vps-infra/proxy/nginx.conf /home/victor/codigo/nginx-proxy/nginx.conf
docker exec nginx-proxy nginx -s reload
# ou, para recriar o container:
cd /home/victor/codigo/nginx-proxy/
docker compose up -d
```

### 3. Subir o Tamois
O código-fonte fica em `/home/victor/Projetos/Publicado/Tamois-Ia-Juridica/`.
```bash
cd /home/victor/Projetos/Publicado/Tamois-Ia-Juridica/
cp .env.example .env   # preencher as variáveis reais
docker compose up -d --build
```

### 4. Subir o EstudeOAB
O código-fonte fica em `/home/victor/Projetos/Publicado/EstudeOAB/`.
```bash
cd /home/victor/Projetos/Publicado/EstudeOAB/
cp .env.example .env   # preencher as variáveis reais
docker compose up -d --build
```

### 5. Subir o Amanuense
O código-fonte fica em `/home/victor/Projetos/Publicado/Amanuense/`.
```bash
cd /home/victor/Projetos/Publicado/Amanuense/
cp .env.example .env   # preencher DEEPSEEK_API_KEY
docker compose up -d --build
```

### 6. Subir o Não Esqueci
O código-fonte fica em `/home/victor/Projetos/Publicado/NaoEsqueci/` (clone de `ForjaDosDias/NaoEsqueci2`).
O Dockerfile faz build do front (Vite) + roda os testes e empacota o servidor Node.
```bash
cd /home/victor/Projetos/vps-infra/naoesqueci/
docker compose up -d --build
```

### 7. Subir o Monitoring
Os arquivos ficam em `/home/bugbrain/monitoring/` (pasta original na VPS).
```bash
cd /home/bugbrain/monitoring/
cp .env.example .env   # preencher GF_ADMIN_PASSWORD e GF_ROOT_URL
docker compose up -d
```

## Projetos

### proxy
- **Imagem:** nginx:1.27-alpine
- **Porta pública:** 80
- **Função:** roteamento de domínios para os containers de cada projeto via upstream dinâmico (resolve nomes Docker em runtime)
- **Domínios gerenciados:** tamois.com.br, forjadosdias.com.br, aprovadonaoab.com.br, felipe.forjadosdias.tech, naoesqueci.forjadosdias.tech

### tamois
- **Stack:** Ruby on Rails (Dockerfile no repo da aplicação)
- **Container:** `tamois`
- **Domínio:** tamois.com.br
- **Repo da app:** https://github.com/VictorDG00/Tamois-Ia-Juridica
- **Variáveis obrigatórias:** `DEEPSEEK_API_KEY`, `SECRET_KEY_BASE`
- **Volume persistente:** `tamois_storage` (uploads do Rails Active Storage)

### estudeoab
- **Stack:** Node.js (backend) + React/nginx (frontend) + PostgreSQL
- **Container nginx:** `estude-nginx`
- **Domínios:** forjadosdias.com.br, aprovadonaoab.com.br
- **Repo da app:** https://github.com/ForjaDosDias/EstudeOAB
- **Variáveis obrigatórias:** `JWT_SECRET`, `DEEPSEEK_API_KEY`, `RESEND_API_KEY`
- **Volume persistente:** `postgres_data`

### amanuense
- **Stack:** Python 3.11 + D3.js (multi-stage Docker: `web` + `api`)
- **Containers:** `amanuense` (frontend/nginx:80) + `amanuense-api` (FastAPI:8000)
- **Domínio:** felipe.forjadosdias.tech
- **Repo da app:** https://github.com/ForjaDosDias/Amanuense2 (branch canônico: `claude/implementation-plan-challenges-XpawN` — não há `main` no remoto; `main` local rastreia esse branch)
- **Variáveis obrigatórias:** `DEEPSEEK_API_KEY`
- **Volumes:** `corpus/`, `output/`, `intermediate/`, `data/` (bind mounts do repo)

### naoesqueci
- **Stack:** Vite + React (front) + servidor Node leve (`server/index.js`) num único container
- **Container:** `naoesqueci` (Node serve `dist/` estático + `/api/nfce` + `/api/health` na porta 3001)
- **Domínio:** naoesqueci.forjadosdias.tech
- **Repo da app:** https://github.com/ForjaDosDias/NaoEsqueci2
- **Variáveis:** nenhuma obrigatória (`PORT=3001` opcional; consulta NFC-e usa portais públicos da SEFAZ)
- **Sem volume persistente** — estado do usuário fica no `localStorage` do navegador

### monitoring
- **Stack:** Grafana + Loki + Promtail
- **Pasta na VPS:** `/home/bugbrain/monitoring/`
- **Porta pública:** 3001 (Grafana)
- **Rede:** `coolify` (externa)
- **Variáveis obrigatórias:** `GF_ADMIN_PASSWORD`, `GF_ROOT_URL`
- **Volumes persistentes:** `loki-storage`, `grafana-storage`
- **Promtail coleta:** logs de todos os containers Docker + syslog + auth.log

## Esteira CI/CD (GitHub Actions)

Deploy automatizado por repositório. Modelo de branches com gates crescentes:

- `feature/*` → sem regra. Para entrar na `dev`: PR + check `test` verde (auto-merge, **sem** aprovação humana).
- `dev` → única branch que pode abrir PR para `main` (garantido pelo check `guard`).
- `main` → PR só a partir de `dev` + check `test` verde + **1 aprovação humana**. Merge dispara deploy.

### Workflows (em `<repo>/.github/workflows/`)
- `ci.yml` — `test` em PRs/push de `dev`/`main`, roda no **GitHub-hosted** (ubuntu-latest).
- `guard-main-source.yml` — check `guard`; falha se o PR para `main` não vier de `dev`.
- `deploy.yml` — roda no **self-hosted runner** em push na `main`: `git reset --hard` no HEAD da main
  no diretório do código + `docker compose up -d --build --wait` no diretório do compose.
  Como faz fetch do HEAD atual da main, qualquer execução sobe sempre o estado mais recente (idempotente).

Templates e JSON de rulesets versionados em `vps-infra/ci/templates/` e `vps-infra/ci/rulesets/`.

### Self-hosted runner
- Instalação: `/home/victor/actions-runner/` (binário oficial actions/runner).
- Roda como **serviço systemd, usuário root** (`RUNNER_ALLOW_RUNASROOT=1`) — todos os repos/infra são root:root.
- Labels: `self-hosted, vps-prod` (o `runs-on` do deploy exige ambos).
- **Importante:** registrado **por repositório** (repo-level), NÃO org-level. O registro org-level na
  ForjaDosDias ficou com os jobs presos em `queued` (o Broker do GitHub não roteava os jobs ao runner,
  mesmo com grupo `Default` aberto e labels corretos). Repo-level funcionou de imediato. Cada repo da
  esteira tem seu próprio registro do mesmo runner/host.
- Recuperar/registrar runner para um repo:
  ```bash
  cd /home/victor/actions-runner
  export RUNNER_ALLOW_RUNASROOT=1
  # token: gh api -X POST /repos/<org>/<repo>/actions/runners/registration-token --jq .token
  ./config.sh --unattended --replace --url https://github.com/<org>/<repo> \
    --token <REG_TOKEN> --name vps-prod-runner --labels vps-prod --work _work
  ./svc.sh install root && ./svc.sh start
  ```
- Status: `cd /home/victor/actions-runner && ./svc.sh status` ou `gh api /repos/<org>/<repo>/actions/runners`.

### Aplicar rulesets a um repo novo
```bash
gh api -X POST /repos/<org>/<repo>/rulesets --input vps-infra/ci/rulesets/main.json
gh api -X POST /repos/<org>/<repo>/rulesets --input vps-infra/ci/rulesets/dev.json
gh api -X PATCH /repos/<org>/<repo> -F allow_auto_merge=true -F delete_branch_on_merge=true
```

### Estado atual
- **NaoEsqueci2** — piloto, esteira completa e validada (deploy automático funcionando).
- EstudeOAB, Amanuense, Tamois — pendentes de replicação (Tamois usa `kamal deploy` no lugar do compose).

## Comandos Úteis

```bash
# Ver containers rodando
docker ps

# Ver logs do proxy
docker logs nginx-proxy -f

# Recarregar nginx sem derrubar (após editar nginx.conf)
docker exec nginx-proxy nginx -s reload

# Verificar rede
docker network inspect vps-proxy
```

## Histórico de Contexto

- **Jun/2026:** Repo criado após `sudo apt upgrade` desconfigurar o nginx do sistema. Objetivo: ter toda infra versionada para recuperação rápida.
- Nginx do sistema não é usado — todo o tráfego passa pelos containers Docker.
- Cloudflare fica na frente: o header `CF-Connecting-IP` é usado para IP real do visitante.
