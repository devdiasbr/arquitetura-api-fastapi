# 10. Versionamento e Deploy

## Versionamento de API

- **Versionar na URL** (`/api/v1/`, `/api/v2/`) é o padrão mais simples e visível.
- Mantenha versões antigas funcionando por um período de depreciação anunciado — nunca quebre clientes sem aviso.
- Use header `Deprecation` e `Sunset` (RFC 8594) para sinalizar fim de vida de uma versão.

```python
app.include_router(v1_router, prefix="/api/v1")
app.include_router(v2_router, prefix="/api/v2")
```

## Servidor ASGI em produção

Use **Uvicorn com workers gerenciados por Gunicorn** (ou Uvicorn standalone com múltiplos workers) — nunca o servidor de desenvolvimento (`--reload`) em produção.

```bash
gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 4 --bind 0.0.0.0:8000
```

Número de workers ~ `2 × núcleos de CPU + 1` como ponto de partida, ajustar com benchmark real.

## Containerização

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir poetry && poetry install --no-dev --no-root

COPY app ./app

CMD ["gunicorn", "app.main:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "4", "--bind", "0.0.0.0:8000"]
```

- Imagem `slim`, multi-stage build para reduzir tamanho final.
- Nunca copiar `.env` para dentro da imagem — injetar variáveis via orquestrador (Kubernetes Secrets, ECS task definition, etc.).

## Migrações de banco

Use **Alembic** para versionar schema de banco junto com o código. Migração é parte do deploy, não passo manual.

```bash
alembic revision --autogenerate -m "add orders table"
alembic upgrade head
```

Execute migrações em um *init container* ou job separado antes do rollout da nova versão da API — nunca migração disparada pela própria aplicação em runtime sob concorrência.

## CI/CD — checklist mínimo

1. Lint + type check (`ruff`, `mypy`).
2. Testes (capítulo 9) com cobertura mínima definida.
3. Build da imagem com tag imutável (commit SHA, não `latest`).
4. Migração de banco.
5. Deploy com estratégia de rollout gradual (rolling update, canary ou blue-green) e health checks (capítulo 10) como gate.

## Graceful shutdown

Garanta que o processo termine requests em andamento antes de morrer (importante em rolling updates):

```python
@app.on_event("shutdown")  # ou lifespan, abordagem recomendada no FastAPI atual
async def shutdown():
    await engine.dispose()
```

Prefira o padrão `lifespan` (context manager) ao invés dos eventos `startup`/`shutdown` depreciados:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    await connect_resources()
    yield
    await disconnect_resources()

app = FastAPI(lifespan=lifespan)
```

## Armadilhas comuns

- Rodar `uvicorn app.main:app --reload` em produção — sem múltiplos workers, sem resiliência.
- Migração de banco rodando automaticamente no startup da aplicação com múltiplas réplicas subindo ao mesmo tempo — corrida entre instâncias.
- Tag de imagem `latest` em produção — impossibilita rollback determinístico.
- Sem health check de readiness configurado no orquestrador — tráfego roteado para instância ainda não pronta.
