# vps-infra — Infraestrutura da VPS

Repositório central de configurações de infra e deploy de todos os projetos publicados na VPS. Criado para facilitar recuperação rápida do ambiente após atualizações do sistema (ex: `sudo apt upgrade` pode reconfiguar o nginx do sistema).

## Arquitetura

Todos os projetos rodam em containers Docker conectados via uma rede externa compartilhada chamada `vps-proxy`. O tráfego HTTP entra pelo container `proxy` (nginx reverso) e é roteado para cada aplicação.

```
Internet → proxy (nginx:80) ──► tamois:80       (tamois.com.br)
                            ──► estude-nginx:80  (forjadosdias.com.br, aprovadonaoab.com.br)
```

A rede Docker deve existir antes de subir qualquer serviço:
```bash
docker network create vps-proxy
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
└── estudeoab/          # infra do EstudeOAB (Node + React)
    ├── docker-compose.yml
    ├── nginx/
    │   └── nginx.conf
    └── .env.example
```

## Como Recuperar o Ambiente

### 1. Criar a rede Docker
```bash
docker network create vps-proxy
```

### 2. Subir o proxy reverso
```bash
cd proxy/
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

## Projetos

### proxy
- **Imagem:** nginx:1.27-alpine
- **Porta pública:** 80
- **Função:** roteamento de domínios para os containers de cada projeto via upstream dinâmico (resolve nomes Docker em runtime)
- **Domínios gerenciados:** tamois.com.br, forjadosdias.com.br, aprovadonaoab.com.br

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
