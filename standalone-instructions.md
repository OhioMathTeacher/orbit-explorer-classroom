# Standalone Orbit Explorer — Build Instructions

Goal: produce an **offline, double-clickable** version of Orbit Explorer that
Savannah (or any student) can run on their own computer with no internet
connection and no install step.

These instructions are written for Claude Code running inside VS Code on a
machine with normal internet access (the cloud sandbox session was blocked
from reaching `unpkg.com`, so the build couldn't be done there).

---

## Recommended approach: single self-contained HTML file

The current app (`index.html`) is a single-page React app that pulls three
libraries from `unpkg.com` at runtime:

- `react@18` (UMD development build)
- `react-dom@18` (UMD development build)
- `@babel/standalone` (in-browser JSX transpiler)

To make it work offline, inline those three libraries directly into the HTML.
The result is one `.html` file the student can double-click — opens in any
browser, fully cross-platform, no install, emailable as an attachment.

### Steps

1. Create a `standalone/` folder at the repo root.

2. Download the **production minified** versions of the three libraries into
   `standalone/vendor/`:

   ```bash
   mkdir -p standalone/vendor
   cd standalone/vendor
   curl -sSL -o react.production.min.js \
     https://unpkg.com/react@18/umd/react.production.min.js
   curl -sSL -o react-dom.production.min.js \
     https://unpkg.com/react-dom@18/umd/react-dom.production.min.js
   curl -sSL -o babel.min.js \
     https://unpkg.com/@babel/standalone/babel.min.js
   ```

   Sanity check: each file should be substantial (React ~6KB gzipped /
   ~25KB raw, ReactDOM ~130KB raw, Babel ~3MB raw). If any file is tiny or
   contains an error message, the download failed.

3. Write a build script `standalone/build.js` (Node, no dependencies) that:
   - Reads `../index.html`
   - Reads the three vendor files
   - Replaces the three `<script src="https://unpkg.com/...">` tags with
     inline `<script>...</script>` blocks containing the file contents
   - Writes the result to `standalone/orbit-explorer-offline.html`

   Keep it simple — a few `fs.readFileSync` calls and string replacements.
   No bundler, no npm install.

4. Run `node standalone/build.js`. Verify `orbit-explorer-offline.html`
   exists and is roughly 3–4 MB.

5. **Test it offline.** This is the important step:
   - Disconnect from the internet (or use airplane mode)
   - Double-click `orbit-explorer-offline.html`
   - Confirm the app loads, the sliders work, you can type a starting
     value, and the orbit visualization renders
   - Confirm the data reference grids on the right populate
   - Test on at least one other browser if possible

6. If any feature breaks offline, check the browser console. Likely
   culprits: a `fetch(...)` call to GitHub for "Orbit Data" files
   (around `index.html:1444`), or a stray CDN reference. Wrap network
   calls in try/catch so they fail silently rather than breaking the UI.
   Don't remove the features — just make them degrade gracefully when
   offline.

7. Commit `standalone/build.js`, `standalone/vendor/*`, and
   `standalone/orbit-explorer-offline.html` to `main`. Add a short
   `standalone/README.md` explaining "double-click the .html file to run
   offline" for end users.

### Optional polish

- Add a tiny banner or footer note in the offline build like "Offline
  edition — v2.x" so it's distinguishable from the hosted version.
- Update the top-level `README.md` to mention the offline build exists.

---

## Why not Electron / Tauri / Java?

We considered wrapping this as a "real" desktop app:

- **Electron**: ~150 MB per platform, requires platform-specific builds
  (Windows `.exe`, macOS `.app`, Linux `.AppImage`), code signing for a
  smooth install on macOS/Windows. Heavy for what we need.
- **Tauri**: Much smaller (~10 MB) but requires the Rust toolchain and
  per-platform builds.
- **Java**: Would require rewriting the whole app. The current code is
  React/JS — no reason to port it.

A single offline HTML file is dramatically less friction:
- One file, ~3 MB
- Truly cross-platform (anything with a browser)
- No install, no permissions prompts, no Gatekeeper warnings
- Emailable
- Trivial to update (rebuild + resend)

If we later want a "real app icon" experience, Electron or Tauri can wrap
the same offline HTML — but start here.

---

## Delivery to Savannah

Once tested, the simplest delivery is to email `orbit-explorer-offline.html`
as an attachment. She saves it anywhere (Desktop, Downloads) and
double-clicks. Done.

If her school's email blocks `.html` attachments, zip it first or rename
to `.html.txt` with instructions to rename back.
