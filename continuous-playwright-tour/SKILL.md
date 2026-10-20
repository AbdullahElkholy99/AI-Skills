---
name: continuous-playwright-tour
description: Build a single, continuous end-to-end Playwright "tour" test that walks an entire web application in one browser session and produces one uninterrupted video, numbered screenshots, a trace, downloaded exports, and a markdown summary. Use when asked to create an app walkthrough video, a smoke test that exercises every feature, a demo recording, a QA tour, or an E2E suite that must not modify application source code.
---

# Continuous Playwright Tour

> One test. One browser session. One continuous video. Every feature walked, screenshotted, and reported — without touching application source code.

## Scope

Use this skill when you:
- Need a single video that demonstrates the whole application end to end (demos, handover, marketing, QA sign-off).
- Want a smoke tour that opens and exercises every screen, form and dialog in one run.
- Must produce a per-step report (pass/fail) plus artefacts (video, screenshots, trace, exported files).
- Are asked to "record the app", "walk through all features", "make a tour test", "produce a demo video", or "add an end-to-end tour".
- Need the tour to run in CI on demand and publish the video as an artefact.

Do NOT use this skill for fast unit-level E2E suites. A tour is deliberately slow (slowMo + pauses) and forgiving (a failed step must not abort the run).

## Core Principles

### 1. One test = one video
Playwright emits **one video per test**. Splitting the tour into multiple `test()` blocks means multiple videos. Keep the entire walkthrough in a single `test()` and never call `page.close()` or open extra contexts mid-test.

### 2. A failing step must not kill the tour
Wrap every section in a helper that catches errors, records the failure, screenshots it, recovers to a known-good page (`/dashboard`), and continues. The run only fails at the very end if any step failed — after the video and report are written.

### 3. Never modify application source
Drive the UI only. If a flow cannot be completed through the UI, degrade the step (open the form, close it) and document why in the summary rather than patching the app.

### 4. Reconnaissance before assertions
Do not guess locators. First run a throwaway exploration spec that dumps buttons, inputs, roles and aria-labels for every screen, then write the tour against the real DOM.

### 5. The video is a narrative
Add a persistent on-screen caption banner per step (`الخطوة 03 — ...` / `Step 03 — ...`) and a short pause so a human can follow the recording. Inject the banner with `addInitScript` so it survives SPA navigation.

## Rules

**DO**
- Use `test.describe.configure({ mode: 'serial' })` and `workers: 1`.
- Use `test.setTimeout(45 * 60 * 1000)` and a matching `timeout` in the config.
- Install one `page.on('dialog')` handler that accepts native dialogs, and one `page.on('popup')` handler that closes stray windows (PDF/print).
- Save exactly one video with `await page.video().saveAs('…/store-tour.webm')` **after** the test body finishes.
- Namespace all test data with a unique prefix, e.g. `${'E2E-' + Date.now()}-...`.
- Read credentials from env vars (`E2E_ADMIN_PASSWORD`, …) with seed-derived defaults as fallback. Never commit secrets.
- Screenshot every step with a zero-padded index (`step-01.png`) into a stable folder.
- Write a `tour-summary.md` table (step, status, error) and print it to the console.

**DON'T**
- Don't split the tour into several `test()` calls.
- Don't `page.close()` / create a second context mid-tour.
- Don't rely on `getByRole('button', { name: 'حفظ (Alt+S)' })` when the shortcut is only in `title` — the accessible name is just `حفظ`. Match the visible label exactly.
- Don't use `{ name: 'X' }` without `exact: true` for short Arabic labels (`الكل`, `تطبيق`) — they substring-match unrelated buttons.
- Don't hardcode selectors you have not seen in the DOM.
- Don't delete or edit records the tour did not create.

## Anatomy

```
project/
├─ playwright.config.ts          # tour project: 1 worker, 45min, video+screenshot+trace "on", slowMo
├─ e2e/
│  ├─ tour.spec.ts               # THE single test (many test.step sections)
│  ├─ helpers/tour-helpers.ts    # caption, step wrapper, login/logout, downloads, summary
│  └─ screenshots/               # step-01.png, step-02.png, …
└─ ../test-results/
   ├─ store-tour.webm            # the one continuous video
   ├─ tour-summary.md            # pass/fail table
   └─ downloads/                 # exported PDF/Excel/CSV files
```

### Helpers you will need
- `login(page, role)`, `logout(page)`
- `installCaption(page, text)` — persistent RTL/LTR banner via `addInitScript` + `MutationObserver` re-mount
- `step(page, testInfo, title, fn)` — numbered `test.step`, caption, pause, screenshot, catch + recover + continue
- `handleDialogs(page)`, `handlePopups(page)`
- `captureDownload(page, trigger)` — `waitForEvent('download')` + `saveAs`
- `writeSummary(seconds)` — markdown table + console print
- `goto(page, path)` — always wait for the app shell (`header`) after navigation (lazy route chunks)

See `references/helpers.md` for a full, copy-pasteable implementation.

## The Step Wrapper (heart of the tour)

```ts
export async function step(page: Page, testInfo: TestInfo, title: string, fn: () => Promise<void>) {
  const n = stepRecords.length + 1;
  const padded = String(n).padStart(2, '0');
  await test.step(`[${padded}] ${title}`, async () => {
    await installCaption(page, `Step ${padded} — ${title}`).catch(() => {});
    await page.waitForTimeout(pauseMs);
    try {
      await fn();
      stepRecords.push({ index: n, name: title, status: 'passed' });
      await page.screenshot({ path: `e2e/screenshots/step-${padded}.png` }).catch(() => {});
    } catch (error) {
      stepRecords.push({ index: n, name: title, status: 'failed', error: message(error) });
      await page.screenshot({ path: `e2e/screenshots/step-${padded}-FAILED.png` }).catch(() => {});
      await page.goto('/dashboard', { waitUntil: 'domcontentloaded' }).catch(() => {}); // recover
    }
  });
}
```

## The Final Wrap-Up (order matters!)

```ts
// 1. attach + save the ONE video (before asserting, so the artefact always exists)
const video = page.video();
if (video) {
  await testInfo.attach('store-tour', { path: await video.path(), contentType: 'video/webm' }).catch(() => {});
  await video.saveAs(path.join('..', 'test-results', 'store-tour.webm')).catch(() => {});
}
// 2. write the summary
writeSummary((Date.now() - startedAt) / 1000);
// 3. only NOW fail if any step failed
expect.soft(stepRecords.filter(r => r.status === 'failed').length, 'Tour steps failed').toBe(0);
```

## Reconnaissance Workflow (do this first)

1. Write `explore.spec.ts` that logs in and, for each route, dumps `button`/`input`/`select` text + `title` + `aria-label` + `disabled`.
2. Run it with a throwaway config (no video, no slowMo) → fast.
3. Grep the output; build the tour from real attributes.
4. Delete the exploration files before delivery.

Typical traps this reveals: shortcut text living only in `title`; `role="dialog"` elements identified by `aria-label` but with no `<h1>`; two breadcrumbs with `aria-current="page"` (use `.last()`); disabled primary actions that need a prior selection.

## Manifests (config + scripts)

`playwright.config.ts`:
```ts
export default defineConfig({
  testDir: './e2e',
  testMatch: /tour\.spec\.ts/,
  workers: 1, retries: 0, fullyParallel: false,
  timeout: 45 * 60 * 1000,
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:5174',
    locale: 'ar-EG', viewport: { width: 1366, height: 768 },
    video: { mode: 'on', size: { width: 1366, height: 768 } },
    screenshot: 'on', trace: 'on', launchOptions: { slowMo: 300 },
  },
});
```
`package.json`: `"test:tour": "playwright test --project=tour"`, `"test:tour:headed": "playwright test --project=tour --headed"`.

> ESM note: if `package.json` has `"type": "module"`, use `import.meta.dirname` (not `__dirname`) in configs and specs.

## CI

Publish the video as an artefact from an on-demand workflow (`workflow_dispatch`): start the DB/API/web, run `npx playwright test --project=tour`, upload `test-results/**` and `store-tour.webm`. See `references/github-actions.md`.

## Checklist

- [ ] Exactly one `test()`; `mode: 'serial'`; `workers: 1`.
- [ ] No `page.close()` / extra context mid-test.
- [ ] Every section wrapped in the recovering `step()` helper.
- [ ] Persistent caption banner survives navigation.
- [ ] Credentials from env, seed defaults only as fallback.
- [ ] Test data namespaced with a timestamp prefix.
- [ ] Dialogs and popups auto-handled.
- [ ] `saveAs` the single video + write summary **before** the final assertion.
- [ ] Exploration/throwaway files deleted.
- [ ] CI publishes the video artefact.

## Cross-references

- `references/helpers.md` — full helper implementation.
- `references/github-actions.md` — on-demand video workflow.
- `references/lessons-store-management.md` — worked example: real locator traps and fixes from the Store Management SPA tour.
- Related skills: `accessibility` (labels/roles), `usability` (flows), `ui-patterns` (what to exercise).
