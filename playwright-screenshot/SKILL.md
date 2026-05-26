# playwright-screenshot

Capture screenshots of a running local web app at a specified viewport size, navigating through multiple screens, and save them as PNG files.

## Workflow

### 1. Gather requirements

Ask the user (or infer from context) if not already clear:
- **URL** — typically `http://localhost:<port>`. Check which port the dev server is on: `lsof -i :<port> | head -3`.
- **Viewport size** — default is `636×1048` (phone-like). Ask if they want a different size. Common: `390×844` (iPhone 14), `412×915` (Pixel 7), desktop `1280×800`.
- **Screens to capture** — which pages/states to visit. Infer from context if the app structure is visible.
- **Output directory** — default to a `screen/` sibling of the project dir, or ask.

### 2. Set up Playwright

Check and install dependencies as needed:

```bash
# Is Playwright installed locally?
ls node_modules/@playwright/test 2>/dev/null || echo "not installed"

# Install if missing (adds to devDependencies):
npm install -D @playwright/test

# Install Chromium browser if not present:
npx playwright install chromium
```

Create the output directory:
```bash
mkdir -p <output_dir>
```

### 3. Write the screenshot script

Write a `screenshot.mjs` at the project root. Use this template:

```js
import { chromium } from '@playwright/test';
import path from 'path';

const OUT_DIR = '<absolute_output_path>';
const VIEWPORT = { width: 636, height: 1048 };
const URL = 'http://localhost:<port>';

const browser = await chromium.launch();
const page = await browser.newPage();
await page.setViewportSize(VIEWPORT);

await page.goto(URL, { waitUntil: 'networkidle' });

// Wait for any loading spinner to disappear before first screenshot.
// Adjust the selector/text to match what this app shows while loading.
await page.waitForFunction(
  () => !document.body.innerText.includes('로딩 중...'),
  { timeout: 8000 }
);

// --- Screen 1 ---
await page.screenshot({ path: path.join(OUT_DIR, '1_<name>.png') });
console.log('1/<total> saved');

// --- Navigate to next screen ---
// Prefer getByRole over getByText — it avoids ambiguity when the same text
// appears in multiple places (banner copy, nav label, body text, etc.).
await page.getByRole('button', { name: '<label>', exact: true }).click();
await page.waitForTimeout(300);

// --- Screen 2 ---
await page.screenshot({ path: path.join(OUT_DIR, '2_<name>.png') });
console.log('2/<total> saved');

await browser.close();
console.log(`\nAll screenshots saved to ${OUT_DIR}`);
```

**Navigation tips:**
- Use `getByRole('button', { name: '...' })` for bottom nav or tab buttons — `getByText` often matches multiple elements.
- Use `waitForTimeout(300)` after clicks to let transitions settle.
- For pages with async data loading, use `waitForSelector` or `waitForFunction` rather than fixed timeouts.
- If the app shows a loading screen on first visit (SDK timeout, auth check, etc.), use `waitForFunction` to wait for it to clear.

### 4. Run the script

```bash
node screenshot.mjs
```

Verify the output files exist and show file sizes:
```bash
ls -lh <output_dir>
```

### 5. Show the user

Read the saved PNG files using the Read tool so the user can see the results directly in the conversation. If something looks wrong (wrong page, empty screen, clipped content), adjust the script and re-run.

## Common issues

| Problem | Fix |
|---------|-----|
| `Cannot find package '@playwright/test'` | Run `npm install -D @playwright/test` in the project root |
| Browser not installed | Run `npx playwright install chromium` |
| `strict mode violation: getByText(...)` | Switch to `getByRole('button', { name: '...', exact: true })` |
| Page still shows loading state | Increase timeout in `waitForFunction`, or wait for a specific element to appear |
| Screenshots are blank / all white | The page may use canvas or WebGL — try `{ fullPage: false }` and a longer wait |
| Content cut off | Increase `height` in `VIEWPORT` |
