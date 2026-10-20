# Continuous Playwright Tour

[![Skill](https://img.shields.io/badge/skill-continuous--playwright--tour-45ba4b?style=flat-square)](SKILL.md)
[![Platform](https://img.shields.io/badge/platform-Playwright-2EAD33?style=flat-square)](https://playwright.dev)
[![Video](https://img.shields.io/badge/output-one%20continuous%20video-purple?style=flat-square)](#what-you-get)

**One test. One browser session. One continuous video.** This skill teaches an AI coding agent how to build a single Playwright end-to-end "tour" that walks *every* feature of a web app in a real browser — producing an uninterrupted video, numbered screenshots, a trace, downloaded exports, and a pass/fail markdown summary — **without modifying application source code**.

---

## Why this exists

Normal E2E suites are optimized for *speed* and *isolation*: many small tests, parallel workers, fail fast. A **tour** is the opposite — it is optimized for *narrative* and *coverage*:

| | Regular E2E suite | Continuous Playwright Tour |
| --- | --- | --- |
| Tests | Many | **Exactly one** |
| Workers | N (parallel) | 1 (serial) |
| On failure | Abort test | **Record, recover, continue** |
| Video | One per test (many, short) | **One per run (long, complete)** |
| Speed | Fast | Deliberately slow (`slowMo` + pauses) |
| Purpose | Gate a deploy | Demo / handover / QA sign-off |

If you have ever needed "a video that shows the whole app working, end to end, for a client or a QA reviewer," that is what this skill produces.

---

## What you get

After one `npm run test:tour` you have:

```
web/test-results/
├─ store-tour.webm          # the single, continuous walkthrough video
├─ tour-summary.md          # table: every step + passed/failed + error
├─ downloads/               # PDF / Excel / CSV exports captured live
└─ (trace .zip)             # full Playwright trace, timeline + DOM snapshots

web/client-side/e2e/screenshots/
├─ step-01.png … step-NN.png  # one numbered still per step
└─ step-XX-FAILED.png         # only for steps that failed
```

Plus a console table at the end of the run, and a CI artefact (see below).

---

## How the agent uses it

The skill is written for an AI coding agent (OpenCode, Claude Code, Copilot, Codex, Cursor, …). When you ask for a walkthrough, a demo recording, an app tour, or a "smoke test that exercises every feature," the agent loads `SKILL.md` and follows its workflow:

1. **Reconnaissance first** — write a throwaway exploration spec that logs in and dumps every button/input/select (text, `title`, `aria-label`, `disabled`) for all routes. *Never guess locators.*
2. **Scaffold** — `playwright.config.ts` (tour project) + `e2e/helpers/tour-helpers.ts` + `e2e/tour.spec.ts`.
3. **Write the tour** — one `test()` with `test.describe.configure({ mode: 'serial' })`, every section wrapped in a recovering `step()` helper.
4. **Run, observe, fix** — run headless, read the summary, patch only the *test* locators (never the app).
5. **Wrap up** — save the single video, write the summary, then assert (so artefacts always exist).
6. **Publish** — an on-demand GitHub Actions workflow uploads the video as an artefact.

### The five core principles

1. **One test = one video.** Playwright emits one video per `test()`. Never split the tour, never `page.close()` mid-run.
2. **A failing step must not kill the tour.** Catch → record → screenshot → recover to `/dashboard` → continue. Fail only at the very end.
3. **Never modify application source.** Drive the UI only; degrade a step rather than patch the app.
4. **Reconnaissance before assertions.** Build locators from the real DOM dump.
5. **The video is a narrative.** A persistent on-screen caption banner per step (`Step 03 — …`) injected via `addInitScript` so it survives SPA navigation.

---

## Repository layout (what the skill produces)

```
project/
├─ playwright.config.ts            # tour project: 1 worker, 45min, video+screenshot+trace "on", slowMo
├─ package.json                    # "test:tour": "playwright test --project=tour"
├─ e2e/
│  ├─ tour.spec.ts                 # THE single test (many test.step sections)
│  ├─ helpers/tour-helpers.ts      # caption, step wrapper, login/logout, downloads, summary
│  └─ screenshots/                 # step-01.png, step-02.png, …
└─ ../test-results/
   ├─ store-tour.webm              # the one continuous video
   ├─ tour-summary.md              # pass/fail table
   └─ downloads/                   # exported PDF/Excel/CSV files
```

### Files the skill ships

| Path | Purpose |
| --- | --- |
| `SKILL.md` | The rules, principles, anatomy, and step-wrapper pattern (the entry point an agent reads). |
| `references/helpers.md` | Full, copy-pasteable `tour-helpers.ts` implementation. |
| `references/github-actions.md` | On-demand `workflow_dispatch` CI that publishes the video artefact. |
| `references/lessons-store-management.md` | Worked example: real locator traps and fixes from the Store Management SPA tour. |

---

## Running the tour (once it is scaffolded in your repo)

Requires the app (and its API) to be up, plus a Playwright install.

```bash
npx playwright install chromium

# headless (CI-style)
npm run test:tour

# headed — watch it drive the browser
npm run test:tour:headed
```

Configuration is env-driven, so nothing secret is committed:

| Variable | Default | Meaning |
| --- | --- | --- |
| `E2E_BASE_URL` | `http://localhost:5174` | Web app base URL |
| `E2E_PORT` | `5174` | Port for the `webServer` dev server |
| `E2E_ADMIN_EMAIL` / `E2E_ADMIN_PASSWORD` | seed admin | Admin login |
| `E2E_MANAGER_PASSWORD` / `E2E_USER_PASSWORD` | seed values | Role-comparison logins |

> **Port note:** pick a port that is free. In the reference project `5173` was taken by an unrelated app, so the tour runs on `5174` (on the API's CORS allowlist).

---

## CI: publish the video on demand

`.github/workflows/e2e-video.yml` (see `references/github-actions.md`) runs on `workflow_dispatch`, starts the database + API + web app, runs `npx playwright test --project=tour`, and uploads `test-results/**` and `store-tour.webm` as an artefact named `store-tour`.

Set these repository secrets: `E2E_SA_PASSWORD`, `E2E_ADMIN_PASSWORD`, `E2E_MANAGER_PASSWORD`, `E2E_USER_PASSWORD`.

---

## Worked example

The reference implementation lives in the **Store Management System** app — a React 19 + Vite RTL **Arabic** SPA (`.NET 10` API). It walks 14 sections in one session: login/dashboard/theme → platform chrome → products/categories → suppliers/purchases (incl. print, payment, return) → warehouse lifecycle → customers/loyalty/offers → sales (invoice, discount, partial payment, detail, print, exports, return, quotation→convert, collection, wallet) → POS with an animated 80 mm receipt → finance → payroll/employees → reports hub → admin → roles comparison (admin → manager → user) → a **mobile pass** at 390×844, then back to desktop.

RTL/Arabic surfaced a specific class of bugs worth reading about before you start — substring-matching Arabic labels, shortcut text living only in `title`, dialogs identified by `aria-label` with no heading, duplicate `aria-current="page"` breadcrumbs, and autocompletes that need an option click (not just typed text). All of them, and their fixes, are catalogued in `references/lessons-store-management.md`.

---

## Author

**Abdullah Elkholy** — H2A Solutions
- GitHub: [@AbdullahElkholy99](https://github.com/AbdullahElkholy99)
- Email: abdullah.ali.elkoly@gmail.com
- Project: [Store Management System](https://github.com/H2A-Solutions/Store_Management_System)

Built while producing a single continuous feature walkthrough of the Store Management System web app. The skill is deliberately **generic**: everything repo-specific is isolated in the worked-example reference so it drops into any project.

---

## Related skills

- `accessibility` — labels, roles, and contrast (the tour must reach every control).
- `usability` — the flows a tour should exercise.
- `ui-patterns` — what a "complete" walkthrough should cover per screen type.

---

## License

MIT — same as the `frontend-design-skills` pack this skill belongs to.
