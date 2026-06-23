# 9. Observabilidade e Logging

## Logging estruturado

Prefira logs em JSON (estruturado) a texto livre — facilita busca/filtro em ferramentas como ELK, Datadog, Loki.

```python
import logging
import json_log_formatter  # ou structlog

logger = logging.getLogger("app")
```

Com `structlog`:

```python
import structlog

logger = structlog.get_logger()

logger.info("order_created", order_id=order.id, user_id=user.id, amount=order.total)
```

## Middleware de request ID

Correlacione todos os logs de um request com um ID único, propagado também na resposta para suporte ao cliente:

```python
import uuid
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = str(uuid.uuid4())
        structlog.contextvars.bind_contextvars(request_id=request_id)
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response

app.add_middleware(RequestIDMiddleware)
```

## O que logar (e o que não logar)

- **Logar**: ID de request, endpoint, status code, latência, ID de usuário/recurso, erros com stack trace (nível interno).
- **Nunca logar**: senha, token, JWT completo, dados de cartão, CPF/dados pessoais sensíveis sem mascaramento. Use scrubbing/redação automática se a lib suportar.

## Métricas

Exponha métricas Prometheus (`prometheus-fastapi-instrumentator`) para: contagem de requests por endpoint/status, latência (histograma), requests em andamento.

```python
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app)
```

## Tracing distribuído

Para sistemas com múltiplos serviços, use OpenTelemetry para rastrear uma requisição através de API → banco → serviços externos:

```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

FastAPIInstrumentor.instrument_app(app)
```

## Health checks

Separe **liveness** (processo está rodando) de **readiness** (pronto para receber tráfego, ex.: banco conectado):

```python
@app.get("/health/live")
async def liveness():
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness(db: AsyncSession = Depends(get_db)):
    await db.execute(text("SELECT 1"))
    return {"status": "ok"}
```

## Armadilhas comuns

- Logar `request.body()` completo sem filtrar campos sensíveis.
- Não correlacionar logs entre serviços (sem trace/request ID) — debugging em produção fica quase impossível.
- Health check que só responde `200` sem checar dependências reais (banco, cache) — dá falso positivo no orquestrador (Kubernetes etc.).
