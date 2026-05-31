# Django Code Style Guide

---

## Directory Layout

```
apps/           — All Django apps
services/       — Third-party integrations (Redis, SMS, Celery)
tools/          — Pure Python utilities with no Django dependency
utils/          — Django-aware utilities (base classes, permissions, validators, middlewares)
project_title/  — Django project (settings, root URLs, logging)
manager/        — Internal admin management app
tests/          — Unit tests
CONSTANTS.py    — All magic numbers, strings, time values, cache key prefixes
```

---

## Comments & Docstrings

Write comments in English. Format: `# Capital Words Case`.

```python
# --- Authentication Section ---

# Validate Token And Return User
def authenticate(token: str) -> User | None:
    ...
```

Docstrings only in view classes, only to describe supported HTTP methods:

```python
class AuthView(BaseView):
    """
    GET  -> Get User Info
    POST -> Login Or Register
    """
```

---

## Type Annotations

All variables, attributes, function parameters, and return types must be explicitly annotated:

```python
user_id: int = 42
name: str = "Ali"

def get_user(user_id: int) -> User | None:
    return User.objects.filter(id=user_id).first()
```

---

## Naming

| Entity      | Convention      | Example           |
|-------------|-----------------|-------------------|
| Views       | `NameView`      | `AuthView`        |
| Serializers | `NameSerializer`| `AuthSerializer`  |
| Models      | `NameModel` or plain | `Order`      |
| Constants   | `UPPER_SNAKE`   | `MAX_RETRY_COUNT` |

---

## Security — Zero-Trust Access

Default is deny. Access must be explicitly granted.

### Serializers — always `fields`, never `exclude`

```python
# Correct
class UserSerializer(ModelSerializer):
    class Meta:
        model = User
        fields: list[str] = ["id", "username", "email"]

# Wrong — new model fields leak automatically
class UserSerializer(ModelSerializer):
    class Meta:
        model = User
        exclude: list[str] = ["password"]  # Never do this
```

### Checklist before writing any serializer, view, or permission

- Are all exposed fields explicitly declared?
- Does adding a new model field accidentally expose it?
- Is the minimum permission level enforced?
- Is any sensitive or internal data reachable through this endpoint?

---

## Views — Early Return Pattern

Avoid `if/else` blocks. Use guard clauses at the top.

```python
# Wrong
def get(self, request: Request) -> Response:
    user = User.objects.filter(id=request.user.id).first()
    if user:
        return Response({"name": user.name}, status=status.HTTP_200_OK)
    else:
        return Response(status=status.HTTP_404_NOT_FOUND)

# Correct
def get(self, request: Request) -> Response:
    user = User.objects.filter(id=request.user.id).first()
    if not user:
        return Response(status=status.HTTP_404_NOT_FOUND)
    return Response({"name": user.name}, status=status.HTTP_200_OK)
```

Business logic must delegate to model methods and standalone functions — not inline in views.

---

## Object Retrieval — `get_object_or_error`

Never use Django's `get_object_or_404`. Use `get_object_or_error` from `utils.views`:

```python
# Wrong
from django.shortcuts import get_object_or_404
order = get_object_or_404(Order, id=id, is_finished=True)

# Correct
from utils.views import get_object_or_error
order = get_object_or_error(
    status_code=status.HTTP_400_BAD_REQUEST,
    message="Finished order not found.",
    queryset=Order,
    id=id,
    is_finished=True,
)
```

---

## Response Status Codes

Always use named constants from `rest_framework.status`. Never raw integers.

```python
# Wrong
return Response({"token": token}, status=200)

# Correct
from rest_framework import status
from rest_framework.response import Response

return Response({"token": token}, status=status.HTTP_200_OK)
return Response(status=status.HTTP_404_NOT_FOUND)
```

---

## String Formatting — `.format()` over f-strings

```python
# Wrong
cache_key = f"{CACHE_PREFIX}{user_id}"

# Correct
cache_key = "{}{}".format(CACHE_PREFIX, user_id)
```

---

## Model Choice Fields — Integer Steps of Ten

Use integers in steps of 10. Define in `CONSTANTS.py`.

```python
# Wrong
STATUS_CHOICES = (("created", "Created"), ("paid", "Paid"))

# Wrong — no room to insert between values
STATUS_CHOICES = ((1, "Created"), (2, "Paid"))

# Correct
ORDER_STATUSES: tuple[tuple[int, str], ...] = (
    (10, "Created"),
    (20, "Pending"),
    (30, "Paid"),
)
```

---

## Response Messages — User vs Developer

```python
# Wrong — leaking internal errors to user
return Response({"message": serializer.errors}, status=status.HTTP_400_BAD_REQUEST)

# Correct — separate concerns
return Response(
    {
        "message": "اطلاعات وارد شده صحیح نیست",
        "devMessages": serializer.errors,
    },
    status=status.HTTP_400_BAD_REQUEST,
)

# Correct — server-side check, user must know
return Response({"message": "کد تایید اشتباه است"}, status=status.HTTP_400_BAD_REQUEST)
```

| Situation                                   | Key          |
|---------------------------------------------|--------------|
| Format / required field error               | `devMessage` |
| Wrong code, expired token                   | `message`    |
| Serializer validation detail                | `devMessage` |
| Business rejection (insufficient balance)   | `message`    |

Status code rules:
- `2xx` → no message field needed
- `5xx` → no message field; never leak internals
- `4xx` → include `message` and/or `devMessage`

---

## Property Methods — Annotation Check

```python
@property
def items_count(self) -> int:
    if hasattr(self, "_items_count"):
        return self._items_count
    return self.items.count()
```

Use `annotate(_items_count=...)` with leading underscore in querysets.

---

## Imports

Within an app (with `__all__` defined in each file):
```python
from .models import *
```

Across apps:
```python
from apps.authentication.models import User
```

---

## Base Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `AbstractModel` | `utils/db.py` | Adds `id`, `created_at`, `updated_at` |
| `AbstractAdmin` | `utils/admin.py` | Query optimization, Jalali widgets, auto list_display |
| `BaseView` | `utils/views.py` | Catches `HTTPError`, returns structured JSON |
| `WebAppAPITestCase` | `utils/test.py` | All view tests inherit from this |

---

## Logging

```python
from project_title.log import logger_set

logger = logger_set("authentication.serializer")
logger.info(msg={"message": "Verification code sent", "mobile": mobile})
logger.warning(msg={"message": "Invalid token attempt", "user_id": user_id})
```

Logger name format: `<app>.<module>`

Log every meaningful event: login, register, logout, state change, failure.

---

## Testing Pattern

```python
# File: tests/test_authentication.py

from utils.test import WebAppAPITestCase, APITestCasePack, APITestCaseModel


class AuthViewTest(WebAppAPITestCase):
    base_url = "/api/v1/auth/"
    test_cases = [
        APITestCasePack(
            title="Login With Code",
            method="POST",
            auth_required=False,
            cases=[
                APITestCaseModel(
                    input={"mobile": "09123456789", "code": "12345"},
                    output={"token": ...},
                    expected_code=200,
                )
            ],
        )
    ]

    def setUp(self) -> None:
        self.base_setUp()

    def test_login(self) -> None:
        self.run_tests()
```
