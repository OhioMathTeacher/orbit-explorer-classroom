# Orbit Explorer — Standalone (Offline) Build

This folder is **self-contained**. Copy it to a thumb drive (or your hard drive),
double-click `index.html`, and Orbit Explorer opens in your default browser. No
installation, no internet required for the app itself.

## What's in here

```
standalone/
├── index.html              ← double-click this
├── vendor/
│   ├── react.production.min.js
│   ├── react-dom.production.min.js
│   └── babel.min.js
└── README.md
```

Total size: ~3.2 MB.

## What works offline

- The full Orbit Explorer canvas, sequence generation, graph drawing, data
  reference grids, save-case workflow, and language switching.
- The AI Setup menu — you can browse the cards and configure things.

## What needs the internet (or a local LLM server)

- **Cloud AI providers** (Claude / Gemini / Groq) require an internet connection
  to reach their servers. Bring your own API key.
- **Local AI providers** (your own LLM running on your machine) require the LLM
  server to be running. The AI Setup menu lets you add a local endpoint URL
  (e.g. `http://127.0.0.1:8765` if you're running the AthenaV6 shim).

If you just want to explore sequences without AI, pick **No AI** in the AI Setup
menu and the chat panel stays out of your way.

## How to launch

- **Windows:** double-click `index.html`. (If Windows asks which app to open it
  with, choose your browser — Chrome, Edge, Firefox, etc.)
- **macOS:** double-click `index.html`, or right-click → Open With → your
  browser.
- **Linux:** `xdg-open index.html` from the terminal, or double-click in your
  file manager.

## Where this came from

Built from the canonical Orbit Explorer at
<https://github.com/OhioMathTeacher/orbit-explorer> and
<https://github.com/OhioMathTeacher/orbit-explorer-classroom>. The standalone
version is the same code, bundled with its three runtime dependencies
(React, ReactDOM, Babel-standalone) so it runs from any directory without a
network connection.

## License

MIT (same as the parent project).
