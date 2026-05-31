# Nuxt.js Code Style Guide

---

# Philosophy

Project structure must be feature-clear, modular, scalable, and SEO-friendly.

Frontend communicates with backend only through one centralized API layer.

Pages must stay thin.

Business logic must live inside:

- Services
- Composables
- Stores
- Utilities

Nuxt built-in features should be preferred over custom implementations whenever possible.

---

# Directory Layout

```text
app/
├── assets/
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

nuxt.config.ts
```

---

# Preferred Structure

```text
app/
├── components/
│   ├── Loading.vue
│   ├── Message.vue
│   ├── ConfirmModal.vue
│   └── EmptyState.vue
│
├── composables/
│   ├── useApiCall.js
│   ├── useSeoPage.js
│   └── usePagination.js
│
├── stores/
│   ├── auth.js
│   ├── global.js
│   └── settings.js
│
├── layouts/
│   ├── default.vue
│   └── auth.vue
│
├── middleware/
│   ├── auth.global.js
│   └── guest.js
│
├── pages/
│   ├── index.vue
│   ├── login.vue
│   └── profile/
│       └── [id].vue
│
├── plugins/
│   ├── normalize-digits.client.js
│   └── number-format.js
│
├── utils/
│   ├── storage.client.js
│   ├── messages.js
│   └── formatters.js
│
├── constants/
│   ├── enums.js
│   └── routes.js
│
├── styles/
│   ├── variables.css
│   ├── helpers.css
│   └── app.css
│
└── app.vue

server/
├── api/
│   ├── auth/
│   └── users/
│
└── middleware/
```

---

# Naming

| Entity | Convention | Example |
|----------|----------|----------|
| Pages | `name.vue` | `index.vue` |
| Components | `Name.vue` | `UserCard.vue` |
| Layouts | `name.vue` | `default.vue` |
| Stores | `useNameStore` | `useAuthStore` |
| Composables | `useName` | `useApiCall` |
| Middleware | `name.js` | `auth.global.js` |
| Route Names | `camelCase` | `profileDetail` |
| CSS Classes | `kebab-case` | `user-card` |

---

# Comments

All comments must be written in English.

```js
// --- Authentication Section ---

// Validate Token And Redirect User
```

Large sections:

```js
// ==============================
// API Error Handling Section
// ==============================
```

---

# API Layer

All backend communication must pass through:

```text
composables/useApiCall.js
```

Never use:

```js
axios.get(...)
$fetch(...)
```

directly inside pages.

Correct:

```js
const apiCall = useApiCall();

const user = await apiCall("/auth/me/");
```

Wrong:

```js
await $fetch("/auth/me/");
```

---

# API Responsibilities

Central API layer must handle:

```text
Base URL
Authorization
Timeout
JSON
HTML
Blob
401 Redirect
Loading
Messages
Network Errors
```

Every request must return:

```text
Object
Array
String
Blob
undefined
```

`undefined` means error was already handled globally.

---

# Services

Pages must never know backend URLs.

Correct:

```js
import {getCurrentUser} from "~/services/auth";

const user = await getCurrentUser();
```

Service:

```js
export async function getCurrentUser() {
    const apiCall = useApiCall();

    return await apiCall("/auth/me/");
}
```

---

# Store Rules

Stores are only for shared state.

Allowed:

```text
User
Auth State
Theme
Global Loading
Global Message
Settings
```

Not allowed:

```text
Temporary Form Inputs
Local Modal States
Page Specific Variables
```

---

# Global Store Structure

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

# Loading Pattern

Loading must be managed only inside API layer.

Correct:

```js
await apiCall("/users/");
```

Wrong:

```js
store.isLoading = true;
await apiCall("/users/");
store.isLoading = false;
```

---

# Message Pattern

All messages must go through:

```js
showMessage(message, status);
```

Allowed statuses:

```text
primary
success
warning
danger
info
```

Never:

```js
alert(...)
```

---

# Validation Rules

Validation must happen before API calls.

```js
const valid = await verifyForm([
    {
        val: mobile,
        reg: /^09\d{9}$/,
        msg: "شماره موبایل صحیح نیست",
    },
]);
```

Supported validators:

```text
RegExp
Function
Async Function
Custom Message
```

---

# Pages

Pages may contain:

```text
Lifecycle
Calling Services
Local State
Passing Data To Components
```

Pages must not contain:

```text
Raw API Requests
Complex Business Logic
Status Code Handling
Global Loading Logic
```

---

# Components

Components must do one thing only.

Good:

```text
UserCard.vue
Loading.vue
Message.vue
PriceBadge.vue
```

Bad:

```text
DashboardEverything.vue
```

---

# Composition API

Use:

```vue
<script setup>
</script>
```

for all new files.

Correct:

```vue
<script setup>
const user = ref(null);
</script>
```

---

# Computed Rules

Use computed values for derived state.

Correct:

```js
const fullName = computed(() => {
    return `${user.value.firstName} ${user.value.lastName}`;
});
```

---

# Middleware

Authentication must be handled through middleware.

```js
export default defineNuxtRouteMiddleware(() => {
    const authStore = useAuthStore();

    if (!authStore.isAuthenticated) {
        return navigateTo("/login");
    }
});
```

Never put business logic inside middleware.

---

# Layout Rules

Every page must use a layout.

Layouts may include:

```text
Loading
Message
Global Modals
Theme Switcher
Navigation
Footer
```

Pages must not import global UI manually.

---

# SEO Rules

Every public page must define:

```text
Title
Description
Canonical URL
OpenGraph
Twitter Card
Schema.org
```

Use:

```js
useSeoPage(...)
```

Never manually repeat SEO code inside pages.

---

# SEO Composable

```js
useSeoPage({
    title,
    description,
    path,
    image,
});
```

Must generate:

```text
title
meta description
canonical
og tags
twitter tags
schema
```

---

# Open Graph

Required:

```html
<meta property="og:title">
<meta property="og:description">
<meta property="og:image">
<meta property="og:url">
<meta property="og:type">
```

---

# Twitter Card

Required:

```html
<meta name="twitter:card">
<meta name="twitter:title">
<meta name="twitter:description">
<meta name="twitter:image">
```

---

# Canonical URL

Every public page must define:

```html
<link rel="canonical">
```

Example:

```html
<link rel="canonical" href="https://example.com/about">
```

---

# Schema.org

Minimum required schemas:

### Website

```json
{
  "@type": "WebSite"
}
```

### WebPage

```json
{
  "@type": "WebPage"
}
```

### Organization

```json
{
  "@type": "Organization"
}
```

Optional:

```text
Article
Product
FAQPage
BreadcrumbList
Person
LocalBusiness
```

---

# Dynamic SEO Example

```js
useSeoPage({
    title: article.title,
    description: article.description,
    path: `/blog/${article.slug}`,
    image: article.image,
});
```

---

# CSS Rules

Global styles:

```text
styles/variables.css
styles/helpers.css
styles/app.css
```

Component styles:

```vue
<style scoped>
</style>
```

Preferred:

```css
.user-card {
    border-radius: var(--radius-md);
}
```

---

# Theme System

Dark mode must:

```text
Read Preference
Persist Preference
Apply Root Class
Toggle Mode
```

Only global store may control dark mode.

---

# Runtime Config

Public values:

```ts
runtimeConfig.public
```

Example:

```ts
runtimeConfig: {
    public: {
        apiBaseUrl: "",
        siteUrl: "",
        siteName: "",
    },
}
```

Never place secrets inside public config.

---

# PWA

PWA configuration must be centralized.

Handle:

```text
Install Prompt
Update Detection
Refresh Notification
Offline Support
```

Never silently ignore updates.

---

# Security Rules

Checklist:

```text
Never Trust Local Storage Alone
Never Expose Internal Errors
Never Store Secrets In Frontend
Always Handle 401 Globally
Always Clear Invalid Tokens
Always Protect Auth Pages
```

---

# Imports

Always use aliases.

Correct:

```js
import {useApiCall} from "~/composables/useApiCall";
```

Wrong:

```js
import {useApiCall} from "../../../composables/useApiCall";
```

---

# File Upload

Always use:

```js
const formData = new FormData();
```

Never manually set:

```js
Content-Type: multipart/form-data
```

Browser must generate boundary automatically.

---

# HTML Responses

HTML responses must be normalized:

```js
{
    isHTML: true,
    contents: "<html>...</html>"
}
```

Never render unknown HTML blindly.

---

# Nuxt Philosophy Summary

```text
Pages Stay Thin
API Must Be Centralized
SEO Must Be Automatic
Schema Must Exist
Business Logic Lives Outside Pages
Stores Hold Shared State Only
Middleware Handles Access Control
Components Stay Small
Composables Stay Reusable
Runtime Config Controls Environment
```
