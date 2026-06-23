# 11. ORM e Banco de Dados (SQLAlchemy)

## Princípio

ORM fica isolado em uma camada de acesso a dados (repository). Service/endpoint nunca monta query SQLAlchemy direto — facilita troca de implementação e testes com fake repository.

## Setup assíncrono (SQLAlchemy 2.0)

```python
# db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    settings.database_url,  # postgresql+asyncpg://...
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    echo=settings.environment == "development",
)

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

- `pool_pre_ping=True` evita erro de conexão morta (timeout do banco/proxy).
- `expire_on_commit=False` evita lazy-load acidental depois do commit (que falharia em contexto async).

## Modelos (declarative + `Mapped`)

```python
# domain/orders/models.py
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey
from db.base import Base

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    quantity: Mapped[int]
    status: Mapped[str] = mapped_column(default="pending")

    user: Mapped["User"] = relationship(back_populates="orders")
```

- Use `Mapped[...]` (estilo SQLAlchemy 2.0) em vez de `Column(...)` solto — ganha type hints reais.
- Índice em toda FK e em colunas usadas em `WHERE`/`ORDER BY` com frequência.

## Repository pattern

```python
# domain/orders/repository.py
class OrderRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get(self, order_id: int) -> Order | None:
        return await self.db.get(Order, order_id)

    async def list_by_user(self, user_id: int, limit: int, offset: int) -> list[Order]:
        result = await self.db.execute(
            select(Order).where(Order.user_id == user_id).limit(limit).offset(offset)
        )
        return list(result.scalars().all())

    async def create(self, order: Order) -> Order:
        self.db.add(order)
        await self.db.commit()
        await self.db.refresh(order)
        return order
```

Service usa o repository, nunca a sessão direto — troca de banco/ORM no futuro não vaza para regra de negócio.

## Evitar N+1

```python
from sqlalchemy.orm import selectinload

result = await db.execute(
    select(Order).options(selectinload(Order.items)).where(Order.user_id == user_id)
)
```

Sem `selectinload`/`joinedload`, acessar `order.items` depois dispara uma query por order (N+1). Em endpoint que lista N pedidos, isso vira N+1 queries — gargalo clássico de performance.

## Transações

Commit/rollback explícito no repository ou no service — não deixe transação "pendurada" sem fechamento:

```python
async def transfer_balance(self, from_id: int, to_id: int, amount: Decimal) -> None:
    async with self.db.begin():
        await self.db.execute(update(Account).where(Account.id == from_id).values(balance=Account.balance - amount))
        await self.db.execute(update(Account).where(Account.id == to_id).values(balance=Account.balance + amount))
```

`async with db.begin()` garante commit automático no fim do bloco e rollback se exceção ocorrer.

## Migrações (Alembic)

Schema do banco é versionado junto do código — nunca alterar tabela manualmente em produção. Detalhe de pipeline de migração no [capítulo 10](10-deploy-versionamento.md).

```bash
alembic revision --autogenerate -m "add status column to orders"
alembic upgrade head
```

Revise sempre o autogenerate manualmente — ele não detecta tudo (ex.: rename de coluna vira drop+add por padrão).

## Connection pooling

- `pool_size` dimensionado para `workers × conexões_simultâneas_esperadas`, sem exceder `max_connections` do banco.
- Monitore conexões ociosas/saturação do pool — pool esgotado trava requests em fila silenciosamente.

## Armadilhas comuns

- Acessar relationship lazy (`order.items`) fora de uma sessão ativa — erro `DetachedInstanceError` em contexto async.
- Misturar sessão síncrona e assíncrona no mesmo projeto sem necessidade — dobra complexidade de configuração.
- Query solta em loop (`for id in ids: await db.get(Model, id)`) em vez de uma query `WHERE id IN (...)`.
- Não usar `pool_pre_ping`/health check de conexão — erro esporádico "conexão fechada pelo servidor" sob baixo tráfego.
