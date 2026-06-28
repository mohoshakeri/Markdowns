# Frontend Agent README

This document defines how an AI agent must write Vue, Vite, and Nuxt frontend code for this developer's style.

The goal is a predictable, explicit, page-first frontend with centralized API, loading, messages, validation, and routing behavior.

---

## 1. Core Personality Of The Code

Write frontend code that is:

- Explicit, not magical.
- Page/View-first.
- Component-based only for reusable UI pieces.
- Centralized for API, loading, messages, auth failure handling, and validation.
- Pinia-based for shared state.
- CSS-simple and Bootstrap-friendly unless the project says otherwise.
- Direct and readable, not pattern-heavy.
- Easy for a junior developer to follow.

Do not over-engineer with deep composables, repositories, class services, or hidden auto-magic.

---

## 2. Preferred Vue/Vite Structure

Use this structure for classic Vue projects:

```text
src/
├── assets/
├── components/
│   ├── Message.vue
│   ├── Loading.vue
│   ├── EmptyState.vue
│   └── ConfirmModal.vue
├── layouts/
│   ├── MainLayout.vue
│   └── SingleLayout.vue
├── views/
│   ├── Dashboard.vue
│   ├── Auth.vue
│   └── NotFound.vue
├── router/
│   ├── index.js
│   ├── routes.js
│   └── guards.js
├── stores/
│   ├── global.js
│   └── auth.js
├── api/
│   ├── client.js
│   ├── call.js
│   └── services/
│       ├── auth.js
│       ├── users.js
│       └── payments.js
├── utils/
│   ├── public.js
│   ├── storage.js
│   └── formatters.js
├── validators/
│   ├── form.js
│   └── regex.js
├── constants/
│   ├── ENUMS.js
│   ├── ROUTES.js
│   └── MESSAGES.js
├── styles/
│   ├── variables.css
│   ├── helpers.css
│   └── app.css
├── pwa/
│   └── register.js
├── env.js
└── main.js
```

Legacy projects may keep the compact structure:

```text
src/
├── views/
├── components/
├── utils/
│   ├── client.js
│   ├── store.js
│   ├── router.js
│   └── public.js
├── env.js
└── main.js
```

When editing an existing project, follow the current structure. Do not rewrite the architecture without being asked.

---

## 3. Preferred Nuxt Structure

Use Nuxt conventions only in Nuxt projects:

```text
app/
├── components/
├── composables/
├── constants/
├── layouts/
├── middleware/
├── pages/
├── plugins/
├── stores/
├── styles/
├── utils/
└── app.vue

server/
├── api/
├── middleware/
└── utils/

public/
├── images/
├── icons/
└── robots.txt
```

Nuxt public pages must be SEO-friendly and use one reusable SEO helper.

---

## 4. Responsibility Map

| Location | Responsibility |
|---|---|
| `views/` / `pages/` | Page State, Page Workflow, Form Flow, Calling API Helpers, Passing Data To Components |
| `components/` | Reusable UI Only: Cards, Buttons, Modals, Message, Loading, Empty State |
| `api/` or `utils/client.js` | Axios Client, Auth Header, Loading, Error Handling, 401 Redirect, HTML/FormData Handling |
| `api/services/` | Optional Feature API Wrappers Like `getCurrentUser` |
| `stores/` | Global Shared State: User, Loading, Message, Dark Mode, Sidebar, Cart |
| `utils/` | Public Helpers, Formatting, Storage, Messages |
| `validators/` | Reusable Form Validation Rules |
| `constants/` | Enums, Routes, LocalStorage Keys, Status Strings |
| `router/` | Route Records And Guards |
| `plugins/` | Global Input Normalizers, Number Formatting, Bootstrap/PWA Setup |

---

## 5. Views Are Workflow Owners

A View may contain local state, page lifecycle, form validation calls, API calls, redirects, and screen-specific formatting.

Correct:

```vue
<script>
import SingleLayout from "/src/components/SingleLayout.vue";
import {loginUser} from "/src/api/services/auth.js";
import {sendMessage, verifyForm} from "/src/utils/public.js";
import router from "/src/router/index.js";

export default {
    components: {SingleLayout},
    data: () => ({
        mobile: null,
        code: null,
    }),
    methods: {
        async loginRequest(event) {
            event.preventDefault();

            const isValid = await verifyForm([
                {
                    val: this.mobile,
                    reg: /^09\d{9}$/,
                    msg: "شماره موبایل صحیح نیست",
                },
            ]);
            if (!isValid) return;

            const res = await loginUser({mobile: this.mobile, code: this.code});
            if (!res) return;

            localStorage.setItem("auth-token", res.token);
            await sendMessage("ورود با موفقیت انجام شد", "success");
            router.push({name: "dashboard"});
        },
    },
};
</script>
```

Do not move every page-specific action into a composable just to make the view shorter.

---

## 6. Component Rules

Components are UI pieces, not business-process owners.

Good:

```text
UserCard.vue
Avatar.vue
Loading.vue
Message.vue
PriceBadge.vue
ConfirmModal.vue
```

Avoid:

```text
DashboardEverything.vue
PaymentAndUserAndOrderManager.vue
```

Split a 600-line component into smaller components when it contains separate visual sections or repeated UI blocks.

Prefer folder grouping when a component family grows:

```text
components/user/
├── UserCard.vue
├── UserAvatar.vue
└── UserBadge.vue
```

PascalCase file names are preferred.

---

## 7. API Layer Is Mandatory

Views and components must not call `axios` directly.

All backend calls must pass through one central API layer:

```js
import apiCall from "/src/api/call.js";

const data = await apiCall("/auth/", "GET", {}, {}, {});
```

The API layer must handle:

```text
Base URL
Authorization Header
JSON Response
HTML Response
Blob Response
FormData
Timeout
Network Errors
401 Redirect
Global Loading
Global Message
```

Return values:

```text
Object | Array | String | Blob | undefined
```

`undefined` means the error was already handled globally.

---

## 8. API Call Example

```js
import axios from "axios";

import {BACKEND_BASE_URL} from "/src/env.js";
import globalStore from "/src/stores/global.js";
import {sendMessage} from "/src/utils/public.js";

const apiClient = axios.create({
    baseURL: BACKEND_BASE_URL,
    timeout: 30000,
    headers: {"Content-Type": "application/json"},
});

let activeRequestsCount = 0;

apiClient.interceptors.request.use((config) => {
    const raw = localStorage.getItem("auth-token");
    if (raw) {
        config.headers.Authorization = /^Bearer\s+/i.test(raw) ? raw : `Bearer ${raw}`;
    }
    return config;
});

apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
        if (!error.response) {
            await sendMessage("ارتباط با سرور برقرار نشد. اتصال اینترنت را بررسی کنید.", "danger");
            return Promise.reject(error);
        }

        if (error.response.status === 401) {
            localStorage.removeItem("auth-token");
            await sendMessage("نشست شما منقضی شده است. مجددا وارد شوید.", "warning");
        }

        return Promise.reject(error);
    },
);

export default async function apiCall(path, method = "GET", headers = {}, params = {}, data = {}) {
    const store = globalStore();

    activeRequestsCount++;
    store.isLoading = true;

    if (data instanceof FormData) {
        headers = {...headers, "Content-Type": undefined};
    }

    try {
        const res = await apiClient.request({url: path, method, headers, params, data});
        const contentType = res.headers["content-type"] || "";

        if (contentType.includes("application/json")) return res.data || {};
        if (contentType.includes("text/html")) return {isHTML: true, contents: res.data};

        return res.data;
    } catch (e) {
        console.error(e);
        return undefined;
    } finally {
        activeRequestsCount--;
        if (activeRequestsCount === 0) store.isLoading = false;
    }
}
```

---

## 9. API Services

Use feature API files when the project has `api/services/`.

```js
import apiCall from "/src/api/call.js";

export async function getCurrentUser() {
    // Get Current User Data
    return await apiCall("/auth/", "GET", {}, {}, {});
}

export async function loginUser(payload) {
    // Send Login Payload
    return await apiCall("/auth/", "POST", {}, {}, payload);
}
```

For smaller legacy projects, raw endpoint paths inside views are acceptable if all requests still use `apiCall`.

---

## 10. Global Store

Use Pinia for shared app state only.

Required global state:

```js
import {defineStore} from "pinia";

const globalStore = defineStore("global", {
    state: () => ({
        message: {
            id: null,
            content: null,
            status: null,
        },
        userData: {
            wallet: 0,
            keys: 0,
            isLoaded: false,
        },
        isLoading: false,
        isDarkMode: localStorage.getItem("dark-mode") === "true",
        sideBarShow: true,
    }),
    actions: {
        resetUserData() {
            this.userData.wallet = 0;
            this.userData.keys = 0;
            this.userData.isLoaded = false;
        },
    },
});

export default globalStore;
```

Good store state:

```text
User Data
Auth State
Loading
Message
Dark Mode
Sidebar
Cart Count
Global Modals
```

Bad store state:

```text
Temporary Form Inputs
Page-Only Filters
One-Time Local Modal State
```

---

## 11. Global Message Pattern

All user messages must go through one helper.

```js
import globalStore from "/src/stores/global.js";

export async function sendMessage(msg, status) {
    // Send Global User Message
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

## 12. Validation Pattern

Forms must validate before API calls.

```js
export async function verifyForm(rules) {
    // Validate Form Rules
    for (const rule of rules) {
        let ok = false;

        try {
            if (rule.reg instanceof RegExp) {
                const value = rule.val == null ? "" : String(rule.val);
                ok = rule.reg.test(value);
            } else if (typeof rule.reg === "function") {
                const res = rule.reg(rule.val);
                ok = res instanceof Promise ? await res : res;
            }
        } catch (_) {
            ok = false;
        }

        if (!ok) {
            await sendMessage(rule.msg || "ورودی نامعتبر است.", rule.status || "danger");
            return false;
        }
    }

    return true;
}
```

Use this in views:

```js
const isValid = await verifyForm([
    {
        val: this.mobile,
        reg: /^09\d{9}$/,
        msg: "شماره موبایل صحیح نیست",
    },
]);
if (!isValid) return;
```

---

## 13. Persian And Arabic Digit Normalization

Numeric inputs must normalize Persian and Arabic digits to English digits globally.

Use a plugin or shared input handler.

Only numeric fields should be normalized.

Correct:

```html
<input v-model="mobile" inputmode="numeric" dir="ltr" />
```

Wrong:

```html
<input v-model="name" inputmode="numeric" />
```

---

## 14. Router Rules

Routes must be explicit and named.

```js
import {createRouter, createWebHistory} from "vue-router";

const Dashboard = () => import("/src/views/Dashboard.vue");
const Auth = () => import("/src/views/Auth.vue");
const NotFound = () => import("/src/views/NotFound.vue");

const routes = [
    {path: "/", component: Dashboard, name: "dashboard", meta: {requiresAuth: true}},
    {path: "/auth/", component: Auth, name: "auth", meta: {requiresAuth: false}},
    {path: "/:pathMatch(.*)*", component: NotFound, name: "notFound", meta: {requiresAuth: false}},
];

const router = createRouter({
    history: createWebHistory(),
    routes,
});

export default router;
```

Guard must handle:

```text
Guest Redirect
Authenticated Redirect
User Data Loading
Invalid Token Cleanup
Next Query
```

Keep business logic out of router guards.

---

## 15. Options API Vs Composition API

For existing Vue/Vite projects, Options API is preferred unless the project already uses Composition API.

Preferred for this style:

```vue
<script>
export default {
    data: () => ({
        items: [],
        isReady: false,
    }),
    methods: {
        async loadItems() {
            // Load Items From Api
            this.items = await getItems();
            this.isReady = true;
        },
    },
};
</script>
```

For Nuxt or new projects that already use `<script setup>`, follow the project convention. Do not mix styles randomly inside the same project.

---

## 16. CSS Rules

Prefer:

```text
CSS Simple
Bootstrap Classes
Bootstrap Icons
Global CSS Variables
Scoped Component CSS When Needed
```

Global styles:

```text
styles/variables.css
styles/helpers.css
styles/app.css
```

Do not introduce Tailwind, SCSS, CSS-in-JS, or a custom design system unless explicitly asked.

---

## 17. Naming Rules

| Entity | Convention | Example |
|---|---|---|
| Views | PascalCase `.vue` | `Dashboard.vue` |
| Components | PascalCase `.vue` | `UserCard.vue` |
| Layouts | PascalCase + `Layout` | `SingleLayout.vue` |
| Router Names | camelCase | `profileDetail` |
| Store Files | camel/simple | `global.js`, `auth.js` |
| Utility Files | simple lower/camel | `client.js`, `public.js` |
| Constants | UPPER_SNAKE | `AUTH_TOKEN_KEY` |
| CSS Classes | kebab-case | `single-card` |

---

## 18. Imports

Use absolute imports from `/src` in Vue/Vite:

```js
import apiCall from "/src/api/call.js";
import globalStore from "/src/stores/global.js";
```

Use Nuxt alias in Nuxt:

```js
import {useApiCall} from "~/composables/useApiCall";
```

Avoid deep relative imports:

```js
import apiCall from "../../../api/call.js";
```

---

## 19. Backend Error Contract

Frontend expects backend errors like:

```js
{
    message: "پیام مناسب کاربر",
    devMessage: {},
}
```

Rules:

- Show `message` to user.
- Do not show `devMessage` to normal users.
- Log `devMessage` only in development mode.
- Handle status codes in API layer, not inside every View.

---

## 20. File Upload

Use `FormData`.

Do not manually set multipart `Content-Type`.

Correct:

```js
const formData = new FormData();
formData.append("file", file);

const result = await apiCall("/upload/", "POST", {}, {}, formData);
```

The API layer must remove manual `Content-Type` so the browser sets the boundary.

---

## 21. HTML Responses

If backend returns HTML, wrap it:

```js
{
    isHTML: true,
    contents: "<html>...</html>",
}
```

Never render unknown HTML with `v-html` unless the source is trusted.

---

## 22. Nuxt SEO Rules

For public Nuxt pages, use one SEO helper.

```js
useSeoPage({
    title: article.title,
    description: article.description,
    path: `/blog/${article.slug}`,
    image: article.image,
});
```

It must generate:

```text
Title
Description
Canonical URL
OpenGraph
Twitter Card
Schema.org
```

Do not repeat SEO boilerplate manually in every page.

---

## 23. Avoid These

Do not generate:

- Direct `axios` calls inside views or components.
- `alert()` messages.
- Manual global loading in views when `apiCall` is involved.
- Hidden auto-import assumptions.
- Deep feature-folder architecture unless project already uses it.
- Class-based frontend services.
- Dependency injection containers.
- Complex functional programming style.
- Tailwind or SCSS unless requested.
- Large business logic inside reusable UI components.
- Store state for temporary form inputs.
- Unnamed routes.
- Scattered localStorage strings.
- Unhandled 401 behavior.

---

# Overcheck Checklist

Before finalizing any frontend change, verify every item:

## Structure

- [ ] File is in the correct layer: view/page, component, api, store, router, util, validator, constant, style, or plugin.
- [ ] Existing project structure is respected.
- [ ] Components are reusable UI pieces, not business workflow containers.
- [ ] Page-specific state stays in the View/Page.
- [ ] Shared app state is in Pinia only when truly shared.

## API

- [ ] No direct `axios` call exists in a View or Component.
- [ ] Every backend request goes through `apiCall` or an API service using `apiCall`.
- [ ] API layer handles auth header, loading, messages, network errors, timeout, 401, JSON, HTML, Blob, and FormData.
- [ ] `undefined` is treated as a globally handled API failure.
- [ ] FormData does not manually set multipart `Content-Type`.

## Loading And Messages

- [ ] Global loading is controlled by API layer request count.
- [ ] Global messages go through `sendMessage`.
- [ ] No `alert()` is used.
- [ ] Allowed message status is used: `primary`, `success`, `warning`, `danger`, `info`.

## Validation

- [ ] Form validation runs before API calls.
- [ ] `verifyForm` uses object rules with `val`, `reg`, `msg`, and optional `status`.
- [ ] Validation errors are shown through global message.
- [ ] Numeric inputs normalize Persian and Arabic digits when needed.

## Router

- [ ] Routes are explicit and named.
- [ ] Auth routes use `meta: {requiresAuth: true}`.
- [ ] Public routes explicitly use `requiresAuth: false` when practical.
- [ ] Guard handles invalid token cleanup.
- [ ] Guard preserves `next` query for redirect after login.
- [ ] Heavy business logic is not inside router guard.

## Store

- [ ] Store contains only shared state.
- [ ] Temporary form inputs are not stored globally.
- [ ] User data has a clear loaded/reset flow.
- [ ] Dark mode is controlled only from global store.
- [ ] Sidebar/body-scroll logic is centralized if used.

## UI And CSS

- [ ] PascalCase `.vue` files are used.
- [ ] CSS remains simple and readable.
- [ ] Bootstrap classes/icons are used consistently when the project uses Bootstrap.
- [ ] Complex template expressions are moved to computed/methods.
- [ ] No new design system or Tailwind setup was introduced without request.

## Security

- [ ] Frontend does not trust localStorage alone.
- [ ] Invalid tokens are removed globally.
- [ ] Internal `devMessage` is not shown to normal users.
- [ ] No secrets are placed in frontend env files.
- [ ] Unknown HTML is not rendered with `v-html`.

## Nuxt Only

- [ ] Public pages define SEO through `useSeoPage`.
- [ ] Canonical URL, OpenGraph, Twitter Card, and Schema.org are covered.
- [ ] Nuxt middleware handles auth where appropriate.
- [ ] Nuxt aliases are used instead of deep relative imports.

## Final Sanity

- [ ] The code looks like this developer's projects, not a generic Vue tutorial.
- [ ] Workflow is readable inside the owning View/Page.
- [ ] Cross-cutting behavior is centralized.
- [ ] No fashionable abstraction was added without real need.
- [ ] A junior developer can trace the behavior from route to view to API call.
