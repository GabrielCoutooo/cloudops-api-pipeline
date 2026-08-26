# CloudOps API — Pipeline CI/CD

[![CI](https://github.com/GabrielCoutooo/cloudops-api-pipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/GabrielCoutooo/cloudops-api-pipeline/actions/workflows/ci.yml)
[![CD](https://github.com/GabrielCoutooo/cloudops-api-pipeline/actions/workflows/cd.yml/badge.svg)](https://github.com/GabrielCoutooo/cloudops-api-pipeline/actions/workflows/cd.yml)
[![Docker Hub](https://img.shields.io/badge/docker%20hub-cloudops--api-blue?logo=docker)](https://hub.docker.com/r/gabrielcoutooo/cloudops-api)

API REST em Python (FastAPI) com pipeline de CI/CD em GitHub Actions: testes automatizados, análise estática de segurança (SAST) com Semgrep, containerização com Docker e publicação automática no Docker Hub, seguindo o fluxo de branching GitFlow.

## Índice

- [Stack](#stack)
- [Endpoints](#endpoints)
- [Executando localmente](#executando-localmente)
- [Como funciona o pipeline](#como-funciona-o-pipeline)
- [GitFlow](#gitflow)
- [Secrets](#secrets)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Checklist de requisitos](#checklist-de-requisitos)

## Stack

| Camada           | Tecnologia                          |
|-------------------|--------------------------------------|
| Linguagem          | Python 3.14                          |
| Framework          | FastAPI + Uvicorn                    |
| Testes             | Pytest + HTTPX (`TestClient`)        |
| SAST               | Semgrep (`p/python`, `p/security-audit`) |
| Containerização    | Docker                               |
| CI/CD              | GitHub Actions                       |
| Registry           | Docker Hub                           |

## Endpoints

| Método | Rota      | Descrição                    | Resposta de exemplo |
|--------|-----------|-------------------------------|----------------------|
| `GET`  | `/`       | Mensagem de status da API     | `{"mensagem": "Hello World - CloudOps Pipeline!", "status": "online", "versao": "1.0.0"}` |
| `GET`  | `/health` | Health check da aplicação     | `{"status": "healthy"}` |

## Executando localmente

**Pré-requisitos:** Python 3.11+ e pip.

```bash
# clonar o repositório
git clone https://github.com/GabrielCoutooo/cloudops-api-pipeline.git
cd cloudops-api-pipeline

# instalar dependências
pip install -r requirements.txt

# subir a aplicação
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

A API sobe em `http://localhost:8000`. Documentação interativa (Swagger) em `http://localhost:8000/docs`.

### Rodando os testes

```bash
pytest tests/ -v
```

### Rodando com Docker

```bash
docker build -t cloudops-api .
docker run -p 8000:8000 cloudops-api
```

Ou usando a imagem já publicada:

```bash
docker pull gabrielcoutooo/cloudops-api:latest
docker run -p 8000:8000 gabrielcoutooo/cloudops-api:latest
```

## Como funciona o pipeline

O projeto tem dois workflows separados em `.github/workflows/`, cada um com uma responsabilidade bem definida.

### `ci.yml` — Integração Contínua

**Gatilho:** Pull Request para `develop` ou `main`.

1. **Testes unitários** — roda a suíte `pytest` contra os endpoints da API. Se algum teste quebrar, o job falha e o PR fica bloqueado para merge.
2. **Análise de segurança (Semgrep)** — executa os rulesets `p/python` e `p/security-audit` sobre `app/` e `tests/`. Essa etapa não bloqueia o merge (`continue-on-error: true`); os findings ficam registrados no log do job, garantindo visibilidade sem travar o fluxo de entrega.

### `cd.yml` — Entrega Contínua

**Gatilho:** push em `main` (ou seja, todo merge de PR nessa branch).

1. Login no Docker Hub usando os secrets do repositório.
2. Build da imagem com Docker Buildx.
3. Push para `gabrielcoutooo/cloudops-api` com duas tags:
   - `latest` — sempre aponta para a versão mais recente
   - `<sha do commit>` — permite reverter para uma versão específica quando necessário

> O CI nunca faz build/push de imagem, e o CD nunca roda testes — cada workflow tem uma única responsabilidade, o que facilita debugar quando algo falha.

## GitFlow

| Branch      | Papel                                                        |
|-------------|----------------------------------------------------------------|
| `main`      | Código de produção. **Protegida** — só aceita merge via PR com CI verde |
| `develop`   | Integração das features antes de irem pra produção             |
| `feature/*` | Uma branch por funcionalidade em desenvolvimento                |

**Fluxo:** `feature/*` → PR → `develop` → PR → `main`. Nenhuma alteração — nem esta documentação — entra diretamente em `main`.

## Secrets

Configurados em `Settings → Secrets and variables → Actions`:

| Secret                | Uso                                  |
|------------------------|----------------------------------------|
| `DOCKERHUB_USERNAME`   | Autenticação no Docker Hub             |
| `DOCKERHUB_TOKEN`      | Access Token do Docker Hub (não a senha da conta) |

Nenhuma credencial aparece no código-fonte, em arquivos versionados ou nos logs do pipeline.

## Estrutura do repositório

```
.
├── .github/
│   └── workflows/
│       ├── ci.yml          # Testes + Semgrep (PRs)
│       └── cd.yml          # Build + push da imagem (merge em main)
├── app/
│   ├── __init__.py
│   └── main.py              # Código da API
├── tests/
│   └── test_main.py         # Testes unitários
├── .dockerignore
├── .gitignore
├── Dockerfile
├── requirements.txt
└── README.md
```

## Checklist de requisitos

- [x] API com 2+ endpoints funcionais
- [x] Testes unitários com Pytest, pipeline falha em caso de erro
- [x] Semgrep integrado e reportando findings no CI
- [x] CI disparando em Pull Requests
- [x] CD disparando apenas no merge para `main`
- [x] Imagem publicada no Docker Hub com tags `latest` e SHA
- [x] GitFlow implementado (`main`, `develop`, `feature/*`)
- [x] Histórico com PR `feature → develop` e `develop → main`
- [x] Branch `main` protegida
- [x] Nenhuma credencial exposta