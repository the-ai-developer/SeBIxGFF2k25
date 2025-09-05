# BondFlow Frontend Application — UX/UI Architecture, Standards, and Playbooks

Responsive, accessible, multilingual, high-performance UI for web and mobile. Built for trust, clarity, and speed — from onboarding to trading to portfolio analytics.

## Executive Snapshot
- Stack: React (web) + React Native (iOS/Android) + TypeScript + Design System + Charting + i18n
- Principles: Accessibility (WCAG 2.2 AA), Performance (Lighthouse > 90), Security (CSP, OWASP), Privacy (DPDP), Observability (RUM + Sentry)
- UX Goals: Clean, intuitive flows; bond-first data viz; education embedded; zero-trust handling of tokens/PII

---

## 1) Architecture & Tech Choices

- Monorepo: Turborepo/Nx with packages:
  - apps/web, apps/mobile
  - packages/ui (design system), packages/logic (utils/hooks), packages/api (generated client)
  - packages/i18n, packages/charts, packages/config
- Web: React + TypeScript
  - Recommended: Next.js (App Router) for SSR/ISR on discovery/education pages; SPA mode for trading if preferred
  - Alternative: Vite + SPA with service worker for PWA
- Mobile: React Native + TypeScript
  - Navigation: React Navigation
  - Storage: MMKV/SQLite (encrypted)
- Styling: Tailwind CSS or Chakra UI + CSS Variables design tokens; dark mode support
- State:
  - Global/client state: Redux Toolkit or Zustand
  - Server state: RTK Query or React Query
- Data Viz: Recharts/Visx or ECharts with A11y overlays
- Testing: Jest + React Testing Library, Cypress/Playwright (E2E), axe-core (a11y), Percy/Chromatic (visual)
- Tooling: ESLint + Prettier + Husky + Lint-Staged, Commitlint (Conventional Commits)

Directory sketch:
```
/apps
  /web
  /mobile
/packages
  /ui
  /charts
  /api
  /logic
  /i18n
  /config
```

---

## 2) Design System & Accessibility

- Tokens: color, spacing, typography, z-index, radii via CSS variables/Style Dictionary
- Components: Button, Input, Select, Dialog, Drawer, Tabs, Tooltip, Snackbar, DataTable, Card, Badge, Stepper, Skeleton, EmptyState
- Theming: Light/Dark, high-contrast variant; brand-safe palette with AA contrast
- A11y (WCAG 2.2 AA):
  - Keyboard-first navigation; visible focus rings
  - ARIA roles/labels; skip-to-content; LiveRegion for async updates
  - Form errors announced to screen readers
  - Color-blind safe charts; patterns/markers not just color
- Localization:
  - react-i18next; ICU pluralization; date/number formats
  - Indian numbering (lakhs/crores) toggle; English + vernaculars (hi, ta, te, bn, mr, etc.)
  - Dynamic resource loading; fallback locales; RTL-ready plumbing (for future)

Design artifacts: Figma library aligned to tokens; Storybook for component docs and accessibility tests.

---

## 3) Security & Privacy (Frontend)

- Auth:
  - OAuth 2.1 / OIDC with PKCE; short-lived access tokens; refresh rotation
  - Store tokens in httpOnly SameSite=strict cookies (preferred) or secure storage on mobile (Keychain/Keystore)
- CSP & Headers:
  - Strict CSP (nonce-based scripts), HSTS, X-Frame-Options: DENY, Referrer-Policy: no-referrer
- Input/Output:
  - Client-side validation + server-side validation; sanitize HTML; escape untrusted content
- CSRF:
  - Double-submit cookie or SameSite=strict; verify anti-CSRF token on state-changing requests
- PII:
  - No PII in logs/analytics; redaction at source; consent banner per DPDP
- Device Security (mobile):
  - Certificate pinning, root/jailbreak detection, biometric auth (OS-provided), secure keystore

---

## 4) API Integration Layer

- Client: Axios or Fetch (with “ky”) wrapped in a typed API client
- Types: OpenAPI → TypeScript client generation; keep endpoints in one place
- Patterns:
  - Interceptors for auth/refresh; exponential backoff + jitter; circuit breaker for flaky upstreams
  - Caching and invalidation via RTK Query/React Query; optimistic updates where safe
  - Real-time: WebSocket/SSE for quotes and order status; heartbeat + auto-reconnect
- Observability:
  - X-Request-ID correlation; RUM (web-vitals) + Sentry Performance; masked breadcrumbs

Example RTK Query slice:
```ts
export const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: process.env.NEXT_PUBLIC_API_BASE }),
  tagTypes: ['Bond','Portfolio','Orders'],
  endpoints: (b) => ({
    listBonds: b.query<BondSummary[], BondsQuery>({ query: (q)=>({ url:'/bonds', params:q }), providesTags:['Bond'] }),
    placeOrder: b.mutation<OrderAck, PlaceOrder>({ query: (o)=>({ url:'/orders', method:'POST', body:o }), invalidatesTags:['Orders','Portfolio'] }),
  })
});
```

---

## 5) State Management Strategy

- Global UI state: theme, modals, toasts, language (Zustand/Redux slice)
- Domain state:
  - Server cache (RTK Query/React Query): bonds, quotes, orders, portfolio
  - Derived metrics computed via memoized selectors (duration, DV01, convexity)
- Error boundaries per route; retry policies per resource; stale-while-revalidate

---

## 6) Core User Flows & Pages

- Onboarding & KYC
  - Stepper: signup → consent → Aadhaar Offline/PAN → selfie/V-CIP (if required) → bank link (penny-drop) → summary
  - Real-time validation, save-and-resume, error explainers in local language
- Bond Discovery
  - Search/filter: rating, tenor, YTM, issuer, liquidity
  - Cards: ISIN, coupon, price (clean/dirty), YTM/YTC, rating trend, call schedule badge
  - Yield curve and spread heatmaps; “compare” drawer
- Trading
  - Order ticket: clean vs dirty price display, accrued interest calc, fees, TDS preview
  - Quotes: RFQ/limit/IOC; slippage guardrails; confirm with risk disclosure
  - Status: partial fills, amendments, cancels
- Portfolio
  - Holdings table, PnL breakdown (coupon vs price vs fees), DV01, duration, convexity
  - Coupon calendar; record/ex-date flags; alerts
  - Export statements (CSV/PDF)
- Education
  - Interactive modules, quizzes, calculators; inline tooltips
- Support
  - FAQ, chat, tickets; SCORES/ODR linkouts; status page

---

## 7) Data Visualization

- Components:
  - Yield Curve (interactive), Price–Yield scatter, Time-series price/YTM, Coupon Calendar, Ladder Builder
  - Risk widgets: DV01, duration, convexity, exposure by rating/sector/tenor
- Features:
  - Zoom, pan, tooltips, keyboard access, downloadable PNG/CSV
  - A11y: ARIA for datasets, data tables mirrors for screen readers

---

## 8) Performance & PWA

- Budgets: TTI < 2.0s (4G), FCP < 1.5s, CLS < 0.1, LCP < 2.5s; Lighthouse > 90
- Techniques:
  - Code splitting, route-level and component-level lazy loading
  - Image optimization (next/image or responsive srcset), prefetch on idle
  - Skeletons/shimmers; avoid layout shifts
- PWA:
  - Service worker with stale-while-revalidate for static; offline read-only mode (portfolio snapshot)
  - Web Push via FCM for alerts (opt-in)

---

## 9) Internationalization (i18n) Details

- Library: react-i18next with lazy-loaded namespaces per route
- Formats:
  - Numbers: Intl.NumberFormat with “en-IN” option; toggle to international (en-US)
  - Dates: Day.js/Date-fns; locale-aware; time-zone consistent (IST)
- Example:
```json
// i18n/hi/common.json
{
  "buy": "खरीदें",
  "sell": "बेचें",
  "yield_to_maturity": "परिपक्वता तक प्रतिफल (YTM)",
  "clean_price": "क्लीन प्राइस",
  "dirty_price": "डर्टी प्राइस"
}
```

---

## 10) Notifications

- In-app: Toaster for ephemeral, Inbox for durable (trade confirms, coupons, KYC updates)
- Push: Web Push (FCM) and mobile push; granular controls; quiet hours
- Email/SMS: Transactional only; templated via provider with signed links

---

## 11) Error Handling & Resilience

- Global ErrorBoundary with friendly fallback; per-page boundaries
- Network: retry with backoff; offline banner; degraded mode for partial outages
- Maintenance screen with ETA and status page link
- Sentry alerts mapped to severity; redaction + sampling

---

## 12) Getting Started (Dev Setup)

1) cd frontend
2) Node LTS + PNPM/Yarn
3) Install: pnpm i
4) Env: copy .env.example → .env.local and set
   - NEXT_PUBLIC_API_BASE=https://api.dev.bondflow.in
   - NEXT_PUBLIC_SENTRY_DSN=...
   - NEXT_PUBLIC_I18N_DEFAULT=en
5) Run: pnpm dev (or yarn start)
6) Tests: pnpm test; E2E: pnpm e2e

---

## 13) Example Pages/Components

- Order Ticket (clean vs dirty)
```tsx
function OrderTicket({ bond }: { bond: Bond }) {
  const [qty, setQty] = useState(1);
  const [cleanPrice, setCleanPrice] = useState(bond.cleanPrice);
  const accrued = useAccruedInterest(bond);
  const dirty = cleanPrice + accrued;

  return (
    <Card>
      <h3>{bond.isin} · {bond.coupon}% · Maturity {formatDate(bond.maturity)}</h3>
      <NumberInput label="Quantity (fractions)" value={qty} onChange={setQty} min={1} />
      <PriceInput label="Clean Price" value={cleanPrice} onChange={setCleanPrice} />
      <InfoRow label="Accrued Interest" value={formatMoney(accrued)} />
      <InfoRow label="Dirty Price" value={formatMoney(dirty)} emphasis />
      <Button variant="primary">Place Order</Button>
    </Card>
  );
}
```

- Chart hook (yield curve)
```tsx
const useYieldCurveData = () => {
  const { data } = api.useGetYieldCurveQuery();
  return data?.points.map(p => ({ tenor: p.years, yield: p.yield_bps / 100 })) ?? [];
};
```

---

## 14) Testing Strategy

- Unit: Jest + RTL; 80%+ coverage for core logic
- Integration: Cypress/Playwright on key flows (KYC, order, portfolio)
- Accessibility: axe-core in CI; Storybook a11y addon
- Visual: Percy/Chromatic on critical pages
- Contract: Pact tests for API schemas
- Smoke: On every deploy, synthetic buy/sell test in staging

Sample E2E (Playwright):
```ts
test('place buy order flow', async ({ page }) => {
  await page.goto('/bonds');
  await page.getByPlaceholder('Search').fill('AAA PSU');
  await page.getByRole('link', { name: /AAA PSU 2028/ }).click();
  await page.getByRole('button', { name: /Buy/ }).click();
  await page.getByLabel(/Quantity/).fill('10');
  await page.getByRole('button', { name: /Place Order/ }).click();
  await expect(page.getByText(/Order confirmed/)).toBeVisible();
});
```

---

## 15) Analytics, Consent, and Telemetry

- Consent banner (DPDP): opt-in categories; no PII; IP anonymization
- RUM: Web Vitals (CLS/LCP/INP), route timings, error rates; dashboards
- Event schema: page_view, search, filter_apply, order_place, order_cancel, kyc_step_complete (with minimal, non-PII params)

---

## 16) Mobile (React Native) Specifics

- Navigation: Stack + Bottom Tabs; deep links (bondflow://bond/ISIN)
- Security: Keychain/Keystore for secrets; biometric unlock; device attestation (Play Integrity/App Attest) if required
- Offline: SQLite/WatermelonDB for cached portfolio; sync on reconnect
- Performance: Hermes engine; FlashList for large lists; image caching; reanimated for charts
- Updates: OTA via CodePush/App Store policies; feature flags for gradual rollout

---

## 17) Performance Guardrails

- Bundle splitting; < 200KB critical JS per route (gzipped)
- Avoid heavy polyfills; modern browsers baseline
- Memoization and virtualization on tables/lists
- Pre-connect/dns-prefetch for API and fonts; local fonts to avoid FOUT

---

## 18) Contribution Guidelines (TL;DR)

- Code style: ESLint + Prettier; TypeScript strict
- Git: Conventional Commits; PR templates; small, reviewable changes
- Tests: Add/maintain unit + E2E; a11y checks mandatory for UI
- Storybook: Every UI component with docs, props, and accessibility notes
- Design: Follow tokens and components; no ad-hoc colors/paddings
- Security: No secrets in code; no 3rd-party scripts without approval; CSP-safe

---

## 19) Roadmap (Frontend)

- Phase 1: Core flows (Discovery, Order, Portfolio), i18n (en + 2 vernaculars), charts v1, PWA baseline
- Phase 2: Advanced analytics (DV01, convexity tools), notifications, dark mode, RN app beta
- Phase 3: Performance tuning (LCP/INP targets), tournaments/Playground integration, accessibility audit pass
- Phase 4: Voice commands (experimental), portfolio backtesting UI, educator mode widgets

---

## 20) Future Enhancements

- Biometric auth (mobile), passkeys (web)
- Real-time RFQ tiles, dealer streaming quotes (UI)
- Advanced chart overlays (curve roll-down paths, spread ladders)
- In-app tutorials with guided tours (react-joyride)
- Personalization: saved filters, watchlists, nudges based on learning goals

---

## 21) Env & Config Example

```
NEXT_PUBLIC_API_BASE=https://api.dev.bondflow.in
NEXT_PUBLIC_I18N_DEFAULT=en
NEXT_PUBLIC_FEATURE_FLAGS=https://flags.dev.bondflow.in
NEXT_PUBLIC_SENTRY_DSN=...
```
