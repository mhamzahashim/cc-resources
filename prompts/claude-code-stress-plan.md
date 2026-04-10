# Claude Code Stress Plan

A self-review prompt you paste after Claude Code drafts a plan in plan mode. It forces Claude to walk every layer of the stack, answer "is this handled? what about that edge case?", and add missing pieces to the plan before a single line of code is written.

Cut my production regressions by ~90%. Tuned for Next.js 15 + Supabase (self-hosted) + Clerk + Dokploy but most of it is stack-agnostic.

---

## Why this exists

Plan mode in Claude Code looks thorough, but the plans have repeat blind spots that ship as production bugs. A partial list of what was biting me before I wrote this:

- `tsc --noEmit` passed but `next build` failed on a server-only module (nodemailer, node:crypto, geoip-lite) leaking into the client bundle via a barrel file
- Feature worked in my personal workspace but broke in team workspaces because the query wasn't scoped to `workspace_id`
- Double-click created two DB rows because there was no idempotency key
- New page had no `loading.tsx` or `error.tsx`, so the default Next.js fallback rendered for users
- Middleware regression because the new public route wasn't added to the public matcher
- Race condition because the limit check happened BEFORE the insert instead of in the same transaction
- React hooks ordering bug: early return above a `useEffect`, crashed every published page with React Error #310
- Controlled input anti-pattern: backspace got eaten on slow networks because the debounce re-hydrated mid-keystroke
- `process.env.X` used directly instead of through the env validator, prod crashed on startup
- New form field type added to the editor but not to the public renderer switch, so published forms crashed

Every one of these was catchable in the plan. Claude just wasn't being asked the right questions.

## Workflow

1. Enter plan mode in Claude Code
2. Describe the feature you want
3. Claude drafts its plan
4. Paste the prompt below as your next message
5. Claude walks every section, flags N/A where it doesn't apply, and adds missing pieces to the plan as it goes
6. Claude ends with a forced summary in four buckets:
   - **READY**: fully defined and buildable
   - **ADDED**: things missing from the original plan that the stress-test just added
   - **NEEDS MY INPUT**: open questions you need to answer
   - **RISK WATCHLIST**: top 3 things most likely to break in prod for this specific feature
7. Review the four buckets, answer the open questions, THEN approve the plan

The forced summary at the end is the real trick. Without it, Claude buries the important stuff 2000 tokens deep in the self-review and nobody scrolls that far. With it, the risks and gaps land at the top where you can actually act on them.

## Notes before you paste

- **It's long on purpose.** The cost of Claude saying "N/A" to 40 irrelevant questions is nothing. The cost of one missed question is a production bug.
- **Many of the ⚠️ items are bugs I've actually shipped broken at least once.** If it seems paranoid about a specific area, that's because it bit me.
- **Delete sections that don't apply to your product.** If you don't have a quiz builder, cut that. If you don't have workspaces, cut the multi-tenancy section. Don't paste checks that don't match your app or you'll dilute signal.
- **Swap in your stack's equivalents.** It's tuned for Next.js 15 + Supabase + Clerk + Dokploy. Internal helper names (`createApiHandler()`, `useCurrentWorkspaceId()`, `getEffectiveTier()`) and file paths (`form-renderer-v2.tsx`, `api-auth.ts`) are from my codebase. Replace with yours or delete.
- **Keep the final "Do NOT write a single line of code until I review and confirm" line verbatim.** Without it, Claude races ahead and starts writing code while you're still reviewing the summary.

---

## The prompt

Copy everything in the code block below and paste it as your next message in plan mode after Claude drafts a plan:

````
Before we proceed, I want you to stress-test this plan completely. Do a full self-review pass and answer EVERY single point below. Do NOT skip any section.

If something is not planned yet, say so explicitly and add it to the plan now. If a question is not aligned with the feature at hand, say "N/A: <reason>" and move on. You may rewrite any question or any part of the plan if it seems misaligned with what we're actually building.

This stress test is tuned for a Next.js 15 + Supabase (self-hosted) + Clerk + Dokploy stack. Pay extra attention to the project-specific gotchas marked with ⚠️. These are patterns that have bitten me in production before.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔁  SCENARIO & EDGE CASE COVERAGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

For EVERY feature, walk through:

HAPPY PATH
- Step by step, what does a successful flow look like from click to database to renderer to analytics?

FAILURE PATH at every single layer:
- What if the API call fails (500, 422, 401, network error)?
- What if the DB write succeeds but the response never reaches the client?
- What if the DB write fails halfway through a multi-step operation?
- What if a third-party service (email, payment, storage, logo.dev, ip-api.co) is down or returns 404?
- What is the fallback when the primary path fails?

VICE VERSA & REVERSE FLOW:
- What if the user does the steps in reverse or out of order?
- What if they trigger the same action twice in a row (double submit)?
- What if they click the button while a request is already in flight?
- What if they open the same feature in two browser tabs simultaneously?
- What if two different users act on the same record at the same time (race condition: TOCTOU on a limit check)?
- What if the user hits the browser back button mid-flow?
- What if they refresh the page mid-flow, do they lose progress or get stuck?
- What if they close the tab and reopen it, where do they land?
- What if they "undo" an action, does the prior state fully restore?

TIMING & CONNECTION:
- What if they lose internet halfway through an action?
- What if the Clerk session token expires while they are in the middle of the flow? Is it silent refresh or forced logout?
- What if Clerk clock skew causes an intermittent sign-in redirect?
- What if a background job or webhook takes longer than expected?
- What if the same webhook/event fires twice (Shopify, Stripe, etc.)?
- What if a cron job runs while the user is actively using the feature?

DATA EDGE CASES:
- What if a required field is null, empty, or an unexpected type?
- What if the data is at the max allowed length or size?
- What if the user pastes in malformed data (HTML, SQL, scripts, emoji, RTL)?
- What if a related record was deleted by someone else mid-flow?
- What if a JSONB column has a shape the UI doesn't expect (schema drift)?
- What if numbers include locale-specific decimal separators (1,23 vs 1.23)?

USER TYPE EDGE CASES:
- Brand new user with zero existing data (onboarding / first-run)?
- Power user with thousands of records?
- Free user hitting a Pro/Enterprise gate?
- Pro user in an Enterprise-owned workspace (billing inherits from workspace OWNER)?
- User mid-subscription (cancelled but still in billing period)?
- Admin role: does it bypass plan limits correctly?
- Suspended account / deactivated workspace?
- Viewer-role user (read-only permissions)?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏗️   BUILD, BUNDLING & WEBPACK LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  This is the #1 silent killer. Passes locally, breaks in production.

SERVER/CLIENT BOUNDARY:
- Does any new file import a Node-only module? (node:crypto, node:dns, node:fs, nodemailer, geoip-lite)
- If YES, is that file ONLY imported from server components or API routes, never from a "use client" component or a hook that bundles client-side?
- Is there a barrel file (index.ts) that re-exports both server and client code and could leak a server module into the client bundle?
- Should the Node module be loaded via dynamic import() or require() instead of static import?
- Does any hook (e.g., useSubscription) transitively import nodemailer, email utils, or crypto?
- Use Web Crypto API (globalThis.crypto) instead of node:crypto where possible.

TYPE CHECK vs REAL BUILD:
- Will this pass BOTH `npm run type-check` AND `npm run build`? (tsc misses: CSS purging, dynamic import errors, module resolution, missing Tailwind classes)
- Any invalid Tailwind (e.g., focusRingColor, custom delay-* not in config)?
- Any new dependency that requires `npm install --package-lock-only` to sync package-lock.json?
- Any route using edge runtime that imports node: modules, Email/SMTP, or crypto? → remove `export const runtime = 'edge'`

BIOME & LINT:
- Any `as any` without `// biome-ignore lint/suspicious/noExplicitAny: <reason>`?
- Any `import` that should be `import type`?
- Any `<button>` without `type="button"`?
- Any `onClick` handler on a non-button element without a keyboard handler?
- Any `delete` operator (forbidden: use destructuring)?

CI CHAIN (all MUST pass before push):
- `npm run lint && npm run type-check && npm run test && npm run build`
- Do coverage thresholds still pass?
- Are any test mocks now stale because the handler signature changed?
- Does `git diff --name-only` show ONLY files related to this feature? (never cascade to unrelated files: that has caused force-push reverts)


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🛣️   ROUTING & MIDDLEWARE LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ROUTE CONFLICTS:
- Does the new route path exist in both (dashboard) and (marketing) route groups? Next.js will throw.
- Is a dynamic segment [slug] shadowing a static one (e.g., /templates/new vs /templates/[slug])?
- Is a new admin route at a path that a user-facing page uses (e.g., /changelog vs /changelog/manage)?
- Are parallel routes and route groups sanity-checked for the target URL?

MIDDLEWARE PUBLIC / PROTECTED:
- Is this route public (no auth) or protected?
- If PUBLIC (sitemap.xml, robots.txt, /sign-in, OAuth callbacks, public form URLs, webhook receivers, health checks): is it in the `isPublicRoute` matcher in middleware.ts?
- If PROTECTED: is it reachable via a dashboard route matcher?
- ⚠️  middleware.ts has bitten me 7+ times. If you're touching it, STOP and ask first. NEVER unconditionally call `auth.protect()`. NEVER fully skip auth on RSC requests.

AUTH EDGE CASES:
- Does the flow work for RSC requests differently than regular requests?
- Does it work when Clerk token is expired but refresh token is valid (silent refresh)?
- Does it work after 10+ minutes with the tab open (focus-triggered token refresh)?
- Does it work for the admin role (admin pages have profile cache quirks: use cache bust on subscription change)?
- Does it work on custom domains (different hostname, same app instance)?

SHORT_ID vs UUID (⚠️  common bug):
- Does the public URL use `short_id` (e.g., /f/<short_id>)?
- Does the mutation use the real UUID, not the URL short_id?
- Does quiz use /q/ prefix (NOT /f/) and redirect /f/<quiz> → /q/<quiz>?
- Is the Share tab URL, preview URL, and detail page URL ALL using short_id consistently?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🖥️   UI LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STATES, is every possible visual state accounted for?
- Loading: tailored skeleton (not generic spinner) via `loading.tsx`
- Empty: brand-styled, with an actionable CTA, not a dead link
- Error: exact message shown, actionable, via `error.tsx` boundary
- Success: confirmation pattern (toast, inline, redirect?)
- Disabled: buttons/inputs, when and why?
- Partial/degraded: some data loaded, some failed
- Chunk load error: auto-recovery for failed dynamic imports?
- 404: lives INSIDE dashboard layout (not breaking out)?

FORMS:
- Exact validation rules per field (required, min/max, format)?
- Validation trigger: on blur, on change, or on submit?
- Exact error messages per rule?
- Is submit disabled until valid, or does it validate on click?
- Is there a confirmation step before destructive actions?
- Is the form reset after submit or does it retain values?

TEXT CLIPPING & OVERFLOW (⚠️  very common bug):
- Does every dropdown/select have enough width for the longest option?
- Are you using `leading-*` (not just `h-*`) for vertical text centering in dropdowns?
- Are tables using `overflow-visible` instead of `overflow-hidden` where needed (Matrix columns clipped otherwise)?
- Did you use `overflow-x-clip` instead of `overflow-x-hidden` (so sticky sidebars still work)?
- Will long text (100+ chars) cause unwanted horizontal scrollbars?
- Can buttons/action icons be hidden behind another element on hover?

CONTROLLED INPUT ANTI-PATTERN (⚠️  caused the "backspace ignored" bug):
- Is the input `value` bound directly to server/DB state? → BUG if yes.
- Is there LOCAL state for the text input that merges with server state only on initial load?
- Is the debounce ONLY on the persistence (save to DB), not on the display value?
- Test: type in a field, press backspace rapidly, does the cursor jump?

REACT HOOKS ORDER (⚠️  React Error #310 / #185):
- Are ALL useEffect / useMemo / useState hooks placed BEFORE any early returns?
- In the public renderer, early returns above hooks crash ALL published pages.
- Any useRef initialized with `useRef(computedValue)` where the guard checks `prev !== computedValue`? → First render won't fire; silent data loss on submit.
- Any stale closure in a keyboard / timer handler? Use refs for the latest handler.
- Any infinite re-render loop (setState in render, missing deps)?

FIELD TYPE COVERAGE (⚠️  every new type needs ~10 touchpoints):
When adding a new field type, update ALL of these:
  1. FieldType enum in src/shared/types/form.ts
  2. Editor Add Block dialog (+ preview)
  3. Editor block preview (editor-block-preview.tsx)
  4. Public renderer switch (form-renderer-v2.tsx)
  5. NON_INPUT_TYPES list (if screen/statement)
  6. Deprecated aliases map (if renaming)
  7. Template picker on forms/new
  8. Quiz builder (if quiz-compatible)
  9. validation.ts
 10. Answer piping resolver
Did you miss any? (16 field types were missing from the renderer once.)

FORM/QUIZ BUILDER SYNC:
- Did you make the SAME change in BOTH form builder and quiz builder?
- Quiz editor has all the same tabs as form editor?
- Short_id handling consistent across both?
- Quiz mutations use real UUID from DB, not URL short_id?

THEME CONSISTENCY:
- Every color uses a CSS variable, never hardcoded hex?
- Both dashboard AND published page are themed?
- Cookie banner, toasts, modals, error pages, loading states, all themed?
- Email templates use brand colors?
- No old-theme leftover hex codes? (grep for the old palette)

PREVIEW vs PUBLISHED:
- Does the editor preview actually match the published output?
- Does the mobile preview (375px phone frame) render like real mobile?
- Does preview work for DRAFT records (not just published)?
- Does the theme apply in preview same as in public?

NAVIGATION & FLOW:
- Where does the user land after each action?
- Unsaved changes warnings before navigating away?
- Breadcrumbs or back buttons needed?
- Does the URL reflect state changes (shareable/bookmarkable)?

RESPONSIVENESS & COMPATIBILITY:
- Designed for mobile (375px), tablet, desktop?
- Touch targets ≥44px on mobile?
- Safari iOS: any known quirks (100vh, date input, sticky)?
- Chrome, Firefox, Edge latest?
- Does the `100vh` issue affect the layout? Use `100dvh` for dynamic.

ACCESSIBILITY:
- Full keyboard navigation (tab order, enter/escape)?
- Screen-reader reachable? ARIA labels where needed?
- Sufficient color contrast (WCAG AA)?
- Error messages linked to fields via aria-describedby?
- Focus visible and trapped in modals?

FEEDBACK & COMMUNICATION:
- Toast notifications for async actions (centered, themed)?
- Inline confirmation for destructive actions?
- Progress indicator if action takes >1s?
- "Still working..." message if >5s?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🪝   HOOKS / STATE MANAGEMENT LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

BASIC HYGIENE:
- Every custom hook has a single, clear responsibility?
- What's local state, global state, and server state? (no overlap)
- Optimistic updates used? What does rollback look like on failure?
- Every mutation followed by cache invalidation or refetch?
- Expensive computations memoized to avoid stale/re-calc?
- Subscriptions / listeners cleaned up on unmount?
- State that persists across refresh (localStorage, sessionStorage)?
- State that must NOT persist across sessions, fully cleared on logout?
- Any stale closure capturing old state in async callbacks?

CLERK AUTH + PROFILE CACHE:
- Server-side profile cache TTL (currently 60s in api-auth.ts)?
- Client-side staleTime and refetchOnWindowFocus?
- Does the hook invalidate on workspace switch?
- Does the admin-role profile cache refresh when subscription changes?

WORKSPACE-SCOPED DATA (⚠️  NON-NEGOTIABLE):
- Does the query key include `workspaceId`?
- Does the API call include `?workspace_id=<id>` query param?
- Does the hook use `useCurrentWorkspaceId()` from workspace-context.tsx?
- Does switching workspaces auto-refetch the data?

BILLING INHERITANCE:
- Is tier read from the WORKSPACE OWNER's profile, not the logged-in user's?
- `useIsPaidPlan()` uses `getEffectiveTier()`?
- Is tier cached in localStorage to prevent gate-flash on navigation?

AUTO-SAVE:
- Is the auto-save key session-scoped (sessionStorage keyed on userId) to prevent cross-user leakage?
- Is the timer cleared on "Submit Another" and on unmount?
- Does it survive tab close and resume correctly?

DATA ROUTING (⚠️  RLS blocks browser client):
- Can the browser Supabase client read this data via RLS? Clerk-authed users mostly CAN'T, use supabaseAdmin via an API route.
- If you're using `createClientComponentClient` anywhere, ask: does this query hit an RLS policy it can't satisfy? Likely YES → route through an API route that uses `supabaseAdmin`.


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔌  API LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

HANDLER FACTORY (hard rule):
- Every route uses `createApiHandler()`, never raw NextResponse?
- Is `auth: 'clerk' | 'public' | 'apiKey'` set?
- Is `rateLimit: 'apiRead' | 'apiWrite' | 'apiExpensive'` set?
- Is Zod body validation defined (never "trust the client")?
- Returns via `ApiResponse.ok() / ApiResponse.error() / .serverError()`?
- Handler receives `user` from the factory, uses `user.id` (Supabase profile UUID, NOT Clerk ID)?

ENDPOINT DEFINITION:
- Every endpoint listed: method, full route, purpose, who can call it?
- Request body schema (fields, types, required vs optional)?
- Response schema for success AND every error case?
- Pagination params for list endpoints (cursor preferred)?
- Filter and sort params where needed?

VALIDATION & LIMITS:
- All input validated server-side?
- Field length limits enforced?
- Rate limits applied (especially auth, submit, search, AI)?
- File uploads validated for type, size, and actual content (not just extension)?

ZOD GOTCHAS (⚠️  caused a production 422 outage):
- Any `z.string().optional()` for a value that could be `null`? → Use `.nullable().optional()` instead.
- Any `captchaToken: ref.current` that could be `null`? → Conditionally spread the field on the client.
- Any required field that might be missing for guest submissions?

BEHAVIOR & RELIABILITY:
- Are any operations idempotent (safe to retry with same result)?
- If not, is there dedup protection (client-side lock, server-side guard, idempotency key)?
- Long-running operations offloaded to a queue/background job?
- Graceful timeout behavior?
- Retries with exponential backoff for transient failures?

INTERNAL FETCH & SECRETS:
- Any internal `fetch()` call using API_KEY that could be exposed in the client bundle? (V1 endpoints had this bug.)
- Cross-route auth done via headers, not URL params?
- Any secret referenced in a client component, hook, or public route?

THIRD-PARTY FALLBACKS:
- logo.dev 404 → fall back to apple-touch-icon / favicon?
- geo lookup failure → fall back to null / default?
- Email SMTP failure → queue with retry, throttling, dedup?
- Stripe API down → queue or return graceful error?

OUTBOUND WEBHOOKS (⚠️  SSRF prevention):
- Every outbound webhook URL validated via `verifyResolvedIps()`?
- Private IP ranges blocked?
- DNS rebinding protection in place?

VERSIONING & CONTRACTS:
- Will this change break existing clients (mobile, API consumers)?
- V1 API endpoints have separate compatibility rules, touching them?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🗄️   DATABASE LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TYPE SYSTEM REALITY CHECK (⚠️):
- Is this table in `src/shared/types/database.types.ts`?
- If NO (e.g., brand_profiles, custom_domains, user_settings, white_label_settings, CRM tables, lead_scoring tables, stripe_connected_accounts, form_payments, email_queue, etc.): use `(supabaseAdmin as any)` with `// biome-ignore lint/suspicious/noExplicitAny: table not in generated types`.
- DO NOT attempt to fix by adding to database.types.ts, or by casting `as 'forms'` / `as never`. These cascade into 40+ file breakages.
- JSONB fields cast to `Json` type, never `Record<string, unknown>`.

SCHEMA DESIGN:
- Every new table/column defined (name, type, nullable, default, constraints)?
- FKs with correct ON DELETE behavior (CASCADE / SET NULL / RESTRICT)?
- Soft deletes (deleted_at) vs hard deletes for audit/recovery?
- created_at and updated_at on every table?
- User-facing IDs are non-sequential (UUID/short_id) to prevent enumeration?

COLUMN NAME VERIFICATION:
- Column names verified against `database.types.ts` (don't guess: `viewed_at` vs `created_at` wrong name broke dashboard stats once).
- For tables NOT in generated types, verify column names by reading a migration file.

INDEXES & PERFORMANCE:
- Indexes on every column used in WHERE / JOIN / ORDER BY / GROUP BY?
- Composite indexes for multi-column queries?
- Any query doing a full table scan on a large table?
- N+1 query risks in list views or nested data?
- Pagination via cursor (not offset) for large datasets?

MIGRATIONS:
- Written and reversible (up AND down)?
- Does any migration lock a table that could cause downtime?
- Backfill strategy for adding non-nullable columns?
- `IF NOT EXISTS` guards where appropriate?
- Risk of data loss in any step?
- Dropping unused columns: is anything still reading them?

DATA INTEGRITY:
- Multi-table writes wrapped in a transaction?
- What if the transaction fails halfway?
- Unique constraints at the DB level (not just app level)?
- Enum/status values constrained at the DB level?

MULTI-TENANCY (⚠️  workspace isolation):
- Every query scoped to `user_id` OR `workspace_id`?
- `include_unassigned=true` is ONLY passed when `useIsPersonalWorkspace()` is true?
- Non-personal workspaces use strict `.eq('workspace_id', X)` filtering?
- Records with `workspace_id = NULL` ONLY appear in personal workspace?
- IDOR test: can a user manipulate IDs to access another workspace's data?
- RLS OR application-level filters: not mixed confusingly?

CONCURRENT WRITE / TOCTOU:
- Is the limit check (maxResponses, etc.) done in the same transaction as the insert?
- Can two concurrent writes both pass the check before either increments?
- Is there a unique constraint or SELECT FOR UPDATE as a safety net?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔐  SECURITY LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AUTHENTICATION:
- Every protected route/endpoint checks for a valid Clerk session?
- Token expired → silent refresh or forced logout (not silent failure)?
- Any endpoint accidentally left public?

AUTHORIZATION:
- Every action checks "is allowed to do THIS action", not just "logged in"?
- Role/permission checks server-side (not hidden in the UI)?
- Admin role correctly bypasses plan limits? Admin pages check role?
- Workspace-role permissions enforced (owner/admin/editor/viewer)?

IDOR AUDIT (⚠️  real bugs shipped here):
- Can a user change `form_id` / `workspace_id` in URL or body to access another workspace's data?
- Does `verifySession` check workspace MEMBERSHIP, not just auth?
- V1 webhook creation was vulnerable once: is the same pattern elsewhere?

INPUT SECURITY:
- User input sanitized before rendering (XSS prevention via DOMPurify)?
- User input parameterized before DB (never string-concatenated SQL)?
- Any value passed into shell command, eval, or dynamic query?
- Redirects validated to prevent open-redirect attacks?
- CSS user input validated via allowlist parser (NOT regex chain)?

DATA EXPOSURE:
- API response does NOT leak fields the client shouldn't see (passwords, tokens, other users' data, internal IDs)?
- No secrets / API keys / credentials in client-side code or responses?
- Sensitive data masked in logs?

ENVIRONMENT VARIABLES:
- NEVER `process.env.X` directly, always `getEnv()` from @/shared/lib/env.ts?
- Server-only env vars NOT prefixed with NEXT_PUBLIC_?
- Build-phase env validation handled (skipped or provided)?

UPLOADS & FILES:
- Files stored outside web root, not directly URL-accessible?
- File URLs signed/expiring, not permanently public?
- Content validated beyond extension?
- Max file size enforced server-side?
- Per-user storage limits enforced?

COMPLIANCE:
- Does this store PII? Encrypted at rest?
- GDPR data deletion / right to erasure supported?
- Audit logs for sensitive actions?
- Cookie consent respected?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡  PERFORMANCE & SCALABILITY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Expensive queries/computations cached? Invalidation strategy?
- Large lists paginated or virtualized (no 1000+ DOM rows)?
- Images optimized (next/image, lazy load, proper sizing)?
- Large payloads trimmed?
- Any polling: is the interval sane? Should it be SSE/websocket?
- N+1 on the frontend (cascade of requests)?
- Heavy operations (exports, bulk, reports) offloaded to a queue?
- Timeout / cap on any single request?
- Hover prefetch on navigation links where helpful?
- In-memory profile cache used (verifySession)?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔔  NOTIFICATIONS & COMMUNICATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Does this feature trigger emails, push, or in-app notifications?
- Exact trigger, recipient, copy?
- Unsubscribe / preference control?
- Rendering in dark mode and mobile email clients verified?
- Email service down → fail silently or retry via queue (with dedup)?
- In-app real-time (websocket / polling / SSE) needed?
- Vice-versa: if action is undone/cancelled, does the notification also get cancelled / corrected?
- License key / purchase emails use correct sender (orders@)?
- Email domain/branding applied correctly on custom domains?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊  OBSERVABILITY & DEBUGGING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Errors caught and reported to error tracking?
- Server-side events logged for production debugging?
- Logs are structured and include a request/trace ID?
- Sensitive values (passwords, tokens, PII) excluded from logs?
- Analytics events fired for key actions?
- Enough context logged to reproduce a user-reported bug?
- Failed workspace lookups logged (limit-check.ts pattern)?
- Failed email sends logged (verify/route.ts, responses/route.ts)?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🧪  TESTING COVERAGE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Unit tests for business logic / utils?
- Integration tests for API endpoints (with real Supabase where possible)?
- Playwright E2E for critical happy paths?
- Edge cases AND failure paths tested (not just happy)?
- Vice-versa / reverse flow scenarios tested?
- Cross-browser via Playwright (Chromium + Firefox + WebKit)?
- Mobile viewport (375px / 390px) tested?
- Existing test suite still passing (no regressions)?
- Test mocks match current handler implementation signatures?
- Tests can run in CI before any deploy?

SHIP-READINESS (10 LAYERS: never skip layers 1-4):
1. Code completeness: trace full data lifecycle
2. Build verification: tsc --noEmit AND next build
3. Functional E2E: Playwright walks the full flow on staging
4. Data verification: query DB to confirm saved shape matches renderer
5. Visual/UX review: screenshots shown to user
6. Regression: existing tests still pass
7. Cross-browser: Firefox + WebKit, not just Chromium
8. Security & edge-case audit
9. Performance sanity check
10. Post-deploy monitoring plan


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚢  DEPLOY & POST-DEPLOY LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PRE-PUSH:
- `npm run lint && npm run type-check && npm run build` all pass locally?
- `git diff --name-only` shows ONLY files related to this feature?
- package.json AND package-lock.json both staged (if deps changed)?
- Git identity is your developer identity (NEVER `root`)?

DEPLOY FLOW:
- Pushing to `beta` branch, NEVER `master` directly?
- Never auto-triggering Dokploy, user triggers manually?
- After beta→master via PR, reset beta to master (never merge back)?
- Dokploy shallow clones: use `git fetch --unshallow` for history?
- Nixpacks uses Docker cache: same error after a fix may be a cache issue (ask user to clear cache).

POST-DEPLOY VERIFICATION:
- Feature tested on staging before merging to master?
- Staging and production compared side-by-side: anything different?
- `/etc/dokploy/logs/` checked if deploy fails?
- Error tracking monitored for first hour after deploy?

ROLLBACK:
- Migration is reversible (down migration written)?
- Reverts cleanly with `git revert`?
- Feature-flagged so it can be disabled without rolling back?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📝  CONTENT & DOCS SYNC LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Does /features/<x> marketing page match what the feature actually does? (No false claims.)
- Does the help article match current behavior?
- Does the FAQ match current plans?
- Does the pricing page match `plans.ts`?
- Does the chatbot knowledge base match the codebase?
- Does the roadmap still show this as "planned" if it's now shipped?
- Does the sitemap include the new page with correct lastmod?
- Does the new page have SEO metadata (title, description, og:image, canonical, JSON-LD, breadcrumbs)?
- Title is short and punchy, NO pipe separators, NO layout default?
- Does any copy use em dashes? Use colons instead.


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔗  INTEGRATION & REGRESSION CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Touches any shared components / hooks / utilities?
- Could changes to shared code break other existing features?
- Depends on any existing feature working correctly first?
- Interacts with billing / limits / plan gating logic?
- New env vars documented for local / staging / production?
- Feature flag needed so it can be deployed before it's live?
- Affects form builder → did you also check quiz builder?
- Affects renderer → did you check ALL field types still render?
- Affects middleware → did you think 10x before touching it?
- Affects a V1 API endpoint → are external consumers notified?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📦  FINAL COMPLETENESS CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Can a real user complete this end-to-end with zero dev intervention after it ships?
- Anything marked "TODO" / "TBD" / "handle later" / "assume X": resolve it now or flag as a known risk.
- First-run experience considered for new users?
- Onboarding / empty state points them forward?
- Rollback plan documented?
- Migrations reversible?
- New dependency license, maintenance status, bundle size acceptable?
- Every new dashboard route has error.tsx AND loading.tsx?


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Once you have answered all of the above, give me a final summary in EXACTLY this format:

✅ READY: every part of the plan that is fully defined and ready to build
⚠️ ADDED: everything that was missing from the original plan that you just added as part of this stress-test
🚫 NEEDS MY INPUT: open questions, risks, or decisions that need my answer before you start writing code
💣 RISK WATCHLIST: top 3 things most likely to break in production for THIS specific feature, and what would catch them

Do NOT write a single line of code until I review and confirm this summary.
````

---

## License

MIT. Use it, fork it, adapt it. Attribution appreciated but not required.
