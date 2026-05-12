# Orbit Explorer — Revision 2.0 Plans

## Current State

Orbit Explorer is an interactive web-based visualization tool for exploring parameterized sequence rules through force-directed graphs. It's a pedagogical tool for mathematical inquiry, developed by the Technology Educator Alliance.

The entire application lives in two files:

| File | Lines | Purpose |
|------|-------|---------|
| `index.html` | 2,755 | Production app (v2) — all UI, logic, and styling in one file |
| `index-experimental.html` | 2,056 | Older beta (v0.5) for reference |

There is no build system. React 18 and Babel are loaded from CDN, and all JSX is compiled in the browser at runtime.

---

## Architecture — What to Restructure

### The Monolith

Everything lives in a single `App()` function (lines 1007–2749 of `index.html`). That's ~1,740 lines of state, logic, and JSX in one component. A revised version should split this into dedicated components:

| Proposed Component | Responsibility |
|--------------------|----------------|
| **ParameterControls** | Sliders for m, b, d with color-coded labels |
| **GraphCanvas** | Force-directed visualization + pointer/drag handlers |
| **StepperControl** | Visible steps dial (up/down arrows + numeric input) |
| **DataTableOverlay** | Draggable/resizable cases panel with Recent Results and Saved Cases |
| **ChatPanel** | ZhengGPT sidebar (API key management, messages, transcript download) |
| **ScoreboardPanel** | Number grids, grid window controls, recent starts list |
| **OverflowModal** | Value-limit reached overlay |
| **TopBar** | App header with formula display, parameter controls, input, action buttons |

### Build System

Currently zero build tooling (CDN React + in-browser Babel). A Vite or similar setup would enable:

- Proper file splitting across components
- Dev server with hot module replacement
- Production builds with minification and tree-shaking
- CSS modules or a styling solution
- Import/export for shared utilities

### State Management

The `App` function currently has ~30+ `useState` calls plus ~10 `useRef` calls. Many could be consolidated:

- **`useReducer`** for related state (e.g., graph state, data table state, chat state)
- **React Context** for theme, language, and shared config
- Separate physics/canvas state from React render state (already partially done with refs)

---

## UI Areas to Revise

### 1. Top Bar (lines 1746–1871)

**Current:** Very dense — sliders, input, 6+ buttons all in one wrapping flex row. Gets cramped on smaller screens.

**Contains:**
- App title ("Orbit Explorer v2")
- Language selector dropdown
- Formula display: `f(n) = n/d (even) | mn + b (odd)` with color-coded parameters
- Three parameter sliders (m in green, b in orange, d in blue) with range inputs
- Start value text input with validation warning
- Add, Reset, Cases buttons
- Save Latest Case button with dynamic state (disabled/saved/active)
- Help (?) button with tooltip

**Issues:**
- All inline styles, no responsive breakpoints
- Button styles use hardcoded colors (`#1a2d50`, `#3d1020`) instead of theme tokens
- Input field has hardcoded dark-theme colors at line 1810 that don't respond to the light theme toggle

### 2. Stepper Widget (lines 2046–2187)

**Current:** Large circular dial positioned absolutely over the canvas (top-left), with up/down arrow buttons and a hover tooltip.

**Critical issue:** Uses an IIFE (Immediately Invoked Function Expression) to create local state inside the render:
```jsx
{(() => {
  const [arrowUpActive, setArrowUpActive] = React.useState(false);
  const [arrowDownActive, setArrowDownActive] = React.useState(false);
  const [stepperHovered, setStepperHovered] = React.useState(false);
  return ( ... );
})()}
```
This is a React anti-pattern — it creates state inside a render callback, which remounts on every parent re-render, losing local state. This should be extracted into its own `<StepperControl />` component.

**Contains:**
- Up arrow button (▲) with click sound and yellow flash on press
- 100px circular dial with editable numeric input (font size 36px)
- Down arrow button (▼) with click sound and yellow flash on press
- Hover tooltip explaining the control

### 3. Canvas + Overlays (lines 1910–2042)

**Current:** Full-flex canvas element with several absolutely-positioned control groups floating over it.

**Canvas element** (line 1914):
- Handles `onPointerDown`, `onPointerMove`, `onPointerUp`, `onPointerCancel`
- Cursor changes between `grab` and `grabbing` based on drag state

**Floating controls** (lines 1921–2042):
- Reset Layout button
- Help (?) button with drag explanation tooltip
- Grid checkbox toggle
- Light Theme checkbox toggle
- Show Cycles checkbox toggle
- Help (?) button with cycle layout explanation tooltip

**Issues:**
- All checkbox labels have identical styling repeated inline (~15 lines each)
- Floating controls overlap the stepper widget on narrow viewports

### 4. Data Table Overlay (lines 2190–2374)

**Current:** A `position: fixed` panel that is draggable (via pointer events) and resizable (via CSS `resize: both`). Contains two tables in a scrollable area.

**Contains:**
- Drag-to-move header bar with title, Download Cases button, Close button
- Explanatory text about Recent Results vs Saved Cases
- **Recent Results table** — auto-populated, columns: Start, Outcome, Steps, Peak, Keep (save button)
- **Saved Cases table** — user-curated, columns: m, b, d, Start, Outcome, Steps, Peak, Remove

**Supporting logic:**
- `clampDataTableRect()` (line 1083) — keeps panel within viewport bounds
- `startDataTableDrag()` (line 1102) — initiates pointer-based dragging
- `ResizeObserver` integration (line 1149) — syncs CSS resize with React state
- Custom pointer event listeners on `window` for drag tracking

**Issues:**
- ~70 lines of custom drag logic that could use a library or shared hook
- Table styles are all inline with no reuse between the two tables
- Min dimensions hardcoded (540px width, 320px height)

### 5. Chat Panel / ZhengGPT (lines 2376–2526)

**Current:** Fixed-width right sidebar with a drag-to-resize handle on the left edge.

**Contains:**
- Resize handle (12px wide, shows ⋮ icon) with pointer capture for smooth resizing
- Header with "ZhengGPT" title and Change Key button
- API key input section (password field, Save Key button, provider links)
- Provider detection display (OpenRouter vs DeepSeek)
- Message list with bubble styling (user messages right-aligned blue, assistant left-aligned)
- Loading indicator ("ZhengGPT is thinking...")
- Prominent orange Download Chat button (gradient background, hover animation)
- Chat input with Send button

**Supporting logic:**
- `detectChatProvider()` (line 81) — checks if key starts with `sk-or-v1-`
- `getChatRequestConfig()` (line 85) — builds fetch config for OpenRouter or DeepSeek
- `sendChat()` (line 1376) — sends message, handles response/errors
- `downloadChatTranscript()` (line 1281) — generates formatted .txt file
- `saveToGitHub()` (line 1266) — optionally pushes transcript to GitHub repo
- `buildSystemPrompt()` (line 444) — constructs the ZhengGPT system prompt with role instructions
- `clampChatPanelWidth()` (line 1095) — constrains resize between 260px and 720px

**Issues:**
- API key stored in localStorage as plaintext (line 1024)
- Chat panel width state and resize logic are separate from data table drag logic despite being similar patterns
- Hover animation uses direct DOM manipulation (`e.target.style.transform`) instead of React state or CSS

### 6. Scoreboard / Data Reference Grids (lines 2528–2746)

**Current:** 290px fixed-width right panel (ordered before chat panel via `order: 1`).

**Contains:**
- "DATA REFERENCE GRIDS" header with help tooltip
- **Grid Window controls:**
  - Read-only "Start at" display (aligned to GRID_BLOCK_SIZE = 100 boundaries)
  - Read-only "Blocks" count display
  - Previous/Next navigation buttons (move by `gridCount * 100` values)
- **Recent Starts** section — last 8 explored values as clickable buttons showing start value and outcome
- **Number grids** — `gridCount` blocks of 10x10 clickable cells (100 numbers each)
  - Each cell is a button that adds the number to the exploration on click
  - Visited numbers are highlighted with `valColor()` gradient
  - Unvisited numbers show dark background
- **Explanatory text** (red, large) about using grid controls
- **Color scale legend** — green → yellow → orange → red explanation
- **Glow explanation** — describes cycle node glow effect

**Issues:**
- Grid cells use `onMouseDown` with `e.preventDefault()` instead of `onClick` (line 2696)
- No virtualization — renders all grid cells even if many blocks are shown
- Fixed 290px width doesn't adapt to screen size

---

## Core Logic to Preserve

These functions contain the essential math and rendering logic and should be preserved carefully during any restructuring:

### Sequence Computation
- **`computeSeq(start, m, a, d)`** (line 574) — Core math: `f(n) = n/d` if even, `f(n) = mn + b` if odd. Runs up to `MAX_ITER` (600) steps, detects convergence, cycles, divergence, non-integer results, and timeout. Returns sequence array, outcome, cycle start index, and peak value.

### Physics Engine
- **`physicsStep(nodes, edges, W, H)`** (line 625) — Force-directed layout: O(n^2) repulsion between all node pairs (force = 5000/d^2), spring attraction along edges (natural length 85px, stiffness 0.05), gentle gravity toward center (0.018), Euler integration with 0.83 damping. Respects pinned/dragging state.

### Cycle Layout
- **`applyCycleLayout(nodes, edges, cycleNodes, W, H)`** (line 681) — BFS to detect separate cycle groups via edge traversal, arranges each group in a circle (radius scales with group size), places non-cycle nodes in a grid below.

### Canvas Rendering
- **`drawGraph(canvas, nodes, edges, ...)`** (line 760) — Full canvas render pipeline:
  - Background fill + optional grid lines (60px spacing)
  - Formula watermark with color-coded parameters (auto-sizes to fit)
  - Bidirectional cycle edge detection and curved arrow rendering (3 arrow pairs per bidirectional edge, fading alpha)
  - Regular one-way edges with arrowheads
  - Node rendering with: pop-in animation (exponential decay), breathing animation for start nodes, glow halos for cycle/start nodes, color by `valColor()`, label truncation for large numbers
  - Helper functions: `drawCenteredSegments()`, `drawFormulaLine()`, `getSegmentsWidth()`

### Graph Rebuilding
- **`rebuildGraphFromSequences()`** (line 1473) — Reconstructs the entire graph from stored sequences when `maxVisibleSteps` changes. Extends cycle sequences beyond their natural length to show repetitions. Preserves existing node positions. Deduplicates edges.

### Color System
- **`valColor(v)`** (line 606) — Maps value magnitude to color via log10 scale with 4 gradient stops: green (0) → yellow-green (0.3) → orange (0.6) → red (1.0). Threshold at log10(v)/3.5.

### Node Interaction
- **`findNodeAt(nodes, startNodes, x, y, now)`** (line 531) — Hit testing against all nodes (reverse iteration for z-order), accounts for pop-in animation size.
- **`getNodeRadius(position, startNodes, now)`** (line 524) — Dynamic radius: base 22px for start nodes, 15px for others, multiplied by pop-in scale (1 + 2.2 * e^(-t*2.8)).
- Pointer handlers (lines 1622–1685): `handleCanvasPointerDown`, `handleCanvasPointerMove`, `handleCanvasPointerUp`, `handleCanvasPointerCancel` — drag-to-pin with pointer capture.

### Localization
- **Translation system** (lines 127–382) — 3 languages (English, Spanish, Simplified Chinese) with ~60 translation keys each, including nested `outcomes` object. `translate()` function with dot-notation key access and `{variable}` interpolation via `interpolate()`.

### Theme System
- **`THEME_PRESETS`** (lines 384–429) — Dark and light presets with 27 color tokens each covering: page/panel/surface backgrounds, text hierarchy (primary/secondary/muted), borders, overlay backgrounds, and canvas-specific colors (background, grid, edges, watermark, annotation).

### Audio
- **`playClickSound()`** (line 1220) — Web Audio API: 800Hz sine wave, 50ms duration, 0.15 gain with exponential ramp. Used for stepper button clicks.

### Data Export
- **`downloadChatTranscript()`** (line 1281) — Generates formatted .txt with header, parameters, exploration summary, and full conversation. Optionally saves to GitHub via API.
- **`downloadCasesTable()`** (line 1335) — Generates CSV with saved cases and recent results sections.

---

## Specific Issues to Address

### 1. Inline Styles Everywhere
Every element has a `style={{...}}` object. This makes it hard to maintain consistent design, creates large JSX blocks, and prevents pseudo-class styling (`:hover`, `:focus`, media queries). Consider CSS modules, Tailwind, or styled-components.

### 2. Hardcoded Colors Outside Theme
Despite the theme system with 27 tokens, several UI elements use hardcoded colors that don't respond to theme changes:

- Start value input: `background: '#1a2d50'`, `border: '1px solid #3355aa'` (line 1810)
- Add button: `background: '#1e44aa'` (line 1815)
- Reset button: `background: '#3d1020'`, `color: '#ffaaaa'` (line 1819)
- Cases button: `background: '#1a2d50'` / `'#2a4455'` (line 1823)
- Chat input/buttons have similar issues
- Scoreboard grid cells: `'#182840'`, `'#1e3566'` (line 2684–2685)

### 3. No Responsive Design
The layout assumes a wide screen with two fixed sidebars:
- Scoreboard: 290px fixed
- Chat panel: 320px default (resizable 260–720px)
- Canvas: flex remainder

On screens under ~900px, the canvas gets unusably small. No media queries or breakpoints exist. Mobile is effectively unsupported.

### 4. The IIFE Component Anti-Pattern (line 2046)
The stepper widget creates `useState` hooks inside an IIFE within the render. This means:
- State is lost on every parent re-render
- It violates React's rules of hooks (hooks must be called at the top level)
- It works by accident because the IIFE always runs in the same order, but it's fragile

### 5. Duplicated Patterns
- Checkbox labels (Grid, Light Theme, Show Cycles) each repeat ~15 lines of identical styling
- Data table and chat panel both have custom drag/resize logic that could share a hook
- Two tables in the data overlay have similar column structures but duplicate all styling

### 6. GitHub Token in LocalStorage
`ghToken` (line 1030) is stored as plaintext in localStorage. This is a personal access token with repo write access (used in `saveToGitHub()`). Consider whether this feature needs a more secure approach.

### 7. Performance
- `physicsStep()` is O(n^2) — fine for MAX_NODES=120, but could bottleneck if the limit increases
- Number grids render all cells without virtualization (up to 20 blocks x 100 cells = 2,000 buttons)
- `rebuildGraphFromSequences()` clears and rebuilds the entire graph on every step change
- Animation loop runs `requestAnimationFrame` continuously even when nothing is changing

### 8. URL Parameter Handling
Parameters are read from URL on mount (line 1010–1017) but changes to m, b, d are never reflected back to the URL. Deep-linking to a specific parameter state only works one way.

---

## Constants Reference

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_ITER` | 600 | Max sequence steps before timeout |
| `MAX_VAL` | 1e12 | Divergence threshold |
| `MAX_START` | 1e9 | Largest UI-allowed starting value |
| `MAX_NODES` | 120 | Cap on displayed graph nodes |
| `NODE_R` | 15px | Base node radius |
| `START_R` | 22px | Start node radius |
| `GRID_BLOCK_SIZE` | 100 | Numbers per grid block |
| `CHAT_PANEL_MIN_WIDTH` | 260px | Chat panel minimum |
| `CHAT_PANEL_MAX_WIDTH` | 720px | Chat panel maximum |

---

## File Structure Proposal

```
orbit-explorer/
  index.html                  (entry point — slim shell)
  src/
    App.jsx                   (root layout, routing between panels)
    components/
      TopBar.jsx              (header with params, input, actions)
      ParameterSlider.jsx     (reusable m/b/d slider)
      GraphCanvas.jsx         (canvas + pointer handlers)
      StepperControl.jsx      (visible steps dial)
      DataTableOverlay.jsx    (draggable cases panel)
      ChatPanel.jsx           (ZhengGPT sidebar)
      ScoreboardPanel.jsx     (number grids + recent starts)
      OverflowModal.jsx       (value limit overlay)
      NumberGrid.jsx           (single 10x10 grid block)
    hooks/
      useDraggable.js         (shared drag logic for panels)
      useResizable.js         (shared resize logic)
      usePhysics.js           (animation loop + physics engine)
    lib/
      compute.js              (computeSeq, valColor)
      physics.js              (physicsStep, applyCycleLayout)
      draw.js                 (drawGraph + helpers)
      chat.js                 (API config, system prompt, providers)
      i18n.js                 (translations, translate function)
      themes.js               (THEME_PRESETS, PARAM_COLORS)
      audio.js                (playClickSound)
      export.js               (transcript + CSV generation)
    constants.js              (MAX_ITER, MAX_VAL, etc.)
  vite.config.js
  package.json
```
