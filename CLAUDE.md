# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Commands

- `npm run dev` — start the dev server (Next.js App Router, port 3000 by default)
- `npm run build` — production build
- `npm run start` — run the production build
- `npm run lint` — ESLint (`eslint-config-next` core-web-vitals + typescript)

There is no test script/framework configured in this repo.

## Architecture

This is a **client-only demo app — there is no backend and no database.** Everything runs in the browser.

- **State**: one Zustand store per domain in `src/store/*.ts` (`projectStore`, `financeStore`, `meetingStore`, `settingsStore`, `dashboardStore`, `workspaceStore`, `alertStore`, `sessionStore`, …). Each store persists itself to `localStorage` via the shared helpers in `src/lib/localPersist.ts` (`loadLocal`/`persistLocal`, 400ms-debounced writes), falling back to bundled sample data in `src/data/*.ts` when nothing is persisted yet.
- **Bootstrap**: `src/components/shell/DataBootstrap.tsx` hydrates every store on mount (seed data or localStorage) and also subscribes to the browser's `storage` event so a second tab of the same origin (e.g. a Sponsor tab and a Partner tab open side by side) picks up changes live, re-running the exact same `hydrate` each store uses on initial load. It also runs a 5-minute interval to re-evaluate `alertStore` (there's no backend scheduler, so alerts are just recomputed on a timer/on load) and a fire-and-forget migration (`complianceFileStore.ts`) that shrinks old inline file data out of localStorage.
- **File storage**: uploaded compliance documents go through `src/lib/complianceFileStore.ts`, which stores file bytes in IndexedDB (not localStorage) specifically to avoid hitting localStorage's size quota — don't add new large-blob storage into a Zustand-persisted store directly.
- **Auth/roles**: mock auth only — `sessionStore.loginWithCredentials` accepts any non-empty password and auto-detects role by matching the entered email against `settingsStore`'s partner directory; a match logs in as `partner`, otherwise `sponsor`. Every account is one of exactly two roles (`UserRole` in `src/store/sessionStore.ts`): `sponsor` (the main org, full access) or `partner` (an external collaborator — Partner/NGO/Implementing Partner/Co-Sponsor all collapse into this one role by product spec).
- **Route guarding**: `src/components/shell/AppShell.tsx` is the single navigation authority — it stage-gates every route (unauthenticated → `/login`, unverified → `/register/verify`, not-onboarded → `/onboarding`, then role vs. path) using `src/lib/permissions.ts` (`SPONSOR_ONLY_PREFIXES`/`PARTNER_ONLY_PREFIXES`, `isPathAllowedForRole`, `filterNavGroupsForRole`) and `src/lib/roleRouting.ts` (`getDashboardPathForRole`, the shared post-login landing path). Cross-role/cross-boundary navigation uses a hard `window.location` redirect, not `router.push` — client-side transitions across these boundaries proved unreliable, so keep that pattern when touching auth/role redirects.
- **Shared resource ownership**: some per-project resources (WBS, the Beneficiary Indicator Form) are single shared objects with one of three ownership modes set by the Sponsor (`self`/`partner`/`both`), gating who can mutate vs. only view — see `canEditProjectResource` in `src/lib/permissions.ts`. It's resource-agnostic on purpose (same mode/role inputs for both features); enforce new resources like it the same way, at both the UI layer and the store-action layer (mutations must refuse even if called directly, not just hide the button).
- **Path alias**: `@/*` → `src/*` (see `tsconfig.json`).
- **Directory layout**: `src/app` (Next.js routes), `src/components/<domain>` (mirrors the route/feature domains: `finance`, `meetings`, `partners`, `projects`, `settings`, `dashboard`, `compliance`, `reporting`, `onboarding`, `alerts`, `shell`, `ui`, `marketing`), `src/store`, `src/data` (seed data), `src/lib` (domain logic/helpers), `src/types`.

## UAT via Claude in Chrome

When doing manual UAT through Claude in Chrome (e.g. the `uat-sponsor` / `uat-partner` agents):
- If the Claude in Chrome extension's browser-control tool is available in the current session's tool list, use it directly to drive the UAT flow end-to-end without stopping to ask "should I run this?" first — just run it and report results/findings.
- Never use screenshots to check page state. Use DOM inspection / rendered component/accessibility tree reads instead — screenshots are slow and burn time for no accuracy gain here. Only fall back to a screenshot if the user explicitly asks for one (e.g. a visual/layout bug).
- If no such browser-control tool is present in the session (this is a capability gap, not a permission decision — check the actual tool list, don't assume), say so plainly and hand the user the exact click-through steps instead of pretending to have run it.
