# Helpers — full implementation

Copy these into `e2e/helpers/tour-helpers.ts`. Everything is dependency-free (only `@playwright/test` + Node built-ins) and app-agnostic except the sidebar/login selectors you swap per app.

```ts
import { expect, test, type Page, type TestInfo } from '@playwright/test';
import fs from 'node:fs';
import path from 'node:path';

const HERE = import.meta.dirname; // use __dirname if your package is CommonJS
export const TEST_PREFIX = `E2E-${Date.now()}`;
export const SCREENSHOT_DIR = path.join(HERE, '..', 'screenshots');
export const DOWNLOAD_DIR = path.join(HERE, '..', '..', 'test-results', 'downloads');
export const SUMMARY_PATH = path.join(HERE, '..', '..', 'test-results', 'tour-summary.md');

export interface StepRecord { index: number; name: string; status: 'passed' | 'failed'; error?: string; }
export const stepRecords: StepRecord[] = [];

/** Credentials: seed-derived defaults, overridable via env. Never commit real secrets. */
export function credentials(role: 'admin' | 'manager' | 'user' = 'admin') {
  const e = process.env;
  if (role === 'manager') return { email: e.E2E_MANAGER_EMAIL ?? 'manager@localhost', password: e.E2E_MANAGER_PASSWORD ?? 'Manager@123' };
  if (role === 'user')    return { email: e.E2E_USER_EMAIL    ?? 'user@localhost',    password: e.E2E_USER_PASSWORD    ?? 'User@123' };
  return { email: e.E2E_ADMIN_EMAIL ?? 'admin@localhost', password: e.E2E_ADMIN_PASSWORD ?? 'Admin@123' };
}

export function ensureTourDirs() {
  fs.mkdirSync(SCREENSHOT_DIR, { recursive: true });
  fs.mkdirSync(DOWNLOAD_DIR, { recursive: true });
}

/** Navigate and wait for the app shell so lazy route chunks are mounted. */
export async function goto(page: Page, pathName: string) {
  await page.goto(pathName);
  await page.locator('header').first().waitFor({ state: 'attached', timeout: 20000 });
}

let captionCounter = 0;

/** Persistent bottom banner; survives SPA navigation and full reloads. dir='ltr' for non-RTL apps. */
export async function installCaption(page: Page, text: string) {
  captionCounter += 1;
  const id = `e2e-tour-caption-${captionCounter}`;
  const source = `(() => {
    const bannerId = ${JSON.stringify(id)};
    const bannerText = ${JSON.stringify(text)};
    const mount = () => {
      if (document.getElementById(bannerId)) return;
      const el = document.createElement('div');
      el.id = bannerId; el.dir = 'rtl'; el.textContent = bannerText;
      Object.assign(el.style, {
        position:'fixed', left:'0', right:'0', bottom:'0', zIndex:'2147483647',
        background:'rgba(15,23,42,.92)', color:'#fff',
        font:'600 20px/1.6 "Segoe UI",Tahoma,Arial,sans-serif',
        textAlign:'center', padding:'10px 18px', boxShadow:'0 -4px 18px rgba(0,0,0,.35)',
        pointerEvents:'none',
      });
      const place = () => { if (document.body) document.body.appendChild(el); };
      place();
      if (!document.body) new MutationObserver(place).observe(document.documentElement,{childList:true});
    };
    mount();
    new MutationObserver(mount).observe(document.documentElement,{subtree:true,childList:true});
  })();`;
  await page.addInitScript(source);
  await page.evaluate(source).catch(() => {});
}

let pauseMs = 600;
export function setPause(ms: number) { pauseMs = ms; }

/** One tour section: numbered step, caption, pause, screenshot, catch + recover + continue. */
export async function step(page: Page, testInfo: TestInfo, title: string, fn: () => Promise<void>) {
  void testInfo;
  const n = stepRecords.length + 1;
  const padded = String(n).padStart(2, '0');
  await test.step(`[${padded}] ${title}`, async () => {
    await installCaption(page, `Step ${padded} — ${title}`).catch(() => {});
    await page.waitForTimeout(pauseMs);
    try {
      await fn();
      stepRecords.push({ index: n, name: title, status: 'passed' });
      await page.screenshot({ path: path.join(SCREENSHOT_DIR, `step-${padded}.png`) }).catch(() => {});
    } catch (error) {
      const message = error instanceof Error ? error.message.split('\n').slice(0, 6).join('\n') : String(error);
      stepRecords.push({ index: n, name: title, status: 'failed', error: message });
      await page.screenshot({ path: path.join(SCREENSHOT_DIR, `step-${padded}-FAILED.png`) }).catch(() => {});
      console.error(`\n[tour] step ${padded} FAILED: ${title}\n${message}\n`);
      await page.goto('/dashboard', { waitUntil: 'domcontentloaded' }).catch(() => {});
      await page.waitForTimeout(1200);
    }
  });
}

export async function login(page: Page, role: 'admin' | 'manager' | 'user' = 'admin') {
  const { email, password } = credentials(role);
  await page.goto('/login');
  await page.locator('#login-username').fill(email);
  await page.locator('#login-password').fill(password);
  await page.getByRole('button', { name: 'دخول' }).click();
  await page.waitForURL('**/dashboard', { timeout: 20000 });
  await expect(page).toHaveURL(/\/dashboard/);
}

export async function logout(page: Page) {
  const menuBtn = page.locator('[aria-label="قائمة المستخدم"], [aria-label="الحساب"]').first();
  if (await menuBtn.isVisible().catch(() => false)) {
    await menuBtn.click();
    const item = page.getByRole('menuitem', { name: /تسجيل الخروج/ }).first();
    if (await item.isVisible().catch(() => false)) await item.click();
  }
  await page.waitForURL('**/login', { timeout: 15000 }).catch(async () => { await page.goto('/login'); });
}

export function handleDialogs(page: Page) {
  page.on('dialog', async (d) => {
    try { await (d.type() === 'prompt' ? d.accept('E2E') : d.accept()); } catch { /* gone */ }
  });
}

export function handlePopups(page: Page) {
  page.on('popup', async (p) => {
    await p.waitForLoadState('domcontentloaded', { timeout: 8000 }).catch(() => undefined);
    await p.close().catch(() => undefined);
  });
}

export async function captureDownload(page: Page, trigger: () => Promise<void>) {
  const [download] = await Promise.all([page.waitForEvent('download', { timeout: 30000 }), trigger()]);
  const target = path.join(DOWNLOAD_DIR, download.suggestedFilename());
  await download.saveAs(target);
  return download.suggestedFilename();
}

export function writeSummary(totalSeconds: number) {
  const passed = stepRecords.filter((r) => r.status === 'passed').length;
  const failed = stepRecords.filter((r) => r.status === 'failed');
  const mm = String(Math.floor(totalSeconds / 60)).padStart(2, '0');
  const ss = String(Math.floor(totalSeconds % 60)).padStart(2, '0');
  const rows = stepRecords.map((r) =>
    `| ${String(r.index).padStart(2, '0')} | ${r.name} | ${r.status === 'passed' ? '✅' : '❌'} | ${
      r.error ? r.error.replace(/\|/g, '/').replace(/\n/g, ' ⏎ ').slice(0, 300) : ''
    } |`).join('\n');
  const md = [
    `# Tour summary`, '',
    `- **Date:** ${new Date().toISOString()}`,
    `- **Video length:** ${mm}:${ss}`,
    `- **Steps:** ${stepRecords.length} — passed: ${passed} — failed: ${failed.length}`, '',
    `| # | Step | Status | Details |`, `|---|------|--------|---------|`, rows, '',
    failed.length ? `## Failed\n\n${failed.map((f) => `- **${f.name}**: ${f.error}`).join('\n')}` : '_No failed steps._', '',
  ].join('\n');
  fs.mkdirSync(path.dirname(SUMMARY_PATH), { recursive: true });
  fs.writeFileSync(SUMMARY_PATH, md, 'utf8');
  console.log(`\n===== TOUR SUMMARY =====\nSteps: ${stepRecords.length} | Passed: ${passed} | Failed: ${failed.length} | Video: ${mm}:${ss}\n${SUMMARY_PATH}\n========================\n`);
}

export { expect };
```

## Spec skeleton

```ts
import { test } from '@playwright/test';
import path from 'node:path';
import { captureDownload, ensureTourDirs, expect, goto, handleDialogs, handlePopups,
         login, logout, step, stepRecords, writeSummary, TEST_PREFIX } from './helpers/tour-helpers';

test.describe.configure({ mode: 'serial' });

test('Full continuous tour', async ({ page }, testInfo) => {
  test.setTimeout(45 * 60 * 1000);
  ensureTourDirs();
  handleDialogs(page);
  handlePopups(page);
  const startedAt = Date.now();

  await step(page, testInfo, 'Login + dashboard', async () => {
    await login(page, 'admin');
    await expect(page.locator('h1').first()).toBeVisible();
  });

  // ... one `step(...)` per feature section ...

  const video = page.video();
  if (video) {
    await testInfo.attach('tour', { path: await video.path(), contentType: 'video/webm' }).catch(() => {});
    await video.saveAs(path.join('..', 'test-results', 'store-tour.webm')).catch(() => {});
  }
  writeSummary((Date.now() - startedAt) / 1000);
  const failed = stepRecords.filter((r) => r.status === 'failed');
  expect(failed.length, `Tour steps failed:\n${failed.map((f) => `- ${f.name}: ${f.error}`).join('\n')}`).toBe(0);
});
```
