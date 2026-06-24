# 4. Injeção de Dependências

## Princípio

O sistema `Depends()` do FastAPI é a ferramenta central para **desacoplar** endpoints de implementação concreta (banco, autenticação, configuração). Use-o para injetar sessões de banco, usuário autenticado e serviços de negócio.

## Padrão de sessão de banco

```python
# db/session.py
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

```python
@router.get("/{order_id}")
async def get_order(order_id: int, db: AsyncSession = Depends(get_db)):
    ...
```

O `yield` garante que a sessão seja fechada mesmo se o endpoint lançar exceção.

## Camada de serviço como dependência

Evite acessar o repositório/banco direto no endpoint. Injete o serviço:

```python
def get_order_service(db: AsyncSession = Depends(get_db)) -> OrderService:
    return OrderService(OrderRepository(db))

@router.post("/")
async def create_order(payload: OrderCreate, service: OrderService = Depends(get_order_service)):
    return await service.create(payload)
```

Isso torna o endpoint testável isoladamente (mock de `OrderService`) sem precisar de banco real.

## Autenticação via dependência

```python
async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)) -> User:
    payload = decode_token(token)
    user = await db.get(User, payload["sub"])
    if user is None:
        raise HTTPException(status_code=401, detail="Usuário inválido")
    return user
```

Use em qualquer endpoint protegido: `current_user: User = Depends(get_current_user)`.

## Dependências reutilizáveis com classe

Para dependências parametrizáveis (ex.: checar permissão específica):

```python
class RequirePermission:
    def __init__(self, permission: str):
        self.permission = permission

    def __call__(self, user: User = Depends(get_current_user)) -> User:
        if self.permission not in user.permissions:
            raise HTTPException(status_code=403, detail="Sem permissão")
        return user

@router.delete("/{order_id}", dependencies=[Depends(RequirePermission("orders:delete"))])
async def delete_order(order_id: int):
    ...
```

## `Depends` com cache automático

Dentro do mesmo request, FastAPI faz cache de dependências idênticas — `get_db()` chamado por múltiplas dependências no mesmo request retorna a mesma instância (a menos que `use_cache=False`).

## Armadilhas comuns

- Instanciar serviço/repositório diretamente no corpo do endpoint (`service = OrderService(db)`) em vez de via `Depends` — dificulta testes e troca de implementação.
- Lógica de autorização duplicada em cada endpoint em vez de extraída como dependência reutilizável.
- Abrir conexão/sessão sem `yield` + `try/finally` (ou `async with`), arriscando vazamento de conexões.
