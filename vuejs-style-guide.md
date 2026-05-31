# Vue.js Code Style Guide

---

## Philosophy

Project structure must be feature-clear, modular, and predictable.

The frontend communicates with backend only through one centralized API layer.

Views must stay thin.

Business logic must be moved to services, composables, stores, or utilities.

---

## Directory Layout

```text
src/
├── assets/          — Images, Fonts, Static Files
├── components/      — Reusable UI Components
├── layouts/         — Central Page Layouts
├── views/           — Route Pages
├── router/          — Vue Router Config
├── stores/          — Pinia Stores
├── api/             — API Client, API Call, API Services
├── services/        — Frontend Business Logic
├── composables/     — Reusable Composition Functions
├── plugins/         — Vue Plugins
├── utils/           — General Utilities
├── validators/      — Form Validation Rules
├── constants/       — Enums And Constants
├── pwa/             — PWA Registration And Service Worker Helpers
├── styles/          — Global CSS, Variables, Theme
├── tests/           — Unit And Component Tests
├── env.js           — Runtime Environment Variables
└── main.js          — Vue Entry Point
```

---

## Preferred Structure

```text
src/
├── api/
│   ├── client.js
│   ├── call.js
│   ├── errors.js
│   └── services/
│       ├── auth.js
│       ├── users.js
│       └── orders.js
│
├── components/
│   ├── Message.vue
│   ├── Loading.vue
│   ├── EmptyState.vue
│   └── ConfirmModal.vue
│
├── layouts/
│   ├── MainLayout.vue
│   └── AuthLayout.vue
│
├── views/
│   ├── Dashboard.vue
│   ├── Auth.vue
│   └── NotFound.vue
│
├── stores/
│   ├── global.js
│   ├── auth.js
│   └── cart.js
│
├── router/
│   ├── index.js
│   ├── routes.js
│   └── guards.js
│
├── plugins/
│   ├── normalize-digits.js
│   ├── number-format.js
│   └── bootstrap.js
│
├── validators/
│   ├── form.js
│   ├── rules.js
│   └── regex.js
│
├── constants/
│   ├── ENUMS.js
│   ├── ROUTES.js
│   └── MESSAGES.js
│
├── utils/
│   ├── public.js
│   ├── storage.js
│   ├── messages.js
│   └── formatters.js
│
├── styles/
│   ├── variables.css
│   ├── helpers.css
│   └── app.css
│
├── pwa/
│   └── register.js
│
├── env.js
└── main.js
```

---

## Naming

| Entity       | Convention              | Example               |
| ------------ | ----------------------- | --------------------- |
| Views        | `Name.vue`              | `Dashboard.vue`       |
| Components   | `Name.vue`              | `Loading.vue`         |
| Layouts      | `NameLayout.vue`        | `MainLayout.vue`      |
| Stores       | `useNameStore`          | `useGlobalStore`      |
| API Services | Feature Name            | `auth.js`             |
| Validators   | Feature Or General Name | `form.js`, `rules.js` |
| Constants    | `UPPER_SNAKE`           | `MAX_UPLOAD_SIZE`     |
| Route Names  | `camelCase`             | `profileDetail`       |
| CSS Classes  | `kebab-case`            | `page-wrapper`        |

---

## Comments

Write comments in English.

Format: `# Capital Words Case`

```js
// --- Authentication Section ---

// Validate Token And Redirect User
async function checkAuth() {
    ...
}
```

For large partial sections:

```js
// ==============================
// API Error Handling Section
// ==============================
```

---

## API Layer

All backend communication must pass through one central `apiCall`.

Views and components must not use `axios` directly.

Correct:

```js
import apiCall from "/src/api/call.js";

const result = await apiCall("/users/me/");
```

Wrong:

```js
import axios from "axios";

const result = await axios.get("/users/me/");
```

---

## API Client Rules

The API layer must handle:

```text
Base URL
Authorization Header
JSON Response
HTML Response
FormData
Timeout
Network Errors
401 Redirect
Global Loading
Global Message
```

Every request must return one of these:

```text
Object
Array
String
Blob
undefined
```

`undefined` means request failed and was already handled globally.

---

## API Services

Views should not contain raw endpoint paths.

Correct:

```js
import {getCurrentUser} from "/src/api/services/auth.js";

const user = await getCurrentUser();
```

Service example:

```js
import apiCall from "/src/api/call.js";

export async function getCurrentUser() {
    // Get Current User Data
    return await apiCall("/auth/me/");
}

export async function loginUser(payload) {
    // Send Login Data
    return await apiCall("/auth/login/", "POST", {}, {}, payload);
}
```

---

## Store Rules

Stores must hold global or shared state only.

Good store state:

```text
User Data
Auth Token State
Loading State
Message State
Dark Mode State
Cart State
Global Settings
```

Bad store state:

```text
Temporary Form Input
One-Time Modal Data
Pure Page Local State
```

Local page state must stay inside the component.

---

## Global Store Required Fields

```js
{
    isLoading: false,
    isDarkMode: false,
    message: {
        id: null,
        content: "",
        status: "primary",
    },
    user: null,
}
```

---

## Loading Pattern

Loading must be controlled only by the centralized API layer.

Do not manually enable or disable global loading inside views unless there is no API request involved.

Correct:

```js
const data = await apiCall("/records/");
```

Wrong:

```js
store.isLoading = true;
const data = await apiCall("/records/");
store.isLoading = false;
```

---

## Message Pattern

All user messages must pass through one global message function.

Correct:

```js
await sendMessage("عملیات با موفقیت انجام شد", "success");
```

Wrong:

```js
alert("Done");
```

Allowed statuses:

```text
primary
success
warning
danger
info
```

---

## Form Validation

Forms must be validated with object-based rules.

Validation must happen before API call.

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

Rules may support:

```text
RegExp
Function
Async Function
Custom Message
Custom Status
```

---

## Persian Digit Normalization

Numeric inputs must normalize Persian and Arabic digits to English digits.

Use global plugin.

Only numeric fields should be normalized.

Correct:

```html
<input v-model="mobile" inputmode="numeric">
```

Wrong:

```html
<input v-model="name" inputmode="numeric">
```

---

## Number Formatting

Use directive for visual formatting only.

Do not mutate business values for display.

Correct:

```html
<span v-num-format="price"></span>
```

Wrong:

```js
price = price.toLocaleString("en-US");
```

---

## Router Rules

Routes must be split from guards.

```text
router/routes.js
router/guards.js
router/index.js
```

Routes must have explicit names.

Authenticated routes must use:

```js
meta: {requiresAuth: true}
```

Public routes must not rely on missing meta accidentally.

Use:

```js
meta: {requiresAuth: false}
```

---

## Route Naming

```js
{
    path: "/profiles/:token/",
    name: "profileDetail",
    component: ProfileDetail,
    meta: {requiresAuth: true},
}
```

Avoid unnamed routes except final 404 route.

---

## Router Guards

Router guard must handle:

```text
Guest Redirect
Authenticated Redirect
User Data Loading
Invalid Token Cleanup
Next Query
```

Never put heavy business logic in router guards.

Move logic to auth store or auth service.

---

## Layout Rules

Every page must use a layout.

Central layout may include:

```text
Loading
Message
Floating Actions
Dark Mode Toggle
Global Modals
Router Slot
```

Views must not manually import global UI components like Loading or Message.

---

## Component Rules

Components must be small and reusable.

A component should do one job.

Good:

```text
UserCard.vue
ConfirmModal.vue
Loading.vue
Message.vue
PriceBadge.vue
```

Bad:

```text
DashboardEverything.vue
UserAndOrderAndPayment.vue
```

---

## View Rules

Views may contain:

```text
Page Local State
Page Lifecycle
Calling API Services
Passing Data To Components
```

Views must not contain:

```text
Raw Axios
Complex Validation Logic
Backend Error Mapping
Large Business Logic
Global Loading Control
```

---

## Composition API Preference

For new code, prefer Composition API with `<script setup>`.

Correct:

```vue
<script setup>
import {ref, onMounted} from "vue";
import {getCurrentUser} from "/src/api/services/auth.js";

const user = ref(null);

onMounted(async () => {
    // Load User On Page Mount
    user.value = await getCurrentUser();
});
</script>
```

Options API is allowed only for old files or very simple global components.

---

## Template Rules

Templates must stay readable.

Avoid complex inline expressions.

Correct:

```html
<span>{{ formattedPrice }}</span>
```

Wrong:

```html
<span>{{ Number(price || 0).toLocaleString("en-US").replace(",", "،") }}</span>
```

---

## Computed Rules

Use computed values for derived display data.

```js
const fullName = computed(() => {
    // Build User Full Name
    return "{} {}".format(user.value.firstName, user.value.lastName);
});
```

Do not store derived values as separate state unless needed.

---

## Error Handling

Backend error messages must be normalized in API error handler.

Components and views should not repeat status-code handling.

Correct:

```js
const result = await loginUser(payload);

if (!result) return;
```

Wrong:

```js
try {
    ...
} catch (error) {
    if (error.response.status === 401) {
        ...
    }
}
```

---

## Response Message Rules

Frontend expects backend error format:

```js
{
    message: "پیام مناسب کاربر",
    devMessage: {}
}
```

Frontend may show `message`.

Frontend must not show `devMessage` to normal users.

`devMessage` may be logged only in development mode.

---

## Constants And Enums

All magic values must be placed in constants.

```js
export const MESSAGE_STATUS_SUCCESS = "success";
export const MESSAGE_STATUS_DANGER = "danger";

export const AUTH_TOKEN_KEY = "auth-token";

export const HTTP_STATUS_UNAUTHORIZED = 401;
```

Never scatter repeated strings in components.

Wrong:

```js
localStorage.getItem("auth-token");
```

Correct:

```js
localStorage.getItem(AUTH_TOKEN_KEY);
```

---

## Local Storage

Direct `localStorage` usage is allowed only inside storage utilities or auth store.

Correct:

```js
import {getAuthToken} from "/src/utils/storage.js";

const token = getAuthToken();
```

Wrong:

```js
const token = localStorage.getItem("auth-token");
```

---

## String Formatting

Prefer template literals only for UI strings.

For predictable keys and system strings, use `.format()` helper or string join pattern.

Correct:

```js
const cacheKey = "{}{}".format(USER_CACHE_PREFIX, userId);
```

Also acceptable:

```js
const cacheKey = [USER_CACHE_PREFIX, userId].join("");
```

---

## CSS Rules

Use global CSS variables for theme colors.

Component styles must be scoped unless intentionally global.

Correct:

```vue
<style scoped>
.user-card {
    border-radius: var(--radius-md);
}
</style>
```

Global styles must live in:

```text
styles/app.css
styles/variables.css
styles/helpers.css
```

---

## Dark Mode

Dark mode must be controlled from global store.

The store must:

```text
Read Saved Preference
Apply Class To Root Element
Toggle Mode
Persist Preference
```

Views must not directly change root classes.

---

## PWA Rules

PWA registration must live outside `main.js`.

Use:

```text
pwa/register.js
```

PWA logs must be clear and minimal.

On update, either:

```text
Show Update Message
Ask User To Refresh
```

or for admin panels:

```text
Reload Automatically
```

Never silently ignore new versions.

---

## Imports

Use absolute imports from `/src`.

Correct:

```js
import apiCall from "/src/api/call.js";
import {verifyForm} from "/src/validators/form.js";
```

Wrong:

```js
import apiCall from "../../../api/call.js";
```

---

## Security Rules

Frontend security is not enough, but must still be clean.

Checklist:

```text
Never Trust Local Storage Alone
Never Show Internal Errors
Never Store Sensitive Data In Store Unless Needed
Never Put Secrets In env.js
Always Handle 401 Globally
Always Remove Invalid Token
Always Protect Auth Routes
```

---

## File Upload

File upload must use `FormData`.

Do not manually set `Content-Type` for `FormData`.

API layer must remove or ignore manual content type so browser can set boundary.

```js
const formData = new FormData();
formData.append("file", file);

const result = await apiCall("/upload/", "POST", {}, {}, formData);
```

---

## HTML Response

If backend returns HTML, API layer must wrap it:

```js
{
    isHTML: true,
    contents: "<html>...</html>"
}
```

Components must check `isHTML` before rendering.

Never render unknown HTML with `v-html` unless source is trusted.

---

## Suggested Base Files

### `api/client.js`

```js
import axios from "axios";

import {BACKEND_BASE_URL} from "/src/env.js";

const apiClient = axios.create({
    baseURL: BACKEND_BASE_URL,
    timeout: 30000,
    headers: {
        "Content-Type": "application/json",
    },
});

export default apiClient;
```

---

### `api/services/auth.js`

```js
import apiCall from "/src/api/call.js";

export async function getCurrentUser() {
    // Get Current User Data
    return await apiCall("/auth/me/");
}

export async function loginUser(payload) {
    // Login User With Credentials
    return await apiCall("/auth/login/", "POST", {}, {}, payload);
}

export async function logoutUser() {
    // Logout Current User
    return await apiCall("/auth/logout/", "POST");
}
```

---

### `utils/storage.js`

```js
import {AUTH_TOKEN_KEY} from "/src/constants/ENUMS.js";

export function getAuthToken() {
    // Get Auth Token From Storage
    return localStorage.getItem(AUTH_TOKEN_KEY);
}

export function setAuthToken(token) {
    // Save Auth Token In Storage
    localStorage.setItem(AUTH_TOKEN_KEY, token);
}

export function removeAuthToken() {
    // Remove Auth Token From Storage
    localStorage.removeItem(AUTH_TOKEN_KEY);
}
```
