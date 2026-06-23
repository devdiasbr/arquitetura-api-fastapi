# 2. Design de API REST

## Princípios

- **Recursos, não ações.** URLs representam substantivos (`/orders`), métodos HTTP representam verbos (`POST /orders`).
- **Plural para coleções**: `/users`, `/users/{id}`.
- **Versionamento explícito** na URL (`/api/v1/...`) — evita quebrar clientes em mudanças incompatíveis.
- **Status codes corretos**: não devolva sempre `200`. Use `201` para criação, `204` para deleção sem corpo, `400/422` para erro de validação, `404` para recurso inexistente, `409` para conflito.

## Estrutura de endpoint

```python
from fastapi import APIRouter, status

router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("/", response_model=OrderOut, status_code=status.HTTP_201_CREATED)
async def create_order(payload: OrderCreate, service: OrderService = Depends(get_order_service)):
    return await service.create(payload)

@router.get("/{order_id}", response_model=OrderOut)
async def get_order(order_id: int, service: OrderService = Depends(get_order_service)):
    return await service.get_or_404(order_id)
```

## Boas práticas específicas

- **`response_model` sempre declarado.** Garante contrato de saída estável e remove campos sensíveis automaticamente (ex.: `password_hash`).
- **Paginação obrigatória** em listagens (`limit`/`offset` ou cursor-based). Nunca devolva listas sem limite.
- **Filtros e ordenação via query params**, não via corpo em `GET`.
- **Idempotência em `PUT`** (substituição completa) vs **`PATCH`** (atualização parcial) — não misture semântica.
- **Use `Enum` para valores fixos** em path/query params, o Swagger já documenta as opções válidas.

```python
class OrderStatus(str, Enum):
    pending = "pending"
    paid = "paid"
    cancelled = "cancelled"

@router.get("/")
async def list_orders(status: OrderStatus | None = None, limit: int = 20, offset: int = 0):
    ...
```

## Documentação automática (OpenAPI)

- Preencha `summary`, `description` e `tags` nos endpoints — vira documentação Swagger/Redoc de qualidade sem esforço extra.
- Use `responses={...}` para documentar respostas de erro além do caso feliz.

```python
@router.get(
    "/{order_id}",
    response_model=OrderOut,
    summary="Busca pedido por ID",
    responses={404: {"description": "Pedido não encontrado"}},
)
```

## Armadilhas comuns

- Verbos na URL (`/createOrder`, `/getUserById`) — quebra a semântica REST.
- Devolver `200` com corpo `{"error": "..."}" para indicar falha — cliente não consegue tratar via status code.
- Aninhamento excessivo de recursos (`/users/1/orders/2/items/3/reviews/4`) — limite a 2 níveis, use IDs diretos quando possível.
