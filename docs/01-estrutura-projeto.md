# 1. Estrutura de Projeto

## Princípio

Organize por **domínio/funcionalidade** (vertical) em vez de por tipo técnico (horizontal). Isso facilita escalar o projeto sem que pastas como `routers/` ou `models/` virem despejos gigantes de arquivos não relacionados.

## Estrutura recomendada

```
app/
├── main.py                  # cria a instância FastAPI, registra routers
├── core/
│   ├── config.py            # Settings (Pydantic Settings), variáveis de ambiente
│   ├── security.py          # hashing, JWT, OAuth2
│   └── logging.py           # configuração de logging
├── api/
│   ├── deps.py               # dependências compartilhadas (get_db, get_current_user)
│   └── v1/
│       ├── router.py         # agrega todos os routers da v1
│       └── endpoints/
│           ├── users.py
│           └── orders.py
├── domain/                   # ou "modules/" — um pacote por contexto de negócio
│   └── orders/
│       ├── models.py          # modelos ORM (SQLAlchemy)
│       ├── schemas.py         # modelos Pydantic (request/response)
│       ├── service.py         # regras de negócio
│       └── repository.py      # acesso a dados
├── db/
│   ├── base.py
│   └── session.py
└── tests/
    └── ...
```

## Regras

- **`main.py` fino.** Só cria app, monta middlewares, inclui routers. Nenhuma lógica de negócio aqui.
- **Separe schema (Pydantic) de model (ORM).** Nunca exponha o modelo de banco direto na API — acopla schema de banco ao contrato público.
- **Um router por recurso**, agregado em um router central por versão (`api/v1/router.py`).
- **Configuração centralizada** via `pydantic-settings`, nunca `os.getenv` espalhado pelo código.

```python
# core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    secret_key: str
    environment: str = "development"

    class Config:
        env_file = ".env"

settings = Settings()
```

## Armadilhas comuns

- Pasta `utils.py` ou `helpers.py` genérica que acumula funções não relacionadas — prefira nomear pelo que faz (`date_utils.py`, `pagination.py`).
- Lógica de negócio dentro do endpoint (`@router.post`) — endpoint deve só orquestrar (validar entrada → chamar service → formatar saída).
- Importar `Settings()` em vários lugares sem cache — use `lru_cache` ou um singleton para evitar reler `.env` repetidamente.
