# 5. Autenticação e Segurança

## OAuth2 + JWT (padrão de mercado)

```python
from fastapi.security import OAuth2PasswordBearer
from jose import jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")

def create_access_token(subject: str, expires_delta: timedelta) -> str:
    expire = datetime.utcnow() + expires_delta
    return jwt.encode({"sub": subject, "exp": expire}, settings.secret_key, algorithm="HS256")
```

- Tokens de acesso com vida curta (15–30 min) + refresh token de vida mais longa, armazenado de forma segura (httpOnly cookie ou storage seguro no cliente).
- **Nunca** coloque dados sensíveis no payload do JWT — ele é apenas codificado em base64, não criptografado.

## Hashing de senha

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

Nunca armazene senha em texto puro nem use MD5/SHA simples — use `bcrypt` ou `argon2`.

## CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.exemplo.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

Nunca use `allow_origins=["*"]` em produção quando `allow_credentials=True` — combinação insegura.

## Rate limiting

Use `slowapi` (baseado em `limits`) ou um API Gateway/proxy reverso (nginx, Traefik) para limitar requisições por IP/usuário e mitigar brute-force e abuso.

## Checklist de segurança

- **HTTPS obrigatório** em produção (`HTTPSRedirectMiddleware` ou enforçado no proxy/load balancer).
- **Secrets fora do código** — variáveis de ambiente ou um secrets manager (Vault, AWS Secrets Manager), nunca hardcoded ou em `.env` commitado.
- **Validação de entrada** (capítulo 3) evita injeção e payloads malformados.
- **`TrustedHostMiddleware`** para restringir hosts aceitos.
- **Headers de segurança** (`X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`) via middleware ou proxy.
- **Não exponha stack trace** em produção — capítulo 7 trata tratamento de erros.
- **Princípio do menor privilégio** em escopos/permissões de usuário e credenciais de banco.

## Armadilhas comuns

- Guardar JWT secret igual entre ambientes (dev/staging/prod).
- Validar permissão só no frontend, sem reforçar no backend.
- Logar senha, token ou dados de cartão em texto puro (ver capítulo 10 sobre logging).
