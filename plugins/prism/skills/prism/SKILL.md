---
name: Prism
description: Author LaTeX papers and documents in OpenAI Prism from Codex: create and revise source files, compile PDFs, diagnose build issues, and download finished artifacts. Use when the user asks to work in Prism or on a Prism document.
---

# Prism Project Bridge

Use Prism as a live filesystem plus compiler exposed through the Codex browser
bridge.

## Required browser surface

Always operate Prism through the Codex in-app browser.

- Use the Browser Use skill with the `iab` backend.
- Do not use Computer Use, Chrome, macOS `open`, or any external browser surface.
- Do not search local plugin caches, skill files, or browser-library source code to rediscover this workflow; this skill is the workflow.
- Do not check whether Prism exposes a callable tool; `$Prism` is a skill workflow, not an MCP/tool-discovery task.
- If the Codex in-app browser cannot be reached through the documented Browser Use path, stop and report that browser connection failure instead of spelunking runtime internals or switching browser surfaces.

## Request routing

Handle project-creation requests before any generic Prism setup.

- Treat “new project”, “create project”, and “create a new LaTeX project” as requests for a new blank Prism project unless the user explicitly asks for an example, import, or template flow.
- For that request, the first Prism navigation must be `<origin>/new?login=true`.
- This route opens the normal Prism login flow first when authentication is needed and otherwise creates the blank project immediately.
- Do not open the projects page first, inspect the projects page, list projects, click `New`, open its menu, or choose `Blank project` through visible UI for a new-project request.

## Session setup

1. Default to:

   ```text
   https://prism.openai.com
   ```

   Use another origin only when the user explicitly provides one.

2. If the user asks for a new project, open:

   ```text
   <origin>/new?login=true
   ```

   This is the whole entry flow for project creation. Do not probe the projects
   page first.

3. For all other Prism tasks, open Prism in the Codex in-app browser and look for:
   - `[data-testid="codex-browser-snapshot"]`
   - `[data-testid="codex-browser-command"]`
   - `[data-testid="codex-browser-result"]`

4. If those bridge nodes are present, the browser is already logged into Prism;
   use the bridge immediately.

5. If the bridge is absent, navigate directly to:

   ```text
   <origin>/login
   ```

   Tell the user Prism needs authentication and wait for them to finish the
   normal Prism login flow. Do not try to reuse Codex credentials, inspect local
   auth files, or invent a separate bootstrap flow.

6. After login completes, return to Prism and probe for the bridge again.

## Browser bridge helper

Use Browser Use with the `iab` backend and the normal Browser Use bootstrap for
the current environment.

Prefer the hidden DOM bridge over UI scraping:

```js
const prismCommandInput = tab.playwright.getByTestId('codex-browser-command');

async function runPrism(method, args = []) {
  const nonce = `${method}-${Date.now()}-${Math.random()}`;
  await prismCommandInput.fill(JSON.stringify({ method, args, nonce }));

  for (let i = 0; i < 160; i += 1) {
    await tab.playwright.waitForTimeout(100);
    const raw = await tab.playwright
      .getByTestId('codex-browser-result')
      .textContent();
    const parsed = JSON.parse(raw);
    if (parsed?.nonce === nonce) return parsed;
  }

  throw new Error(`Timed out waiting for Prism command result: ${method}`);
}
```

Read the cheap current snapshot when useful:

```js
JSON.parse(await tab.playwright.getByTestId('codex-browser-snapshot').textContent())
```

## Default workflow

### 1. Start from the current surface

For a new-project request, use `<origin>/new?login=true` and wait for Prism
to land on that project. Do not inspect the projects page or click New-project
controls.

On the projects page:

```js
await runPrism('snapshot');
await runPrism('listProjects');
await runPrism('openProject', ['<project-id>']);
```

On an open project page:

```js
await runPrism('snapshot');
await runPrism('listFiles');
await runPrism('stat', ['main.tex']);
```

`listProjects()` and `listFiles()` return metadata only.

### 2. Read narrowly

```js
await runPrism('readTextLines', ['main.tex', { startLine: 1, endLine: 40 }]);
await runPrism('readTextRange', ['main.tex', { offset: 200, length: 300 }]);
await runPrism('searchText', [{ query: '\\section', path: 'main.tex' }]);
```

Use `readFile(path)` only when the full file is needed.

### 3. Edit directly

```js
await runPrism('replaceTextRange', [
  'main.tex',
  { offset: 120, length: 0, content: '\\input{section.tex}\n' },
]);

await runPrism('createFile', [
  'section.tex',
  '\\section{Included Section}\n\nText from another file.\n',
]);
```

Use the bridge APIs instead of accessibility-tree editing when available.

### 4. Compile and verify

```js
const compile = await runPrism('compilePdf');
```

Treat `status === "success"` as “a PDF exists,” not “the compile is clean.”

Check:

- `hardErrorCount`
- `firstHardError`

If `hardErrorCount > 0`, fix the issue or tell the user the PDF exists but the
TeX run is not clean.

Search logs only when needed:

```js
await runPrism('searchPdfLogs', [{ query: '!', kind: 'pdfTex' }]);
await runPrism('readPdfLogLines', ['pdfTex', {
  startLine: 120,
  endLine: 150,
}]);
```

### 5. Download when needed

```js
await runPrism('downloadFile', ['main.tex']);
await runPrism('downloadPdf');
await runPrism('exportProjectZip');
```

## API inventory

Projects page:

- `snapshot()`
- `listProjects()`
- `openProject(projectId, mainDocument?)`

Project page metadata:

- `snapshot()`
- `listFiles()`
- `stat(path)`

Reads:

- `readFile(path)`
- `readTextRange(path, { offset, length })`
- `readTextLines(path, { startLine, endLine })`
- `searchText({ query, path?, caseSensitive?, maxResults? })`

Edits and file control:

- `replaceTextRange`
- `replaceTextLines`
- `writeFile`
- `createFile`
- `createDirectory`
- `renamePath`
- `movePath`
- `deletePath`
- `openFile`
- `downloadFile`
- `exportProjectZip`

PDF and logs:

- `compilePdf()`
- `getPdf({ includeContent?: true })`
- `searchPdfLogs({ query, kind?, caseSensitive?, maxResults? })`
- `readPdfLogLines(kind, { startLine, endLine })`
- `downloadPdf()`

## Output discipline

When reporting back:

- state which files changed,
- report the compact compile result,
- surface remaining compile errors,
- mention log searches only when you used them,
- avoid dumping full logs unless the user asks.
