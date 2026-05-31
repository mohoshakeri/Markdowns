# FastAPI Code Style Guide

---

## Directory Layout

```text
endpoints/       — API Routers And HTTP Layer
models/          — Database Models
schemas/         — Request And Response Schemas
repositories/    — Database Queries And Persistence Logic
services/        — Business Logic And Third-Party Integrations
templates/       — HTML Or Message Templates
tasks/           — Background Jobs, Celery Tasks, Cron Jobs
tests/           — Test Files
utils/           — Framework-Aware Utilities
tools/           — Pure Python Utilities
middlewares/     — FastAPI Middlewares
dependencies/    — FastAPI Dependencies
core/            — Settings, Logging, Security, App Factory
CONSTANTS.py     — Magic Numbers, Strings, Time Values, Cache Prefixes
main.py          — FastAPI Entry Point
```

---

## Preferred Structure

```text
project/
├── core/
│   ├── config.py
│   ├── logging.py
│   ├── security.py
│   └── app.py
│
├── endpoints/
│   ├── auth.py
│   ├── users.py
│   └── orders.py
│
├── models/
│   ├── user.py
│   ├── order.py
│   └── __init__.py
│
├── schemas/
│   ├── auth.py
│   ├── user.py
│   └── order.py
│
├── repositories/
│   ├── users.py
│   └── orders.py
│
├── services/
│   ├── auth.py
│   ├── sms.py
│   └── payments.py
│
├── dependencies/
│   ├── auth.py
│   └── permissions.py
│
├── tasks/
│   ├── celery.py
│   └── notifications.py
│
├── utils/
│   ├── responses.py
│   ├── exceptions.py
│   └── db.py
│
├── tools/
│   ├── datetime_tools.py
│   └── text_tools.py
│
├── tests/
│   ├── test_auth.py
│   └── test_orders.py
│
├── CONSTANTS.py
└── main.py
```

---

## Layer Rules

| Layer           | Responsibility                                    |
| --------------- | ------------------------------------------------- |
| `endpoints/`    | Receive Request, Validate Access, Return Response |
| `schemas/`      | Define Input And Output Shape                     |
| `services/`     | Business Logic                                    |
| `repositories/` | Database Queries                                  |
| `models/`       | ORM Models Only                                   |
| `dependencies/` | Auth, Permission, Common Dependency Injection     |
| `tools/`        | Pure Python Helpers                               |
| `utils/`        | FastAPI-Aware Helpers                             |

Endpoints must stay thin.

Business logic must not be written directly inside endpoints.

---

## Comments

Write comments in English.

Format: `# Capital Words Case`

```python
# --- Authentication Section ---

# Validate Token And Return User
def authenticate_token(token: str) -> UserModel | None:
    ...
```

---

## Type Annotations

All variables, function parameters, and return types must be explicitly annotated.

```python
user_id: int = 42
mobile: str = "09123456789"

def get_user_by_id(user_id: int) -> UserModel | None:
    ...
```

---

## Naming

| Entity         | Convention                         | Example                            |
| -------------- | ---------------------------------- | ---------------------------------- |
| Endpoints File | Plural Or Feature Name             | `users.py`, `auth.py`              |
| Router         | `router`                           | `router = APIRouter()`             |
| Schemas        | `NameSchema`                       | `UserCreateSchema`                 |
| Models         | `NameModel`                        | `UserModel`                        |
| Services       | `NameService` Or Function-Based    | `AuthService`, `login_user`        |
| Repositories   | `NameRepository` Or Function-Based | `UserRepository`, `get_user_by_id` |
| Constants      | `UPPER_SNAKE`                      | `MAX_RETRY_COUNT`                  |

---

## Endpoint Pattern

Use early return and guard clauses.

```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session

from dependencies.auth import get_current_user
from schemas.auth import LoginInputSchema, LoginOutputSchema
from services.auth import login_user
from utils.responses import error_response
from utils.db import get_db

router: APIRouter = APIRouter(prefix="/auth", tags=["Auth"])


@router.post(
    "/login",
    response_model=LoginOutputSchema,
    status_code=status.HTTP_200_OK,
)
def login(
    payload: LoginInputSchema,
    db: Session = Depends(get_db),
) -> LoginOutputSchema:
    # Validate Login Data And Return Token
    result: LoginOutputSchema | None = login_user(db=db, payload=payload)

    if not result:
        return error_response(
            status_code=status.HTTP_400_BAD_REQUEST,
            message="اطلاعات ورود صحیح نیست",
        )

    return result
```

---

## Response Status Codes

Always use named constants from `fastapi.status`.

```python
from fastapi import status

status.HTTP_200_OK
status.HTTP_201_CREATED
status.HTTP_400_BAD_REQUEST
status.HTTP_401_UNAUTHORIZED
status.HTTP_404_NOT_FOUND
status.HTTP_500_INTERNAL_SERVER_ERROR
```

Never use raw integers.

---

## Response Messages

| Situation             | Key                |
| --------------------- | ------------------ |
| User-Facing Error     | `message`          |
| Developer Detail      | `devMessage`       |
| Validation Detail     | `devMessage`       |
| Business Rejection    | `message`          |
| Internal Server Error | No Internal Detail |

```python
return {
    "message": "اطلاعات وارد شده صحیح نیست",
    "devMessage": errors,
}
```

Rules:

```text
2xx → No Message Needed
4xx → Include message And/Or devMessage
5xx → Never Leak Internal Details
```

---

## Schemas — Zero Trust

Always explicitly define exposed fields.

Never return ORM models directly from endpoints.

```python
from pydantic import BaseModel, ConfigDict


class UserOutputSchema(BaseModel):
    id: int
    username: str
    email: str

    model_config = ConfigDict(from_attributes=True)
```

Wrong:

```python
# Never Return Full ORM Model Directly
return user
```

Correct:

```python
return UserOutputSchema.model_validate(user)
```

---

## Object Retrieval

Never query objects directly inside endpoints.

Use repositories or utility functions.

```python
from sqlalchemy.orm import Session

from models.user import UserModel


def get_user_or_none(db: Session, user_id: int) -> UserModel | None:
    # Get User By Id
    return db.query(UserModel).filter(UserModel.id == user_id).first()
```

For expected errors:

```python
from fastapi import HTTPException, status


def get_user_or_error(db: Session, user_id: int) -> UserModel:
    # Get User Or Raise Controlled Error
    user: UserModel | None = get_user_or_none(db=db, user_id=user_id)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={"message": "کاربر پیدا نشد"},
        )

    return user
```

---

## String Formatting

Use `.format()` over f-strings.

```python
cache_key: str = "{}{}".format(USER_CACHE_PREFIX, user_id)
```

---

## Choice Fields

Use integers in steps of ten.

Define choices in `CONSTANTS.py`.

```python
ORDER_STATUS_CREATED: int = 10
ORDER_STATUS_PENDING: int = 20
ORDER_STATUS_PAID: int = 30
ORDER_STATUS_CANCELED: int = 40
```

---

## Imports

Inside same layer:

```python
from schemas.auth import LoginInputSchema
from services.auth import login_user
```

Avoid circular imports.

Avoid wildcard imports unless `__all__` is explicitly defined.

---

## Base Utilities

| Item                       | Location              | Purpose                               |
| -------------------------- | --------------------- | ------------------------------------- |
| `BaseModelMixin`           | `utils/db.py`         | Adds `id`, `created_at`, `updated_at` |
| `get_db`                   | `utils/db.py`         | Database Session Dependency           |
| `error_response`           | `utils/responses.py`  | Structured Error Response             |
| `BaseHTTPExceptionHandler` | `utils/exceptions.py` | Global Error Handling                 |
| `BaseAPITestCase`          | `utils/test.py`       | Shared Test Helpers                   |

---

## Logging

```python
from core.logging import logger_set

logger = logger_set("auth.service")

logger.info({"message": "User Login Success", "user_id": user.id})
logger.warning({"message": "Invalid Login Attempt", "mobile": payload.mobile})
```

Logger name format:

```text
<feature>.<layer>
```

Examples:

```text
auth.endpoint
auth.service
users.repository
orders.task
```

Log every meaningful event:

```text
Login
Register
Logout
Payment
State Change
Permission Failure
External Service Failure
```

---

## Testing Pattern

```python
from fastapi import status
from fastapi.testclient import TestClient

from main import app

client: TestClient = TestClient(app)


def test_login_success() -> None:
    # Send Login Request
    response = client.post(
        "/api/v1/auth/login",
        json={
            "mobile": "09123456789",
            "code": "12345",
        },
    )

    assert response.status_code == status.HTTP_200_OK
    assert "token" in response.json()
```

