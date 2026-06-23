# 7. Assincronismo e Performance

## Quando usar `async def`

Use `async def` quando o endpoint faz I/O **não bloqueante** (chamadas a banco assíncrono, HTTP externo via `httpx.AsyncClient`, etc.). Se o código dentro for síncrono e bloqueante (ex.: `requests`, biblioteca CPU-bound), declarar `async def` **não ajuda** — na verdade pode bloquear o event loop inteiro.

```python
# Correto: I/O assíncrono de ponta a ponta
@router.get("/{order_id}")
async def get_order(order_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Order).where(Order.id == order_id))
    return result.scalar_one_or_none()
```

```python
# Errado: chamada síncrona bloqueante dentro de endpoint async
@router.get("/external")
async def call_external():
    response = requests.get("https://api.terceiro.com")  # bloqueia o event loop
    return response.json()
```

Para código síncrono inevitável (lib sem versão async, processamento CPU-bound), declare `def` simples — o FastAPI roda automaticamente numa threadpool, sem bloquear o loop.

```python
@router.get("/relatorio")
def gerar_relatorio_pesado():  # sync de propósito — roda em threadpool
    return processar_relatorio_cpu_bound()
```

## Banco de dados assíncrono

Use driver async (`asyncpg` para Postgres, `aiomysql` para MySQL) com SQLAlchemy 2.0 async ou um ORM async-first.

```python
engine = create_async_engine(settings.database_url, pool_size=20, max_overflow=10)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

## Chamadas HTTP externas concorrentes

```python
import httpx

async def fetch_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient(timeout=5.0) as client:
        responses = await asyncio.gather(*(client.get(u) for u in urls))
    return [r.json() for r in responses]
```

## Background tasks vs fila externa

- `BackgroundTasks` do FastAPI: tarefas leves, pós-resposta (enviar e-mail simples, registrar log). Roda no mesmo processo — se o servidor cair, a tarefa se perde.
- Para tarefas críticas/pesadas (processamento de imagem, envio em massa), use fila de mensagens real (Celery, RQ, Arq) com worker dedicado e retry.

```python
@router.post("/")
async def create_order(payload: OrderCreate, background_tasks: BackgroundTasks):
    order = await service.create(payload)
    background_tasks.add_task(send_confirmation_email, order.id)
    return order
```

## Connection pooling e timeouts

- Configure `pool_size`/`max_overflow` no engine do banco conforme número de workers.
- Sempre defina `timeout` em clientes HTTP externos — sem isso, uma dependência lenta trava requests indefinidamente.

## Armadilhas comuns

- `async def` com chamada bloqueante dentro (driver de banco síncrono, `time.sleep`, `requests`) — degrada todo o servidor sob carga.
- Abrir nova conexão de banco a cada request em vez de usar pool.
- Não definir timeout em chamadas externas, causando cascata de lentidão.
