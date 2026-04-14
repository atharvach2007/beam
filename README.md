# BeamAnalyser — Frontend & API Communication

## Overview

BeamAnalyser is a single-page web application. The entire user interface lives in one HTML file (`index.html`), styled with CSS (`style.css`), and driven by a single JavaScript file (`app.js`). When the user clicks *Analyse Beam*, the browser collects all input values, serialises them into a JSON object, sends that to a Python Flask server over HTTP, receives a JSON response, and then uses that data to render charts and update the display — all without reloading the page.

---

## The HTML Structure

`index.html` defines the skeleton of the page. It has three main areas:

**The navbar** sits at the top and is purely decorative — it shows the BeamAnalyser logo and a version badge.

**The sidebar** (`<aside class="sidebar">`) is where all user input lives. It is divided into labelled sections using `<div class="sidebar-section">` blocks:

- *Beam configuration* — beam type dropdown, length, segment count, optional support positions, and the self-weight toggle
- *Cross-section* — section type dropdown (rectangular, circular, I-section) with conditionally visible input fields for each
- *Material* — material preset dropdown plus E (GPa) and σ_y (MPa) fields
- *Loads* — a dynamic list of added loads and the *Add load* button

At the bottom of the sidebar is the *Analyse beam* button which triggers the whole pipeline.

**The main panel** (`<main class="main">`) holds everything that updates after analysis:

- The beam diagram SVG
- Four metric cards (safety factor, max bending stress, max deflection, max shear force)
- Four plot areas: SFD, BMD, deflection curve, and stress heatmap
- An error banner that appears if the API call fails

There is also a **modal overlay** (`<div id="loadModal">`) which is hidden by default and slides in when the user adds or edits a load. It contains input fields for all four load types (UDL, VDL, point load, applied moment) and shows the relevant fields depending on which type is selected.

The HTML itself contains no logic — it is entirely static markup. All behaviour is handled by JavaScript.

---

## JavaScript — app.js

`app.js` is the brain of the frontend. It does four distinct things: manages state, handles UI interactions, builds and draws the beam diagram SVG, and communicates with the Flask API.

### State management

At the top of the file two module-level variables hold all runtime state:

```js
const loads  = [];   // array of load objects added by the user
const charts = {};   // holds Chart.js chart instances keyed by name
```

Loads are plain JavaScript objects. Each one has a `type` field (`"udl"`, `"vdl"`, `"point"`, or `"moment"`) and type-specific fields such as magnitude, start/end positions, or x-coordinate. They live in this array for the lifetime of the page and are passed directly to the API on each analysis run.

Self-weight state is tracked separately in two variables: `swEnabled` (boolean toggled by the checkbox) and `swMode` (either `"auto"` for density-based or `"manual"` for user-entered mass).

### DOM shortcut

A single utility function is defined at the top:

```js
const $ = id => document.getElementById(id);
```

This is used throughout the file in place of `document.getElementById()` to keep the code concise. Every interactive element in the HTML has a unique `id` and is accessed this way.

### UI wiring — event listeners

Every interactive element in the sidebar has a JavaScript event listener attached directly when the script loads. These fall into a few categories:

**Dropdown change listeners** watch `beamType`, `sectionType`, and `material`. When `beamType` changes, it shows or hides the support positions input field (only needed for overhanging and continuous beams) and calls `drawBeamDiagram()` to immediately update the visual. When `sectionType` changes, it shows the correct set of dimension inputs for the selected shape and hides the others. When `material` changes, it populates the E and σ_y fields with preset values and disables them to prevent accidental editing (or enables them for custom input).

**Input listeners** on `beamLength` and `supportsInput` immediately call `drawBeamDiagram()` on every keystroke so the visual always reflects the current configuration before any analysis is run.

**The self-weight toggle** (`selfWeightToggle`) shows or hides the self-weight sub-panel and recalculates the live weight display. Any change to density, mass, section dimensions, or beam length also updates this display and redraws the diagram (which shows or hides the weight arrow below the beam).

**The load modal** is opened by `addLoadBtn` and closed by `cancelLoad`. The `newLoadType` dropdown inside it controls which input fields are visible inside the modal. When `confirmLoad` is clicked, the input values are read, a load object is built, and it is either pushed into the `loads` array (new load) or used to overwrite an existing entry (edit mode). After that, `renderLoadList()` and `drawBeamDiagram()` are both called to keep everything in sync.

### The load list

`renderLoadList()` rebuilds the load list in the sidebar from scratch every time a load is added, edited, or deleted. It iterates over the `loads` array, creates a `<div class="load-row">` for each entry, assigns a colour from the `LOAD_PALETTE` array (8 distinct colours that cycle if there are more than 8 loads), and attaches edit and delete buttons. Each load's colour matches exactly what is drawn on the beam diagram SVG.

---

## The API Call — Frontend to Flask

When the user clicks *Analyse beam*, the `runAnalysis()` function runs. This function is `async`, meaning it can use `await` to pause and wait for the HTTP response without freezing the browser.

### Step 1 — Collecting inputs

The function reads every input from the sidebar and assembles two configuration objects:

**`secCfg`** describes the cross-section:
```js
{ section_type: "rectangular", width: 50, height: 100 }
// or: { section_type: "circular", diameter: 100 }
// or: { section_type: "i_section", width: 100, height: 200,
//        flange_thick: 10, web_thick: 6 }
```

**`matCfg`** describes the material:
```js
{ material: "structural_steel", E: 200, sigma_y: 250 }
```

The `loads` array is copied into `allLoads`. If self-weight is enabled, a synthetic point load is appended at that point:
```js
{ type: "point", magnitude: W_kN, x: L / 2, _isSelfWeight: true }
```
where `W_kN` is computed in JavaScript from either `ρ × A × L × g` (auto mode) or `mass × g` (manual mode). The `_isSelfWeight` flag is ignored by the backend — it is just for internal bookkeeping.

### Step 2 — Sending the request

All of this is assembled into one JSON object and sent to the backend:

```js
fetch('/api/analyse', {
  method:  'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    beam_type:  bt,          // e.g. "simply_supported"
    length:     L,           // metres
    n_segments: n,           // default 1000
    section:    secCfg,
    material:   matCfg,
    loads:      allLoads,    // array of load objects (kN / kN·m units)
    supports:   supports,    // array of x-positions, only for overhanging/continuous
  })
});
```

This is a standard HTTP POST request. The `fetch()` API is built into all modern browsers and needs no external library. The URL `/api/analyse` is a relative path — in development it resolves to `http://localhost:5000/api/analyse` (Flask's default port), and in production it resolves to the deployed Vercel URL. The browser does not need to know which one.

While the request is in flight, the *Analyse beam* button is disabled and its label changes to `"Analysing…"` to give the user feedback.

### Step 3 — Receiving the response

The response arrives as JSON. Before parsing it, the code checks the `Content-Type` header to confirm it is actually JSON — if the server crashes and returns an HTML error page instead, this prevents a confusing parse error. If the HTTP status code is not `2xx`, the error message from the JSON body is thrown and displayed in the red error banner.

A successful response looks like this:

```json
{
  "x": [0.0, 0.005, 0.010, ...],         // position array (metres), 1001 values
  "V": [12500.0, 12495.0, ...],           // shear force (Newtons)
  "M": [0.0, 62.5, 125.0, ...],           // bending moment (Newton-metres)
  "y": [0.0, -0.001, -0.003, ...],        // deflection (millimetres)
  "reactions": {
    "R_A": 12500.0,
    "R_B": 12500.0
  },
  "results": {
    "max_shear_kN":           12.5,
    "max_moment_kNm":         31.25,
    "max_deflection_mm":      4.883,
    "max_bending_stress_MPa": 187.5,
    "safety_factor":          1.33
  }
}
```

### Step 4 — Updating the UI

The response data flows into three separate rendering functions:

`updateMetrics(data.results)` writes the four summary numbers into the metric cards and applies the appropriate colour class (`safe`, `warn`, or `danger`) to the safety factor card based on threshold values (≥2.5 is green, ≥1.5 is amber, <1.5 is red).

`drawCharts(data)` passes the arrays to Chart.js.

`drawBeamDiagram(data)` passes the reactions object so reaction arrows can be added to the SVG.

---

## Chart.js — The Three Line Graphs

Chart.js is loaded from a CDN via a `<script>` tag in `index.html`. It is a JavaScript charting library that renders onto HTML `<canvas>` elements.

Three charts are created once on page load by `initCharts()`:

```js
charts.sfd = new Chart($('chartSFD'), makeChartCfg(C.blue,   'SFD',        'Shear force  V (kN)'));
charts.bmd = new Chart($('chartBMD'), makeChartCfg(C.orange, 'BMD',        'Moment  M (kN·m)'));
charts.def = new Chart($('chartDef'), makeChartCfg(C.green,  'Deflection', 'Deflection  δ (mm)'));
```

Each chart is a `line` type. `makeChartCfg()` is a factory function that returns a Chart.js configuration object. It sets the theme colours to match the dark CSS palette, configures both axes with their unit labels, enables the zero-line highlight (the horizontal grid line at y=0 is drawn brighter than the rest), and sets a custom tick callback that always forces the first and last x-axis labels to render while auto-skipping interior ones to prevent crowding.

When analysis data arrives, `drawCharts()` runs. It first passes the raw arrays through `downsample()`, which reduces them to at most 300 points for rendering efficiency (1000-point arrays would work but are unnecessary for a smooth line). The downsampler explicitly keeps the first and last elements so the x-axis always starts at 0 and ends exactly at the beam length. The units are converted here — V is divided by 1000 to go from Newtons to kN, M is divided by 1000 to go from Newton-metres to kN·m. The x positions become string labels formatted to 3 decimal places. Each chart is then updated by writing new `labels` and `data` arrays directly onto the chart object and calling `chart.update('none')` (the `'none'` argument skips the animation).

---

## SVG — The Beam Diagram

The beam diagram is a raw SVG element (`<svg id="beamDiagram">`) embedded directly in the HTML. It has no external library — every element inside it is generated as an HTML string by JavaScript and written to `svg.innerHTML`. This approach means the diagram redraws instantly on every input change without needing to diff a virtual DOM.

The diagram coordinate system is defined by the `DIAG` constants object:

```js
const DIAG = {
  W: 800, H: 320,   // viewBox dimensions in SVG units
  PAD: 50,          // horizontal padding before beam starts
  BY: 200,          // y-coordinate of beam centreline
  BH: 13,           // beam height in pixels
  LOAD_TOP: 14,     // y-coordinate where load arrows start
  ARR_SPACING: 38,  // pixels between UDL arrows
  ARR_MIN: 3,       // minimum arrows for any UDL span
};
```

The beam runs from `x = PAD` to `x = W - PAD`. A helper function `bx(xm)` converts a physical position in metres to an SVG x-coordinate:
```js
const bx = xm => bx0 + Math.max(0, Math.min(1, xm / L)) * bw;
```

`drawBeamDiagram()` runs on every input change (before any analysis) and also after analysis (when it has reaction data to draw). It builds the entire SVG content as one long HTML string and writes it all at once. The content is assembled in this order:

1. **`<defs>`** — SVG definitions block containing a `<clipPath>` that clips everything to the viewBox bounds, a `<linearGradient>` for the beam body shading, and a glow `<filter>` used on point load arrows
2. **Beam body** — a single `<rect>` with the beam gradient fill
3. **Supports** — pin triangles (`pinSupport()`), roller circles (`rollerSupport()`), or hatched wall rectangles (`fixedWall()`) depending on beam type
4. **Loads** — one rendering function per load type, each receiving a palette colour and a unique marker ID so SVG arrowheads do not bleed between loads
5. **Self-weight arrow** — if enabled, a short downward yellow arrow below the beam midpoint with a weight label
6. **Reaction arrows** — only drawn when analysis data is present; yellow upward arrows with R= labels
7. **x-axis labels** — three text elements at x=0, x=L/2, and x=L outside the clip group

Each load type has its own render function. `renderUDL()` draws a dashed top rail and evenly spaced downward arrows. `renderVDL()` draws a filled trapezoid polygon with a sloped top line and arrows of varying height. `renderPoint()` draws a single arrow with the label pill placed above it and the arrow shaft starting from the bottom of the pill so the line never passes through the label. `renderMoment()` draws a curved arc arrow using an SVG `<path>` with an arc command. All of these functions accept a `labelRow` index that offsets the label vertically by `labelRow × 18px` — this staggers labels when multiple loads share the same x-position so they never overlap.

---

## The Flask Route — How Python Receives the Data

On the Python side, `api/index.py` defines one route that handles everything:

```python
@app.route("/api/analyse", methods=["POST"])
def analyse():
    data = request.get_json(force=True, silent=True)
```

Flask's `request.get_json()` deserialises the JSON body that the browser sent. The `force=True` argument makes Flask parse the body as JSON regardless of the Content-Type header (a safety measure), and `silent=True` returns `None` instead of raising an exception if parsing fails.

From that point the Python code reads the fields by key:

```python
beam_type    = data.get("beam_type", "simply_supported")
length       = float(data.get("length", 5))
n_segments   = int(data.get("n_segments", 1000))
section_cfg  = data.get("section", {})
material_cfg = data.get("material", {})
loads_raw    = data.get("loads", [])
supports     = data.get("supports", [])
```

After validation and solving, the results are returned as JSON using Flask's `jsonify()`, which serialises the Python dict to JSON and sets the `Content-Type: application/json` header automatically. This is exactly what the browser checks for before calling `.json()` on the response.

If anything goes wrong, the entire function is wrapped in a `try/except` that always returns a JSON error object — never an HTML page — so the frontend always gets something it can parse and display in the error banner.

---

## The Complete Data Flow

Putting it all together, the full round-trip from user action to rendered result is:

1. User fills in inputs in the sidebar and adds loads via the modal
2. User clicks *Analyse beam*
3. JavaScript reads all DOM input values and assembles a JSON payload
4. `fetch()` sends `POST /api/analyse` with the payload
5. Flask receives the request, deserialises the JSON, runs the physics solvers
6. Flask serialises the results back to JSON and responds with HTTP 200
7. JavaScript receives the response, checks for errors, parses the JSON
8. `updateMetrics()` writes the four summary numbers into the metric cards
9. `drawCharts()` feeds the x, V, M, y arrays into the three Chart.js instances
10. `drawBeamDiagram()` regenerates the SVG with reaction arrows added

The entire round-trip — from button click to fully rendered results — typically completes in under 200 milliseconds for a 1000-segment analysis on a local machine.
