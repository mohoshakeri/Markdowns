# Python Backend Agent README

This document defines how an AI agent must write Python backend code for this codebase and this developer's style.

The goal is not trendy architecture. The goal is explicit, readable, predictable, framework-native code.

---

## 1. Core Personality Of The Code

Write code that is:

- Explicit, not magical.
- Class-first where behavior needs a namespace or future extension.
- Model-centric for business behavior, especially in Django.
- Layer-based, not feature-folder-heavy.
- Readable before compact.
- Fully type annotated.
- Structured with guard clauses and early returns.
- Practical: avoid heavy dependency injection, repositories, interfaces, and micro-exceptions unless the existing project already requires them.

Prefer a clear workflow over clever abstraction.

---

## 2. Directory Rules

### Django Projects

Use this root structure:

```text
apps/              # All Django Apps
services/          # Third-Party Integrations: SMS, Redis, Payment, Celery, Email
tools/             # Pure Python Utilities With No Django Dependency
utils/             # Django-Aware Utilities: Views, Permissions, Validators, Middlewares, DB Helpers
project_title/     # Settings, Root URLs, Logging, ASGI/WSGI
manager/           # Internal Admin Management App
tests/             # Unit And API Tests
CONSTANTS.py       # Shared Constants For Small Projects
constants/         # Split Constants For Larger Projects
requirements.txt
```

Inside a normal Django app, prefer classic files when the app is not large:

```text
apps/authentication/
├── models.py
├── views.py
├── serializers.py
├── urls.py
├── admin.py
├── tasks.py
└── migrations/
```

Only split into folders when a file becomes genuinely large:

```text
apps/payments/
├── models.py
├── views.py
├── serializers.py
├── handlers/
│   ├── payment.py
│   └── invoice.py
└── tasks.py
```

### FastAPI Projects

Use this root structure:

```text
project/
├── core/
│   ├── config.py
│   ├── logging.py
│   ├── security.py
│   └── app.py
├── endpoints/
│   ├── auth.py
│   ├── users.py
│   └── orders.py
├── models/
│   ├── user.py
│   ├── order.py
│   └── __init__.py
├── schemas/
│   ├── auth.py
│   ├── user.py
│   └── order.py
├── services/
│   ├── sms.py
│   └── payments.py
├── repositories/
│   └── users.py
├── dependencies/
│   ├── auth.py
│   └── permissions.py
├── tasks/
├── utils/
├── tools/
├── tests/
├── constants/
└── main.py
```

Important: in this developer's personal style, `services/` mainly means external systems. Do not move all business logic into services by default.

---

## 3. Responsibility Map

| Location | Responsibility |
|---|---|
| `models.py` / `models/` | Business Behavior, State Checks, Query Helpers, DB-Aware Methods |
| `views.py` / `endpoints/` | HTTP Orchestration Only: Parse, Validate, Call Model/Handler, Return Response |
| `serializers.py` / `schemas/` | Input/Output Shape And Explicit Field Exposure |
| `services/` | Third-Party Integrations: SMS, Payment Gateway, Redis, Email, External APIs |
| `utils/` | Framework-Aware Helpers |
| `tools/` | Pure Python Helpers Like Text, Datetime, Number, Crypto |
| `constants/` / `CONSTANTS.py` | Magic Values, Statuses, Cache Keys, Limits |
| `tasks/` | Celery, Cron, Background Jobs |
| `tests/` | Class-Based Test Packs And API Tests |

---

## 4. Business Logic Placement

### Preferred Django Flow

```text
View
  -> Serializer / Input Check
  -> Model Method Or Handler
  -> External Service Only If Needed
  -> Response
```

Correct:

```python
class User(AbstractModel):
    mobile: str
    is_active: bool

    @classmethod
    def get_active_by_mobile(cls, mobile: str) -> "User | None":
        # Get Active User By Mobile
        return cls.objects.filter(mobile=mobile, is_active=True).first()

    def can_buy_product(self, price: int) -> bool:
        # Check User Wallet Balance
        return self.wallet >= price
```

Correct view:

```python
class ProfileView(BaseView):
    """
    GET -> Get Profile Data
    """

    def get(self, request: Request) -> Response:
        user: User | None = User.get_active_by_mobile(request.user.mobile)

        if not user:
            return Response(status=status.HTTP_404_NOT_FOUND)

        ser: ProfileSerializer = ProfileSerializer(user)
        return Response(ser.data, status=status.HTTP_200_OK)
```

Wrong:

```python
def get_active_user(mobile):
    return User.objects.filter(mobile=mobile, is_active=True).first()
```

Do not create free functions when the behavior clearly belongs to a model or handler class.

---

## 5. Services Are For External Integrations

Use service classes for third-party or infrastructure systems.

Correct:

```python
class SmsService:
    def send_code(self, mobile: str, code: str) -> bool:
        # Send Verification Code By External Sms Provider
        payload: dict[str, str] = {
            "mobile": mobile,
            "code": code,
        }
        result: dict[str, object] | None = self._send_request(payload)
        return bool(result)
```

Wrong:

```python
class UserService:
    def get_active_users(self):
        return User.objects.filter(is_active=True)
```

Use `User.get_actives()` or `User.objects.filter(...)` through a model method instead.

---

## 6. Class And Method Rules

Prefer classes over loose functions when behavior is public, reusable, or grouped.

Correct:

```python
class TextTools:
    @staticmethod
    def normalize_mobile(value: str) -> str:
        # Normalize Mobile Digits
        return value.strip().replace("+98", "0")
```

Use private methods for internal workflow steps:

```python
class PaymentHandler:
    def pay(self) -> bool:
        # Run Payment Workflow
        if not self._validate():
            return False
        if not self._create_invoice():
            return False
        if not self._call_gateway():
            return False
        return self._save_result()

    def _validate(self) -> bool:
        # Validate Payment Data
        return True
```

Avoid abstract classes and interfaces unless there are multiple real implementations now.

Avoid `@property` in new code. Use explicit methods:

```python
class User(AbstractModel):
    def get_full_name(self) -> str:
        # Build User Full Name
        return "{} {}".format(self.first_name, self.last_name).strip()
```

Existing legacy properties may remain, but do not introduce new ones unless the project already uses them heavily and consistency requires it.

---

## 7. Type Annotations

Annotate everything:

```python
user_id: int = 42
mobile: str = "09123456789"
items: list[int] = [1, 2, 3]


def get_user(user_id: int) -> User | None:
    # Get User By Id
    return User.objects.filter(id=user_id).first()
```

Use `| None`, not `Optional`.

---

## 8. Constants

Never scatter magic numbers or repeated strings.

Small projects:

```text
CONSTANTS.py
```

Large projects:

```text
constants/
├── auth.py
├── finance.py
└── sms.py
```

Use plain constants, not enums, unless the existing framework requires enum-like choices.

Correct:

```python
USER_ROLE_ADMIN: str = "admin"
USER_ROLE_USER: str = "user"

ORDER_STATUS_CREATED: int = 10
ORDER_STATUS_PENDING: int = 20
ORDER_STATUS_PAID: int = 30
```

Prefer integer choices in steps of ten so future values can be inserted.

---

## 9. Imports

Use standard import order:

```python
import os
from pathlib import Path

import requests
from pydantic import BaseModel

from tools.datetime_tools import DateTimeTools
from utils.views import get_object_or_error
```

Inside a Django app, wildcard imports are allowed only when the imported module defines `__all__`.

Correct:

```python
from .models import *
from .serializers import *
```

With:

```python
from .user import User
from .profile import Profile

__all__: list[str] = ["User", "Profile"]
```

Across apps, be explicit:

```python
from apps.authentication.models import User
```

---

## 10. Error Handling

Expected business failures should usually return `None`, `False`, or a controlled response. Do not create many tiny custom exceptions.

Correct:

```python
def get_user_by_token(token: str) -> User | None:
    # Get User By Token
    if not token:
        return None
    return User.objects.filter(token=token, is_active=True).first()
```

Use exceptions for truly exceptional or framework-level flows only.

Never silence exceptions without logging:

```python
try:
    sent: bool = SmsService().send_code(mobile=mobile, code=code)
except ConnectionError as e:
    logger.error(msg={"message": "Sms Provider Failed", "error": str(e), "mobile": mobile})
    return False
```

---

## 11. API Response Rules

Use named status constants. Never raw status integers.

Django:

```python
return Response(data, status=status.HTTP_200_OK)
```

FastAPI:

```python
return JSONResponse(content=data, status_code=status.HTTP_200_OK)
```

Error response shape:

```python
return Response(
    {
        "message": "اطلاعات وارد شده صحیح نیست",
        "devMessage": ser.errors,
    },
    status=status.HTTP_400_BAD_REQUEST,
)
```

Rules:

```text
2xx -> No Message Needed
4xx -> Include message And/Or devMessage
5xx -> Never Leak Internal Details
```

Use Persian for user-facing `message` when the product is Persian.

---

## 12. Serializer And Schema Security

Zero trust by default.

Always explicitly define exposed fields.

Correct:

```python
class UserSerializer(ModelSerializer):
    class Meta:
        model = User
        fields: list[str] = ["id", "mobile", "name"]
```

Wrong:

```python
class UserSerializer(ModelSerializer):
    class Meta:
        model = User
        exclude: list[str] = ["password"]
```

Never return full ORM models directly from FastAPI endpoints. Use Pydantic schemas.

---

## 13. Validation

Use the framework's validation where it belongs:

- DRF Serializer for API payload shape.
- Pydantic for DTOs, internal structured data, and FastAPI schemas.
- `utils/validators.py` for reusable framework-aware validators.
- Model methods for model-specific business validation.

Prefer Pydantic over dataclasses for structured data contracts.

---

## 14. Logging

Use structured logs with context.

```python
from project_title.log import logger_set

logger = logger_set("authentication.view")

logger.info(msg={"message": "User Login Success", "user_id": user.id})
logger.warning(msg={"message": "Invalid Login Attempt", "mobile": mobile})
```

Logger name format:

```text
<feature>.<module>
```

Log meaningful events:

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

## 15. Comments And Docstrings

Comments must be English and Capital Words Case.

Correct:

```python
# --- Payment Section ---

# Validate Payment Before Gateway Call
```

Do not write long comments. Do not write zero comments. Use short section comments and intent comments.

Django view docstrings may describe methods:

```python
class AuthView(BaseView):
    """
    GET  -> Get User Info
    POST -> Login Or Register
    """
```

Avoid docstrings for every simple helper.

---

## 16. String Formatting

Use `.format()` over f-strings for predictable codebase style.

Correct:

```python
cache_key: str = "{}{}".format(USER_CACHE_PREFIX, user_id)
```

Wrong:

```python
cache_key = f"{USER_CACHE_PREFIX}{user_id}"
```

---

## 17. Dependency Management

Use:

```text
requirements.txt
```

Do not introduce Poetry, Pipenv, or complex packaging unless the project already uses them.

---

## 18. Example: Preferred Django Endpoint

```python
from rest_framework import status
from rest_framework.request import Request
from rest_framework.response import Response

from utils.views import BaseView, get_object_or_error
from .models import *
from .serializers import *


class OrderPayView(BaseView):
    """
    POST -> Pay Order
    """

    def post(self, request: Request, order_id: int) -> Response:
        # Get Order And Validate Access
        order: Order = get_object_or_error(
            status_code=status.HTTP_400_BAD_REQUEST,
            message="سفارش پیدا نشد",
            queryset=Order,
            id=order_id,
            user=request.user,
        )

        if not order.can_pay():
            return Response({"message": "امکان پرداخت این سفارش وجود ندارد"}, status=status.HTTP_400_BAD_REQUEST)

        paid: bool = order.pay()
        if not paid:
            return Response({"message": "پرداخت با خطا مواجه شد"}, status=status.HTTP_400_BAD_REQUEST)

        ser: OrderSerializer = OrderSerializer(order)
        return Response(ser.data, status=status.HTTP_200_OK)
```

---

## 19. Avoid These

Do not generate:

- Heavy Clean Architecture scaffolding.
- Repository pattern in Django.
- Dependency injection containers.
- Many tiny exception classes.
- Large free-function modules when behavior belongs to a model or class.
- `@property` for new explicit behavior.
- `Enum` when plain constants are enough.
- `dataclass` when Pydantic is better for contracts.
- Raw status integers.
- Serializer `exclude`.
- Direct secret exposure.
- Silent `except Exception`.
- F-strings.
- Long comments.

---

# Overcheck Checklist

Before finalizing any Python backend change, verify every item:

## Structure

- [ ] File is placed in the correct layer: model, view/endpoint, serializer/schema, service, util, tool, task, or constants.
- [ ] Django app uses classic files unless splitting is clearly justified.
- [ ] `services/` is used for external integrations, not random business logic.
- [ ] Pure helpers are in `tools/`; framework-aware helpers are in `utils/`.
- [ ] Constants are not scattered.

## Style

- [ ] All variables, parameters, attributes, and returns are type annotated.
- [ ] Guard clauses are used instead of nested `if/else`.
- [ ] Workflow is readable step by step.
- [ ] Comments are short, English, and Capital Words Case.
- [ ] No long explanatory comments hide bad structure.
- [ ] `.format()` is used instead of f-strings.
- [ ] `requirements.txt` remains the dependency source unless project says otherwise.

## Architecture

- [ ] Business behavior that belongs to a model is implemented as a model/class method.
- [ ] Views/endpoints only orchestrate request flow.
- [ ] External APIs are wrapped in service classes.
- [ ] No unnecessary repository, interface, factory, dependency-injection, or micro-exception layer was introduced.
- [ ] Classes are preferred over loose functions when grouping behavior improves readability.

## Security

- [ ] Serializer/schema fields are explicit.
- [ ] No serializer uses `exclude` for public output.
- [ ] No full ORM model is returned directly from FastAPI.
- [ ] Default access is deny.
- [ ] Sensitive fields are not exposed.
- [ ] User-facing errors do not leak internals.

## Responses

- [ ] Named status constants are used.
- [ ] `2xx` responses do not include useless message fields.
- [ ] `4xx` responses include `message` and/or `devMessage`.
- [ ] `5xx` responses do not expose internal details.
- [ ] Persian user-facing messages are used when the product is Persian.

## Error Handling And Logging

- [ ] Expected business failures return `None`, `False`, or a controlled response.
- [ ] No broad exception is swallowed silently.
- [ ] External service failures are logged with context.
- [ ] Logger name follows `<feature>.<module>`.
- [ ] Important events are logged.

## Imports

- [ ] Imports are grouped: stdlib, third-party, local.
- [ ] Wildcard imports are only used locally when `__all__` exists.
- [ ] Cross-app imports are explicit.
- [ ] No circular imports were introduced.

## Final Sanity

- [ ] Code looks like the existing project, not like a generic template.
- [ ] No fashionable architecture was added just to look clean.
- [ ] A junior developer can read the workflow without jumping through many files.
- [ ] The implementation is explicit, boring, and reliable.
