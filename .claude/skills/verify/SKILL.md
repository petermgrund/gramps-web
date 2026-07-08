---
name: verify
description: Verify UI changes in this repo at runtime by mounting components on a harness page and driving them with Playwright.
---

# Verifying gramps-web frontend changes

The app is auth-gated and needs a Gramps Web API backend, so the full app
usually can't be driven without one. Instead, mount the changed components
directly on a harness page served by the real dev server.

## Recipe

1. `npm ci` if `node_modules` is missing.
2. Write a harness HTML page at the **repo root** (e.g. `verify-x.html`,
   delete before committing) with a `<script type="module">` that imports the
   component files from `./src/...` and mounts them with a stub appState:

   ```js
   const appState = {
     i18n: {strings: {}, lang: 'en'},
     path: {page: '', pageId: ''},
     permissions: {canAdd: true, canEdit: true},
     dbInfo: {},
     apiGet: async () => ({data: {default: {}, custom: {}}}),
     apiPost: async () => ({data: []}),
   }
   ```

   Expose mounted elements on `window.__harness` for assertions.
3. Serve with:
   `npx web-dev-server --app-index verify-x.html --host 127.0.0.1 --node-resolve --port 8001`
   (run in background; root dir is the repo, so `./src` imports resolve).
4. Drive with Playwright from a scratch dir (`npm i playwright`, then launch
   with `executablePath: '/opt/pw-browsers/chromium'` if the browsers dir is
   pre-provisioned). Playwright CSS selectors pierce the open shadow roots,
   e.g. `grampsjs-form-string#lat input` reaches the mwc-textfield input.

## Gotchas

- Components extending `GrampsjsObjectForm` render a **modal** `md-dialog`
  whose scrim intercepts clicks on everything else — mount dialogs lazily
  (via a `window.__mountDialog()` hook) after driving non-modal components.
- For real clipboard paste: `context.grantPermissions(['clipboard-read',
  'clipboard-write'], {origin})`, `navigator.clipboard.writeText(...)` in the
  page, then `locator.press('ControlOrCmd+v')`.
- Stub-related console errors to ignore: `data.data.filter is not a function`
  (tags mixin expects arrays), `getCurrentTheme is not a function` (map).
- Map tiles don't load offline; assert on DOM (e.g. `grampsjs-map-marker`
  presence) instead of pixels.
