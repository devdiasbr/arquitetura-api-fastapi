# 3. Validação e Modelos (Pydantic)

## Princípio

FastAPI usa Pydantic para validar e serializar dados na borda da aplicação. Aproveite isso para que **dados inválidos nunca cheguem à camada de negócio**.

## Separe schemas por finalidade

Não use um único modelo para tudo — crie variantes por operação:

```python
class UserBase(BaseModel):
    name: str
    email: EmailStr

class UserCreate(UserBase):
    password: str

class UserUpdate(BaseModel):
    name: str | None = None
    email: EmailStr | None = None

class UserOut(UserBase):
    id: int
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)  # antigo orm_mode
```

- `UserCreate` exige `password`; `UserOut` nunca expõe `password`/hash.
- `UserUpdate` com todos os campos opcionais permite `PATCH` parcial.

## Validações declarativas

Prefira validação no schema a validação manual no endpoint:

```python
from pydantic import field_validator, Field

class OrderCreate(BaseModel):
    quantity: int = Field(gt=0, le=1000)
    discount_pct: float = Field(ge=0, le=100, default=0)

    @field_validator("quantity")
    @classmethod
    def quantity_must_be_even(cls, v: int) -> int:
        if v % 2 != 0:
            raise ValueError("quantidade deve ser par")
        return v
```

## Tipos especializados

Use os tipos do Pydantic em vez de `str` genérico quando aplicável: `EmailStr`, `HttpUrl`, `SecretStr`, `PositiveInt`, `constr`. Eles documentam intenção e validam formato automaticamente.

## `response_model_exclude_unset` e afins

Para `PATCH`, use `model_dump(exclude_unset=True)` para atualizar só os campos enviados:

```python
@router.patch("/{user_id}", response_model=UserOut)
async def update_user(user_id: int, payload: UserUpdate, service=Depends(get_user_service)):
    return await service.update(user_id, payload.model_dump(exclude_unset=True))
```

## Armadilhas comuns

- Reutilizar o mesmo schema para request e response — vaza campos sensíveis ou exige campos que não fazem sentido na entrada.
- Validar regra de negócio (ex.: "email já cadastrado") dentro do Pydantic — isso é responsabilidade da camada de serviço, que tem acesso ao banco. Pydantic valida **forma**, não **regra de negócio**.
- Ignorar `model_config = ConfigDict(from_attributes=True)` e tentar converter objeto ORM manualmente campo a campo.
