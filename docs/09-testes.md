# 8. Testes

## Pirâmide de testes

- **Unitários**: lógica de serviço/domínio, sem FastAPI nem banco real (mocks).
- **Integração**: endpoint completo via `TestClient`/`httpx.AsyncClient`, com banco de teste (SQLite em memória ou container descartável).
- **E2E**: fluxo completo em ambiente próximo de produção — poucos, focados em jornadas críticas.

## `TestClient` (síncrono) e `AsyncClient` (assíncrono)

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_order():
    response = client.post("/api/v1/orders", json={"quantity": 2})
    assert response.status_code == 201
    assert response.json()["quantity"] == 2
```

Para apps totalmente async, prefira `httpx.AsyncClient` com `ASGITransport`:

```python
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.mark.asyncio
async def test_create_order():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/v1/orders", json={"quantity": 2})
        assert response.status_code == 201
```

## Sobrescrever dependências em teste

Use `app.dependency_overrides` para trocar banco real por banco de teste ou mockar autenticação:

```python
async def override_get_db():
    async with TestSessionLocal() as session:
        yield session

app.dependency_overrides[get_db] = override_get_db
```

Isso evita testes de integração tocarem o banco de produção/dev e permite isolar cada teste com transação/rollback.

## Fixtures (pytest)

```python
@pytest.fixture
async def db_session():
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()

@pytest.fixture
def client(db_session):
    app.dependency_overrides[get_db] = lambda: db_session
    return TestClient(app)
```

## Testando regras de negócio isoladamente

Serviços bem desenhados (capítulo 5) permitem testar regra de negócio sem subir a API:

```python
async def test_order_service_rejects_zero_quantity():
    service = OrderService(repository=FakeOrderRepository())
    with pytest.raises(InvalidOrderQuantity):
        await service.create(OrderCreate(quantity=0))
```

## Armadilhas comuns

- Testes de integração que dependem do banco de produção ou de estado deixado por outro teste (falta de isolamento/rollback).
- Mockar `response_model` inteiro em vez de testar o schema real — perde a garantia de que o contrato de saída está correto.
- Cobertura alta só em "caminho feliz", sem testar erros de validação (`422`), não encontrado (`404`) e autorização (`401`/`403`).
