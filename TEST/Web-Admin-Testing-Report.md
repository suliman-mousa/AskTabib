# AskTabib Admin: Testing Stack Report

**Project:** AskTabib Admin Dashboard (`asktabib-admin`)
**Stack under test:** Vite · React 19 · JSX · React Router · Zustand · i18next · Axios
**Report date:** 17 October 2026
**Status:** Baseline stack in place; unit/component suite green; E2E green on Chromium, Firefox, and WebKit
---

## 1. Purpose

This report documents how automated testing was introduced on the admin dashboard, what we chose and why, how the suite is structured today, what is deliberately not tested, and what still needs growth. It is written for engineers who will maintain or extend the suite.

The goal is practical: catch regressions early, keep network boundaries honest, and protect the flows that must not break (admin sign-in and admission control) without turning every UI tweak into a brittle Playwright script.

---

## 2. Context and constraints

1. **JavaScript-first (JSX) codebase.** A thin `tsconfig.json` exists so tooling (Vitest globals, Playwright config, `tsc --noEmit`) works, with `allowJs` on and `checkJs` off. The typecheck step is a **hygiene check**, not static analysis of product code. Type-checking is being introduced file by file (see section 8).
2. **No TanStack Query, no Redux.** State lives in Zustand stores with no provider tree, so the test `render()` helper wraps **i18n + MemoryRouter only**.
3. **Served under `/dashboard/`.** Vite `base` and the React Router basename both use this path. E2E must use a base URL ending in `/dashboard/` and navigate with relative paths (e.g. `login`), not `/login`, or Vite serves its "did you mean /dashboard/login?" page.
4. **npm** is the package manager. Playwright browsers are a separate install step.
5. **Operating scope.** Admin users are located in Germany and Syria only. Tests for timezone behavior assume this scope.
6. **The backend is the security authority.** Frontend tests verify UX gates and client logic; they do not prove authorization.

---

## 3. Architecture of the test stack

| Layer | Tooling | Role |
|---|---|---|
| Unit / component | Vitest 3 + jsdom + Testing Library + user-event + jest-dom | Fast feedback on utils, hooks, and UI behaviour |
| Network in unit tests | MSW v2 (`http` + `HttpResponse`) | Real Axios calls, fake HTTP; no `vi.mock('axios')` |
| Accessibility | jest-axe | Smoke a11y on selected components |
| End-to-end | Playwright | Critical paths across Chromium, Firefox, WebKit |
| CI | GitHub Actions (`.github/workflows/ci.yml`) | Lint → typecheck → coverage → Playwright → artifacts |

Shared bootstrap lives under `src/test/`:

- `setup.js`: jest-dom, axe matcher, MSW lifecycle, RTL cleanup
- `server.js` / `handlers.js`: default handlers (sample login, notifications, session validation)
- `utils.jsx`: `render()` and re-exports from Testing Library
- `i18n.js`: in-memory translations so tests do not depend on HTTP locale loading

Conventions for agents and humans are also captured in `.cursor/rules/testing.mdc` and `TESTING.md`.

---

## 4. Design decisions (and trade-offs)

### 4.1 Vitest over a separate Jest install
The project already used Vitest with Vite. Extending that config avoids a second runner and keeps one transform pipeline.

### 4.2 MSW instead of mocking Axios
The admin client is Axios-centric (`axiosInstance` with interceptors for `lang`, `timezone`, and auth). Mocking Axios would skip that path. MSW sits at the network boundary, so interceptors still run. Handlers can be overridden per test with `server.use(...)`.

### 4.3 Coverage: scoped thresholds plus a global report
A global 70% threshold over all of `src/` would fail immediately on a large untested UI surface. Therefore:

- **Threshold-enforced set (70% lines/functions):** modules in the `include` list, which grows as co-located tests land.
- **Global report (no threshold):** coverage over all of `src/` is generated alongside, so the true project-wide number is always visible. The enforced percentage must never be quoted as project coverage.

Current global figure: **[X]%** (as of the last run).

### 4.4 E2E only for critical paths
Playwright covers admin sign-in, with API traffic stubbed via `page.route`. Broader journeys (appointments, doctors CRUD, FCM) stay out of E2E on purpose; they are better as RTL + MSW until the product surface stabilizes.

**When to write a new E2E test.** Only when all of these hold: the flow is business-critical, it depends on real browser behavior (routing, storage, redirects, service workers) that jsdom cannot represent, and a failure would not be caught by a component test with MSW.

### 4.5 Accessibility as a smoke check
`LanguageSwitcher` has a dedicated axe test. This is a baseline, not a WCAG audit. Note that `jest-axe` in jsdom cannot evaluate colour contrast or layout-dependent rules; those need a real browser (see section 8).

---

## 5. Current suite inventory

| Area | File(s) | What it asserts |
|---|---|---|
| URL safety | `src/utils/safeUrl.test.js` | Only `http`/`https` accepted |
| Timezone mapping | `src/utils/timezone.test.js` | `Europe/*` → Berlin; `Asia/*` and anything else → Damascus; `Intl` failure fallback |
| Auth session | `src/utils/auth.test.js` | Token expiry, storage cleanup, resolve token/user |
| Role admission | `[file]` | Only `role: admin` (case-insensitive) creates a session; other roles are rejected and nothing is persisted |
| Stale sessions | `[file]` | `isAuthenticated()` clears non-admin local sessions |
| Session validation | `[file]` | Startup validation against the server; invalid or tampered sessions are cleared |
| FCM payload allowlist | `[file]` | Only `admin_booking_received` updates the UI; other payloads are ignored |
| Login lockout | `[file]` | 60 s client lockout after five failures (UX layer; server rate limiting is the real control) |
| API hook | `src/hooks/useApi.test.jsx` | Success payload and field errors via MSW override |
| Language UI | `src/components/LanguageSwitcher.test.jsx` | userEvent language switch by role |
| A11y | `src/components/LanguageSwitcher.a11y.test.jsx` | No axe violations on the switcher |
| Pagination | `src/components/Pagination.test.jsx` | Next calls `onPage`; Previous disabled on page 1 |
| E2E login | `e2e/login.spec.ts` | Form visible; successful login (stub returns `role: "admin"`) lands on `/dashboard/home` |

**Last local verification ([date]):**

- `npm run test:run`: **[N]** tests passed
- `npm run test:coverage`: thresholds met on the scoped set; global coverage **[X]%**
- `npx playwright test`: **[N/N]** passed (Chromium, Firefox, WebKit)

### 5.1 Deliberately not tested (and why)

| Area | Reason | Alternative |
|---|---|---|
| FCM delivery / service-worker lifecycle | Depends on browser permission, HTTPS, and real push infrastructure | Manual release checklist; dedicated Playwright project if push becomes a release blocker |
| Server-side authorization and rate limiting | Backend responsibility | Laravel feature tests |
| File upload validation (server side) | Backend responsibility | Laravel tests; client magic-byte check is first-line only |
| Visual layout / regressions | Layout still changing | Revisit once layout churn settles |
| Charts and heavy visual widgets | Low regression value vs. cost in jsdom | Manual check; revisit with Playwright screenshots |

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

- Query the way a user would: **role, label, text**, not implementation details.
- Prefer **userEvent** over `fireEvent`.
- **MSW only** for HTTP in unit/component tests; do not stub `fetch`/`axios` directly.
- One behaviour per test; Arrange → Act → Assert.
- Co-locate tests next to source (`Component.test.jsx` beside `Component.jsx`).
- Keep Playwright for **critical flows**; stub remote APIs in E2E.
- **Security-relevant logic ships with tests.** Role checks, session handling, payload validation, and lockout logic are not considered done without them.
- **Flaky tests are not tolerated silently.** A test that fails intermittently is fixed or quarantined with a tracking issue within the same sprint; retries are a diagnostic tool, not a fix.

These rules are repeated in `.cursor/rules/testing.mdc`.

---

## 8. Gaps and recommended next work

The baseline is solid for tooling and for the security-critical slice of product logic. It is not yet a safety net for the whole admin surface. In priority order:

1. **E2E failure paths:** wrong password, expired session, unauthenticated redirect, and sign-in with a non-admin account being rejected.
2. **Protected / guest routes and logout:** session present vs. missing; logout request includes `fcm_token` when present (MSW request assertion).
3. **Appointments list + accept flow:** RTL + MSW (timezone query param; empty/error/success).
4. **Notification bell:** unread badge and mark-as-read against MSW.
5. **Contract drift protection:** the frontend and Laravel API are never tested together. Add one smoke test against staging, or validate MSW handlers against an OpenAPI spec.
6. **Network failure during session validation:** define and test behavior (retry vs. sign-out) so a weak connection does not eject an admin.
7. **Gradual type checking:** add `// @ts-check` to auth, API, and timezone modules; only then call the typecheck step a gate for those files.
8. **CI cost:** run Chromium only on pull requests; run all three browsers on `main` or nightly.
9. **Accessibility in a real browser:** run axe inside Playwright on the login and home pages (covers contrast, which jsdom cannot).
10. **Widen the coverage `include` list** as tests land, without relaxing the 70% bar; consider a rising global floor (e.g. 10%, then 20%).
11. **Test data factories** for appointments, doctors, and users, to keep MSW handlers consistent.
12. **Optional:** visual regression, once layout stabilizes.

---

## 9. Conclusion

The testing stack fits this repository as it actually is: Vite + React JSX, Router, Zustand, i18n, Axios, and a `/dashboard/` base path. Unit and component tests run fast with a shared provider helper and MSW; Playwright guards the sign-in path across three browsers; CI ties lint, typecheck, coverage, and E2E together. The security-critical client logic is covered by automated tests.

The remaining work is coverage of domain screens and closing the contract-drift gap, not more framework choices. Keep E2E reserved for paths where a real browser is the only honest judge.

---

