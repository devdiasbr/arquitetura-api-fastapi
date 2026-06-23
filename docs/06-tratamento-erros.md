# 6. Tratamento de Erros

## Princípio

Erros devem ter **formato consistente** em toda a API e **nunca expor detalhes internos** (stack trace, query SQL, paths de arquivo) ao cliente.

## Exceções de domínio + handler global

```python
# domain/orders/exceptions.py
class OrderNotFound(Exception):
    def __init__(self, order_id: int):
        self.order_id = order_id

# main.py
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(OrderNotFound)
async def order_not_found_handler(request: Request, exc: OrderNotFound):
    return JSONResponse(
        status_code=404,
        content={"error": "order_not_found", "detail": f"Pedido {exc.order_id} não encontrado"},
    )
```

Service/repository lançam exceções de domínio puras (sem depender de FastAPI); o handler traduz para HTTP. Isso mantém a camada de negócio desacoplada do framework web.

## Handler para erro de validação

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"error": "validation_error", "detail": exc.errors()},
    )
```

## Handler genérico (fallback)

```python
@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.exception("Erro não tratado", exc_info=exc)
    return JSONResponse(status_code=500, content={"error": "internal_error", "detail": "Erro interno"})
```

Captura qualquer exceção não esperada, loga com stack trace internamente, mas devolve mensagem genérica ao cliente.

## Formato de erro consistente

Padronize um envelope único, por exemplo:

```json
{
  "error": "order_not_found",
  "detail": "Pedido 42 não encontrado",
  "request_id": "a1b2c3"
}
```

`request_id` correlaciona o erro com logs/tracing (capítulo 9).

## `HTTPException` para casos simples

Para erros pontuais sem necessidade de exceção de domínio dedicada, `HTTPException` direto no endpoint é aceitável:

```python
if not user:
    raise HTTPException(status_code=404, detail="Usuário não encontrado")
```

## Armadilhas comuns

- Deixar exceção do banco (`IntegrityError`, `OperationalError`) propagar até o cliente — sempre capturar e traduzir.
- Misturar formatos de erro diferentes entre endpoints (um devolve `{"error": ...}`, outro `{"message": ...}`).
- Usar `try/except Exception` genérico dentro do endpoint para "engolir" erro e devolver `200` com corpo de erro — quebra contrato HTTP.
