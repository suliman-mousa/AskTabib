# AskTabib Admin — Testing Stack Report

**Project:** AskTabib Admin Dashboard (`asktabib-admin`)  
**Stack under test:** Vite · React 19 · JSX · React Router · Zustand · i18next · Axios  
**Report date:** 15 October 2026  
**Status:** Baseline stack in place; unit/component suite green; E2E green on Chromium, Firefox, and WebKit

---

## 1. Purpose

This report documents how automated testing was introduced on the admin dashboard, what we chose and why, how the suite is structured today, and what still needs growth. It is meant for engineers who will maintain or extend the suite—not as marketing copy.

The goal was practical: catch regressions early, keep network boundaries honest, and cover the one flow that must not break (admin sign-in), without turning every UI tweak into a brittle Playwright script.

---

## 2. Context and constraints

A few facts shaped the design:

1. **The app is JavaScript-first (JSX),** not a TypeScript codebase. We added a thin `tsconfig.json` so tooling (Vitest globals, Playwright config, `tsc --noEmit`) works, with `allowJs` on and `checkJs` off. Typecheck here is a hygiene gate, not full static analysis of the product code.

2. **There is no TanStack Query and no Redux.** State lives in Zustand stores (no provider tree). The test `render()` helper therefore wraps **i18n + MemoryRouter only**.

3. **The app is served under `/dashboard/`.** Vite `base` and React Router basename both use that path. E2E must use a base URL that ends with `/dashboard/` and navigate with relative paths (e.g. `login`), not `/login`, or Vite serves its “did you mean /dashboard/login?” page.

4. **Package manager is npm.** Playwright browsers are a separate install step (`npx playwright install`).

---

## 3. Architecture of the test stack

| Layer | Tooling | Role |
|--------|---------|------|
| Unit / component | Vitest 3 + jsdom + Testing Library + user-event + jest-dom | Fast feedback on utils, hooks, and UI behaviour |
| Network in unit tests | MSW v2 (`http` + `HttpResponse`) | Real Axios calls, fake HTTP—no `vi.mock('axios')` |
| Accessibility | jest-axe | Smoke a11y on selected components |
| End-to-end | Playwright | Critical path across Chromium, Firefox, WebKit |
| CI | GitHub Actions (`.github/workflows/ci.yml`) | Lint → typecheck → coverage → Playwright → artifacts |

Shared bootstrap lives under `src/test/`:

- `setup.js` — jest-dom, axe matcher, MSW lifecycle, RTL cleanup  
- `server.js` / `handlers.js` — default handlers (sample login + notifications)  
- `utils.jsx` — `render()` / re-exports from Testing Library  
- `i18n.js` — in-memory translations so tests do not depend on HTTP locale loading  

Project conventions for agents and humans are also captured in `.cursor/rules/testing.mdc` and the shorter operator guide `TESTING.md`.

---

## 4. Design decisions (and trade-offs)

### 4.1 Vitest over a separate Jest install

The project already used Vitest with Vite. Extending that config (jsdom, globals, setup file, v8 coverage) avoids a second runner and keeps one transform pipeline.

### 4.2 MSW instead of mocking Axios

The admin client is Axios-centric (`axiosInstance` + interceptors for `lang`, `timezone`, auth). Mocking Axios would skip that path. MSW sits at the network boundary so interceptors still run. Handlers can be overridden per test with `server.use(...)`.

### 4.3 Coverage thresholds scoped to exercised modules

A global 70% line/function threshold over the entire `src/` tree would fail immediately on a large UI surface that is not yet tested. Coverage **include** is limited to modules the starter suite actually hits (auth/timezone/safeUrl utils, `useApi`, LanguageSwitcher, Pagination). Thresholds stay meaningful; the include list should grow as new co-located tests land.

### 4.4 E2E only for the critical path

Playwright covers admin login → home, with API traffic stubbed via `page.route`. Broader journeys (appointments, doctors CRUD, FCM) stay out of E2E for now on purpose: they are better as RTL + MSW until the product surface stabilizes.

### 4.5 Accessibility as a first-class smoke check

`LanguageSwitcher` has a dedicated axe test. That is a baseline, not a full WCAG audit. Expand axe coverage to high-traffic interactive widgets over time.

---

## 5. Current suite inventory

| Area | File(s) | What it asserts |
|------|---------|-----------------|
| URL safety | `src/utils/safeUrl.test.js` | Only `http`/`https` accepted |
| Timezone mapping | `src/utils/timezone.test.js` | Europe → Berlin; else Damascus; Intl failure fallback |
| Auth session | `src/utils/auth.test.js` | Token expiry, storage cleanup, resolve token/user |
| API hook | `src/hooks/useApi.test.jsx` | Success payload + field errors via MSW override |
| Language UI | `src/components/LanguageSwitcher.test.jsx` | userEvent language switch by role |
| A11y | `src/components/LanguageSwitcher.a11y.test.jsx` | No axe violations on the switcher |
| Pagination | `src/components/Pagination.test.jsx` | Next calls `onPage`; Previous disabled on page 1 |
| E2E login | `e2e/login.spec.ts` | Form visible; successful login lands on `/dashboard/home` |

**Last local verification (15 Oct 2026):**

- `npm run test:run` — 19 tests passed  
- `npm run test:coverage` — thresholds met on the scoped include set  
- `npx playwright test` — 3/3 passed (Chromium, Firefox, WebKit)

---

## 6. How to run

```bash
npm run test          # watch
npm run test:run      # single run
npm run test:ui       # Vitest UI
npm run test:coverage # v8 → text + html + lcov
npm run e2e           # Playwright (all configured browsers)
npm run e2e:ui        # Playwright UI
npm run lint
npm run typecheck
```

Playwright browsers (once per machine):

```bash
npx playwright install
# or with OS dependencies:
npx playwright install --with-deps
```

CI mirrors the same gates and uploads `coverage/` and `playwright-report/` as artifacts.

---

## 7. Engineering standards we enforce

- Query the way a user would: **role, label, text**—not implementation details.  
- Prefer **userEvent** over `fireEvent`.  
- **MSW only** for HTTP in unit/component tests; do not stub `fetch`/`axios` directly.  
- One behaviour per test; Arrange → Act → Assert.  
- Co-locate tests next to source (`Component.test.jsx` beside `Component.jsx`).  
- Keep Playwright for **critical flows**; stub remote APIs in E2E.

These rules are repeated in `.cursor/rules/testing.mdc` so tooling and humans stay aligned.

---

## 8. Gaps and recommended next work

The baseline is solid for tooling and for a thin slice of product logic. It is not yet a safety net for the whole admin surface. Highest-value follow-ups:

1. **Appointments list + accept flow** — RTL + MSW (timezone query param, empty/error/success).  
2. **Notification bell** — unread badge and mark-as-read against MSW.  
3. **Protected / guest routes** — session present vs missing.  
4. **Logout** — body includes `fcm_token` when present (MSW assertion on request).  
5. **Widen coverage include** as those tests land, without relaxing the 70% bar.  
6. **Optional:** visual regression later; not justified until layout churn settles.

FCM / service-worker behaviour is intentionally out of unit scope for now (browser permission + SW lifecycle). Prefer a small manual checklist or a dedicated Playwright project if push delivery becomes a release blocker.

---

## 9. Conclusion

We installed a production-shaped testing stack that fits this repo as it actually is: Vite + React JSX, Router, Zustand, i18n, Axios, and a `/dashboard/` base path. Unit and component tests run fast with a shared provider helper and MSW; Playwright guards the login path across three browsers; CI wires lint, typecheck, coverage, and E2E together.

The remaining work is coverage of domain screens—not more framework choices. Expand co-located tests module by module, grow the coverage include list with them, and keep E2E reserved for paths where a real browser is the only honest judge.

---
