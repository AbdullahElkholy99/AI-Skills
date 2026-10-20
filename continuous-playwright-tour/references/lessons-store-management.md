# Worked example — Store Management SPA tour (real traps + fixes)

Concrete findings from building a 64-step Arabic RTL tour (React 19 + Vite SPA, .NET API). Use these as a checklist of the kinds of things reconnaissance will surface.

## Environment facts worth pinning early

- **Login form:** `#login-username`, `#login-password`, submit button label `دخول`. The API body was `{ UserName, Password }` — the field is not always called `email`.
- **Port collisions:** default `5173` was taken by another project. Serve on `5174` with `--strictPort`; make sure the API CORS allowlist includes it.
- **ESM package (`"type":"module"`):** `__dirname` is undefined in `playwright.config.ts` **and** specs → use `import.meta.dirname`.

## Locator traps (and the fixes)

| Symptom | Real cause | Fix |
|---|---|---|
| `getByRole('button', { name: 'حفظ (Alt+S)' })` times out | Shortcut lives in `title`; accessible name is only the visible text `حفظ`. | `page.locator('button[title="حفظ (Alt+S)"]')` or `getByRole('button', { name: 'حفظ', exact: true })`. |
| Strict-mode violation on `الكل` / `تطبيق` | Default `name` match is substring; `تطبيق` also matched `قائمة التطبيقات (Ctrl+K)`. | Add `exact: true`. |
| Breadcrumb `toHaveText` always wrong | Page renders **two** `[aria-current="page"]` (group + page). | Use `.last()`. |
| Dashboard has no breadcrumb marker | Dashboard is not a ControlPanel page; it renders its own `<h1>`. | Wait on `h1` text for the dashboard; use the breadcrumb only for panel pages. |
| Launcher dialog "not found" | It is `role="dialog"` with `aria-label="قائمة التطبيقات"` — there is **no** matching heading. | `locator('[role="dialog"][aria-label="قائمة التطبيقات"]')`. |
| `#null` selector | `'#' + await el.getAttribute('id')` when the select has no id. | Select by index/locator, not by reconstructed id. |
| Primary action "not found" | Button label is `فاتورة جديدة`; `(Alt+N)` is only in `title`. | Match the visible label (with `exact: true`). |
| Report filter `تطبيق` button missing | Inline preset chips apply immediately; `تطبيق`/`إعادة تعيين` only exist in an advanced popover. | Click the preset chip (`هذا الشهر`), don't look for `تطبيق`. |
| Export button click times out | Export control is `disabled` while the report loads. | `await expect(exportBtn).toBeEnabled()` before clicking. |
| Save/confirm `disabled` | Form validation needs a real selection, not free text. | Click the autocomplete option (`li[role="option"]`) so the underlying id is set. |
| Autocomplete "no options" | Options render as `li[role="option"]` inside `[role="listbox"]`, not native `<option>`. | Target the listbox + option list, wait for it to be visible. |

## Flows that are the hardest to automate

- **Multi-select autocompletes** (supplier/product): type → wait for `[role="listbox"]` → click the matching `li[role="option"]`. Typing text alone does not set the id.
- **Nested modals** (create-return inside a returns modal): assert each heading before proceeding; close via the header `button[aria-label="إغلاق"]`.
- **Native prompts** (`window.prompt` for cash-close): auto-accept via the global dialog handler.
- **Print/PDF popups:** the row "print" opens a new window — either avoid it (use the `/…/print` route) or close popups via the global handler.
- **Lazy routes that render empty at a fixed timeout:** always `goto` + wait for the shell or a heading; never rely on `waitForTimeout` alone.

## Degrade gracefully

If a flow cannot be completed through the UI (e.g. an endpoint returns HTML so a picker stays empty), do **not** patch the app. Open the modal, assert its heading, close it, and record the limitation in the summary. The tour's job is coverage + evidence, not to fix bugs.

## Artefacts produced

- `web/test-results/store-tour.webm` — the single continuous video (≈100–150 MB for a 30–45 min tour at 1366×768).
- `web/test-results/tour-summary.md` — the pass/fail table.
- `web/client-side/e2e/screenshots/step-NN.png` (+ `-FAILED.png`).
- `web/test-results/downloads/` — every exported PDF/Excel/CSV.
- `web/playwright-report/` — HTML report; trace under `test-results/artifacts/`.
