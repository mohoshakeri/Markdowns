# Fullstack Agent README

This document tells an AI agent how to write code for a single-source project that contains both a Python backend and a Vue frontend.

The goal is not trendy architecture. The goal is explicit, readable, predictable code that matches this developer's style and the existing deploy model.

---

## 1. Project Personality

Write code that is:

- Explicit, not magical.
- Layer-based, not deep feature-folder-based.
- Model-centric on the Python/Django side.
- View/page-first on the Vue side.
- Centralized for cross-cutting concerns.
- Readable before compact.
- Fully type-aware in Python.
- Simple CSS and Bootstrap-friendly in Vue.
- Easy for a junior developer to follow.

Do not over-engineer with heavy dependency injection, repositories, interfaces, abstract factories, auto imports, micro-exceptions, or hidden framework magic unless the existing source already uses them.

---

## 2. Expected Fullstack Layout

Use this shape for a Django + Vue source in one repository:

```text
project/
├── apps/                 # Django Apps
├── services/             # External Integrations: SMS, Redis, Payment, Celery
├── tools/                # Pure Python Tools, No Django Dependency
├── utils/                # Django-Aware Utilities
├── project/              # Django Settings, WSGI, URLs, Logging
├── manager/              # Internal Admin Management App
├── tests/                # Backend Tests
├── CONSTANTS.py          # Backend Magic Values
├── manage.py
├── requirements.txt
│
├── client/               # Vue/Vite Frontend
│   ├── src/
│   │   ├── views/
│   │   ├── components/
│   │   ├── router/
│   │   ├── store/
│   │   ├── api/          # Preferred For New Code
│   │   ├── utils/        # Existing Practical Utilities
│   │   ├── constants/
│   │   ├── validators/
│   │   ├── styles/
│   │   └── env.js
│   ├── package.json
│   └── vite.config.js
│
├── Dockerfile
├── entrypoint.sh
├── nginx.conf
├── supervisord.conf
└── crontab
```

For older Vue code, `src/utils/client.js`, `src/utils/store.js`, `src/utils/public.js`, and `src/utils/router.js` are acceptable. For new code, prefer splitting into `api/`, `stores/`, `validators/`, and `router/` only when it improves clarity without breaking existing style.

---

## 3. Fullstack Runtime Contract

This project is designed as one container:

```text
Nginx :80
├── /              -> Vue SPA From /app/client/dist
├── /server/       -> Django/Gunicorn Upstream On 127.0.0.1:3000
├── /static/       -> Django Static Files
└── /assets/       -> Shared Public Assets
```

Supervisor runs:

```text
django         -> gunicorn project.wsgi:application --bind 0.0.0.0:3000
celery-worker  -> celery -A services worker -l info
cron           -> supercronic /app/crontab
nginx          -> /usr/sbin/nginx -g "daemon off;"
```

Entrypoint runs:

```bash
python manage.py migrate --skip-checks
python manage.py collectstatic --no-input --skip-checks
```

Do not casually split this into Docker Compose, multiple containers, or a separate frontend deployment unless explicitly requested.

---

## 4. Backend Philosophy

Backend code must be Django-native and model-centric.

Preferred flow:

```text
View
↓
Model Method
↓
External Service If Needed
↓
Response
```

Avoid this unless the current project already uses it:

```text
View
↓
Service
↓
Repository
↓
Model
```

`services/` means external integrations, not the default home for business logic.

Good examples:

```python
class User(models.Model):
    """Represent User Data And Behavior."""
    mobile: str
    is_active: bool

    @staticmethod
    def get_actives() -> QuerySet["User"]:
        """Get Actives."""
        # Get Active Users
        return User.objects.filter(is_active=True)
```

```python
class Payment(models.Model):
    """Represent Payment Data And Behavior."""
    amount: int
    status: int

    def mark_paid(self) -> None:
        """Mark Paid."""
        # Mark Payment As Paid
        self.status = PAYMENT_STATUS_PAID
        self.save(update_fields=["status"])
```

---

## 5. Backend Directory Rules

Use this separation:

```text
apps/       -> Product Apps And Django Models/Views/Serializers
services/   -> External Systems: SMS, Redis, Payment, Email, Celery
tools/      -> Pure Python Helpers: Text, Datetime, Numbers
utils/      -> Django-Aware Helpers: Views, Permissions, Validators, DB
CONSTANTS.py -> Shared Constants
```

Inside Django apps, prefer standard files first:

```text
authentication/
├── models.py
├── views.py
├── serializers.py
├── urls.py
├── admin.py
└── tasks.py
```

Only create folders when the file becomes large:

```text
models/
├── user.py
├── profile.py
└── __init__.py
```

For small apps, keep `models.py`, `views.py`, and `serializers.py`.

---

## 6. Backend Naming

Use:

```text
NameView
NameSerializer
NameHandler
NameService
NameManager
UPPER_SNAKE_CONSTANT
```

Prefer `Handler` for orchestration classes.

```python
class PaymentHandler:
    """Orchestrate Payment Workflow."""
    def pay(self) -> bool:
        """Pay."""
        # Run Payment Workflow
        if not self._validate():
            return False

        self._create_invoice()
        self._call_gateway()
        self._save_result()
        self._send_sms()
        return True
```

Use leading underscore for internal methods.

---

## 7. Backend Type Rules

Python type annotations are mandatory and should be strict.

```python
user_id: int = 42
mobile: str = "09123456789"

def get_user(user_id: int) -> User | None:
    """Get User."""
    # Get User By Id
    return User.objects.filter(id=user_id).first()
```

Use `| None`, not `Optional`.

---

## 8. Backend Imports

Within an app, wildcard import is allowed only when `__all__` is defined.

```python
# models.py
from .user import User
from .profile import Profile

__all__: list[str] = [
    "User",
    "Profile",
]
```

```python
# views.py
from .models import *
```

Across apps, import explicitly.

```python
from apps.authentication.models import User
```

Avoid circular imports.

---

## 9. Backend Constants

Prefer direct constants over Enum for most code.

```python
USER_ROLE_ADMIN: str = "admin"
USER_ROLE_USER: str = "user"

PAYMENT_STATUS_CREATED: int = 10
PAYMENT_STATUS_PENDING: int = 20
PAYMENT_STATUS_PAID: int = 30
```

Use integer choices in steps of ten.

Keep magic strings, numbers, timeouts, cache prefixes, and status values in `CONSTANTS.py` or a dedicated constants module when the list becomes large.

---

## 10. Backend Responses

Use named HTTP status constants. Never use raw integers in Django/DRF responses.

```python
return Response(
    {"message": "اطلاعات وارد شده صحیح نیست"},
    status=status.HTTP_400_BAD_REQUEST,
)
```

Response rules:

```text
2xx -> Usually No message Needed
4xx -> Include Persian message And/Or devMessage
5xx -> Never Leak Internal Details
```

Use:

```text
message     -> User-Facing Persian Message
devMessage  -> Developer Detail Or Serializer Errors
```

Never show internal errors to users.

---

## 11. Backend Error Handling

Do not create many tiny custom exceptions.

Prefer:

```python
try:
    result: dict[str, Any] = service.run()
except ConnectionError:
    logger.warning(msg={"message": "External Service Failed"})
    return None
```

For expected missing objects, returning `None` is acceptable.

```python
def get_user(user_id: int) -> User | None:
    """Get User."""
    # Get User Or None
    return User.objects.filter(id=user_id).first()
```

For API errors, use the project's controlled error response/helper if it exists.

---

## 12. Backend Views

Views are orchestration layers.

They may:

```text
Read Request
Validate Access
Call Model Methods
Call Serializers
Call External Services When Needed
Return Response
```

They must not contain heavy business logic.

Use early returns:

```python
def post(self, request: Request) -> Response:
    """Post."""
    mobile: str | None = request.data.get("mobile")

    if not mobile:
        return Response(
            {"message": "شماره موبایل وارد نشده است"},
            status=status.HTTP_400_BAD_REQUEST,
        )

    user: User | None = User.get_by_mobile(mobile=mobile)

    if not user:
        return Response(
            {"message": "کاربر پیدا نشد"},
            status=status.HTTP_400_BAD_REQUEST,
        )

    token: str = user.create_login_token()
    return Response({"token": token}, status=status.HTTP_200_OK)
```

---

## 13. Backend Serializers And Security

Zero-trust rule: expose only explicit fields.

Correct:

```python
class UserSerializer(ModelSerializer):
    """Serialize Explicit User Fields."""
    class Meta:
        """Define Serializer Metadata."""
        model = User
        fields: list[str] = ["id", "mobile", "name"]
```

Wrong:

```python
class UserSerializer(ModelSerializer):
    """Serialize Explicit User Fields."""
    class Meta:
        """Define Serializer Metadata."""
        model = User
        exclude: list[str] = ["password"]
```

Never allow new model fields to leak automatically.

---

## 14. Backend Logging

Use centralized logger creation.

```python
from project.log import logger_set

logger = logger_set("payment.view")

logger.info(msg={"message": "Payment Started", "user_id": user.id})
logger.warning(msg={"message": "Invalid Payment Attempt", "user_id": user.id})
```

Logger name format:

```text
<app>.<module>
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
Cron Execution
Celery Task Failure
```

---

## 15. Frontend Philosophy

Frontend code must be View-first.

Preferred flow:

```text
View
↓
Central apiCall
↓
Global Loading / Message
↓
Pinia Store For Shared State
↓
Reusable UI Components
```

Components are reusable visual pieces, not the default place for page business logic.

Good components:

```text
UserCard.vue
Message.vue
Loading.vue
ConfirmModal.vue
PriceBadge.vue
```

Avoid:

```text
DashboardEverything.vue
UserAndOrderAndPayment.vue
```

---

## 16. Frontend Layout

Classic existing structure is acceptable:

```text
src/
├── views/
├── components/
├── router/
├── store/
└── utils/
```

Preferred structure for new or cleaned code:

```text
src/
├── api/
│   ├── client.js
│   ├── call.js
│   ├── errors.js
│   └── services/
│       ├── auth.js
│       └── users.js
├── views/
├── components/
├── router/
│   ├── index.js
│   ├── routes.js
│   └── guards.js
├── stores/
├── validators/
├── constants/
├── utils/
├── styles/
└── env.js
```

Do not refactor old working structure unless the task requires it.

---

## 17. Frontend Naming

Use:

```text
PascalCase.vue For Views And Components
camelCase For Variables, Methods, And Route Names
UPPER_SNAKE For Constants
```

Examples:

```text
Dashboard.vue
UserCard.vue
ProfileDetail.vue
profileDetail
BACKEND_BASE_URL
```

---

## 18. Vue API Style

Existing code may use Options API.

```vue
<script>
export default {
    data() {
        return {
            mobile: null,
            stepTwo: false,
        };
    },
    methods: {
        async submitForm(event) {
            // Submit Form Data
            event.preventDefault();
        },
    },
};
</script>
```

For new Vue/Vite files, Options API is acceptable if it matches nearby code.

For Nuxt or modern isolated new code, `<script setup>` is acceptable, but do not mix styles inside one file without reason.

---

## 19. Frontend API Rule

All backend communication must pass through the central API helper.

Do not call axios directly inside views or components unless the current file has a special existing reason.

Preferred call:

```js
const result = await apiCall("/auth/", "POST", {}, {}, payload);

if (!result) return;
```

Better for new organized code:

```js
import {loginUser} from "/src/api/services/auth.js";

const result = await loginUser(payload);

if (!result) return;
```

Frontend must respect backend prefix:

```text
Same Origin API Prefix: /server/
```

So `BACKEND_BASE_URL` is commonly:

```js
export const BACKEND_BASE_URL = "/server";
```

or an absolute backend URL if the deployment is split.

---

## 20. Frontend API Client Duties

The central API layer must handle:

```text
Base URL
Authorization Header
JSON Response
HTML Response
Blob Response If Needed
FormData
Timeout
Network Errors
401 Redirect
Global Loading
Global Message
```

Every request returns:

```text
Object
Array
String
Blob
undefined
```

`undefined` means the error was already handled globally.

Example:

```js
export default async function apiCall(path, method = "GET", headers = {}, params = {}, data = {}) {
    const store = globalStore();

    activeRequestsCount++;
    store.isLoading = true;

    if (data instanceof FormData) {
        headers = {...headers, "Content-Type": undefined};
    }

    try {
        const res = await apiClient.request({
            url: path,
            method: method,
            headers: headers,
            params: params,
            data: data,
        });

        return res.data ? res.data : {};
    } catch (error) {
        console.error(error);
        return undefined;
    } finally {
        activeRequestsCount--;

        if (activeRequestsCount === 0) {
            store.isLoading = false;
        }
    }
}
```

---

## 21. Frontend Global Store

Pinia is the shared state hub.

Allowed in global store:

```text
User Data
Auth State
Loading State
Message State
Dark Mode
Sidebar State
Cart Count
Global Modal Flags
```

Do not store temporary form inputs in global store.

Required global fields:

```js
{
    isLoading: false,
    isDarkMode: false,
    message: {
        id: null,
        content: null,
        status: null,
    },
    userData: {
        isLoaded: false,
    },
}
```

---

## 22. Frontend Message Rule

All user messages must go through one function.

```js
export async function sendMessage(msg, status) {
    const store = globalStore();

    store.message.id = Date.now();
    store.message.content = msg;
    store.message.status = status;
}
```

Allowed statuses:

```text
primary
success
warning
danger
info
```

Never use `alert()`.

---

## 23. Frontend Validation Rule

Validate before API calls.

```js
const isValid = await verifyForm([
    {
        val: mobile,
        reg: /^09\d{9}$/,
        msg: "شماره موبایل صحیح نیست",
    },
    {
        val: password,
        reg: value => value && value.length >= 8,
        msg: "رمز عبور باید حداقل ۸ کاراکتر باشد",
    },
]);

if (!isValid) return;
```

Validation rules may support:

```text
RegExp
Function
Async Function
Custom Message
Custom Status
```

Use global Persian/Arabic digit normalization for numeric inputs.

---

## 24. Frontend Router Rules

Routes must be explicit.

```js
const Dashboard = () => import("/src/views/Dashboard.vue");
const Auth = () => import("/src/views/Auth.vue");

const routes = [
    {
        path: "/",
        component: Dashboard,
        name: "dashboard",
        meta: {requiresAuth: true},
    },
    {
        path: "/auth/",
        component: Auth,
        name: "auth",
        meta: {requiresAuth: false},
    },
];
```

Route guard must handle:

```text
Guest Redirect
Authenticated Redirect
User Data Loading
Invalid Token Cleanup
Next Query
```

Keep heavy business logic out of guards.

---

## 25. Frontend View Rules

Views may contain:

```text
Page Local State
Page Lifecycle
Calling API Functions
Validation Calls
Redirects
Passing Data To Components
```

Views must not contain:

```text
Raw Axios
Repeated HTTP Status Handling
Global Loading Control
Internal Backend Error Display
Large Reusable UI Pieces
```

Large views are acceptable if the workflow is page-specific and readable. Split visual chunks into components when the template becomes hard to scan.

---

## 26. Frontend Components

Use folder grouping for related UI components:

```text
components/
└── user/
    ├── Card.vue
    └── Avatar.vue
```

Use `UserCard.vue` when a component is globally meaningful.

Components should receive data through props and emit events for local interactions. Use store only for global state.

---

## 27. Frontend CSS

Prefer simple CSS, Bootstrap classes, Bootstrap Icons, and global CSS variables.

```text
styles/
├── variables.css
├── helpers.css
└── app.css
```

Component-specific CSS should be scoped unless intentionally global.

```vue
<style scoped>
.user-card {
    border-radius: var(--radius-md);
}
</style>
```

---

## 28. Frontend Security

Never trust frontend storage as real security.

Rules:

```text
Always Handle 401 Globally
Always Remove Invalid Token
Never Show devMessage To Normal Users
Never Store Secrets In env.js
Never Put Backend Secrets In Frontend
Always Protect Auth Routes
```

Local storage access should be centralized when possible.

---

## 29. Backend ↔ Frontend Error Contract

Backend should return user-facing Persian errors like:

```json
{
    "message": "شماره موبایل صحیح نیست"
}
```

For developer details:

```json
{
    "message": "اطلاعات وارد شده صحیح نیست",
    "devMessage": {
        "mobile": ["This Field Is Required"]
    }
}
```

Frontend may show `message`.

Frontend must not show `devMessage` to normal users.

Frontend API client should map common statuses:

```text
400 -> Persian Validation Or Business Message
401 -> Clear Token And Redirect To Auth
403 -> Permission Message
429 -> Rate Limit Message
5xx -> Generic Server Message
Network -> Internet/Connection Message
```

---

## 30. Deployment File Rules

### Dockerfile

Keep the Dockerfile as a single fullstack image unless asked otherwise.

Build order:

```text
Install Linux Packages
Install Python Requirements
Install Vue Dependencies
Build Vue With Vite
Copy Supervisor And Nginx Config
Run Entrypoint
Expose 80
```

Do not remove:

```text
python:3.12-slim
PYTHONUNBUFFERED=1
requirements.txt
/client npm install
npx vite build
nginx
supervisor
supercronic
```

### Nginx

Preserve these routes:

```text
/         -> Vue SPA
/server/  -> Django API
/static/  -> Django Static
/assets/  -> Shared Assets
```

Do not break SPA fallback:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Do not remove `/server` redirect:

```nginx
location = /server {
    return 301 /server/;
}
```

### Supervisor

Keep process priorities clear:

```text
django         priority=1
celery-worker  priority=2
cron           priority=3
nginx          priority=4
```

Use stdout/stderr logs and `/var/log/xxx` unless the project name has been replaced globally.

### Entrypoint

Keep migrations and static collection before process start:

```bash
python manage.py migrate --skip-checks
python manage.py collectstatic --no-input --skip-checks
exec "$@"
```

---

## 31. Comment And Docstring Style

All code comments must be English.

Use Capital Words Case.

Good:

```python
# Validate User Access
```

```js
// Load User Data Before Entering Authenticated Pages
```

Bad:

```python
# validate user access
```

```js
// loads userdata before auth
```

Do not write long noisy comments. Do not write code with zero comments. Use comments as section labels or intent markers.

Python docstring rules are mandatory:

- Every Python class must have a docstring.
- Every Python class method, instance method, static method, and classmethod must have a docstring.
- Python files/modules must not have module-level docstrings.
- Never put a triple-quoted string at the top of a Python file.
- Keep docstrings short, useful, and responsibility-focused.

```python
class Service:
    """
    Handle Authentication API Requests.
    
    And More Description ...
    """

    def service_get(self, data: dict) -> dict:
        """
        Get Current User Data.
        
        And More Description ...
        """
        
        # Process 1 Title	
        process_1 ...
        ...
        ...
        
        # Process 2 Title	
    	process_2 ...
    	...
    	...
    	
        return data
```

Wrong:

```python
"""Authentication Views."""


class Service:
    def service_get(self, data: dict) -> dict:
        process_1 ...
        ...
        ...
    	process_2 ...
    	...
    	...
    	
        return data
```

---

## 32. What Not To Do

Do not:

- Add repository pattern to Django code by default.
- Move Django business methods out of models without a reason.
- Add many tiny custom exceptions.
- Add dependency injection containers.
- Use Auto Import as a core style.
- Return raw ORM models from APIs.
- Use serializer `exclude`.
- Scatter constants inside views or components.
- Use raw HTTP status integers.
- Use direct axios in views.
- Manually toggle global loading around API calls.
- Store page form state in Pinia.
- Show backend internal errors in frontend.
- Break `/server/` proxy behavior.
- Split the one-container deployment without explicit instruction.

---

# Overcheck Checklist

Before returning any code, verify every item below.

## Fullstack Contract

- [ ] Vue and Python are treated as one source, not unrelated projects.
- [ ] Frontend calls backend through the central API layer.
- [ ] Same-origin backend prefix `/server/` is respected when applicable.
- [ ] Backend error format uses `message` for users and `devMessage` for developers.
- [ ] No internal backend details are displayed to normal frontend users.
- [ ] Deployment still serves Vue at `/` and backend at `/server/`.

## Python Backend

- [ ] Business logic is on model methods when it belongs to model/data behavior.
- [ ] `services/` is used for external integrations, not random business logic.
- [ ] `tools/` has no Django dependency.
- [ ] `utils/` contains Django-aware helpers.
- [ ] All function parameters and return values are type annotated.
- [ ] Important variables are type annotated.
- [ ] Guard clauses are used instead of deep `if/else`.
- [ ] Constants are not scattered inline.
- [ ] HTTP statuses use `status.HTTP_*`, not raw integers.
- [ ] Serializers use explicit `fields`, never `exclude`.
- [ ] Exceptions are not over-split into many tiny classes.
- [ ] Logs are structured and include useful context.
- [ ] Comments are English and Capital Words Case.
- [ ] Every Python class has a docstring.
- [ ] Every Python instance method, static method, and classmethod has a docstring.
- [ ] No Python file/module starts with a module-level docstring.
- [ ] Code remains readable even if not extremely short.

## Vue Frontend

- [ ] Views remain page/workflow owners.
- [ ] Reusable UI is extracted into components when the template becomes hard to scan.
- [ ] API calls go through `apiCall` or an API service that uses `apiCall`.
- [ ] No raw axios is added to views/components.
- [ ] Global loading is controlled by the API layer.
- [ ] Global messages go through `sendMessage`.
- [ ] Forms validate with `verifyForm` before requests.
- [ ] Persian/Arabic numeric inputs are normalized when needed.
- [ ] Pinia stores only shared/global state.
- [ ] Temporary form state stays in the view.
- [ ] Routes are explicit and named.
- [ ] Auth routes use `meta.requiresAuth`.
- [ ] 401 clears token and redirects to auth.
- [ ] CSS stays simple, scoped when component-specific, and Bootstrap-friendly.
- [ ] Comments are English and Capital Words Case.

## Deployment

- [ ] Dockerfile still builds Python dependencies and Vue assets.
- [ ] `client/dist` remains the Nginx SPA root.
- [ ] Gunicorn still binds Django to port `3000`.
- [ ] Nginx proxies `/server/` to `127.0.0.1:3000`.
- [ ] Static files are served from `/static/`.
- [ ] Shared assets are served from `/assets/`.
- [ ] Supervisor still starts Django, Celery worker, Cron, and Nginx.
- [ ] Entrypoint still runs migrate and collectstatic.
- [ ] Logs still go to stdout/stderr or the configured log directory.
- [ ] No accidental Docker Compose or multi-container assumption was introduced.

## Final Agent Self-Check

- [ ] Did I preserve the existing style instead of forcing my favorite architecture?
- [ ] Did I keep the code explicit and inspectable?
- [ ] Did I avoid hidden magic?
- [ ] Did I avoid unnecessary abstractions?
- [ ] Did I keep the backend model-centric?
- [ ] Did I keep the frontend view-first?
- [ ] Did I centralize API, loading, messages, and validation?
- [ ] Did I protect the `/server/` deployment contract?
- [ ] Would a junior developer understand this code quickly?
