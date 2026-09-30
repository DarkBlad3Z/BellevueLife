# Eastside Eats Command

**Jimmy's Taste of Washington** — an interactive, sci-fi-styled restaurant guide to Bellevue, WA and the surrounding Eastside. 25 scouted spots on a hand-drawn tactical map, filterable by price, cuisine and sector, with an original synthesized soundtrack.

Zero dependencies. Zero build step. One HTML file.

> **Live:** `https://YOUR-USERNAME.github.io/eastside-eats-command/`
> *(replace `YOUR-USERNAME` after you enable Pages — see [Deploying](#deploying-to-github-pages))*

---

## What it does

- **Tactical minimap** — a hand-built SVG of the Eastside: Lake Washington, Mercer Island, Lake Sammamish, Puget Sound, plus I-405, I-90, SR-520, I-5, SR-167 and the Bel-Red corridor. Drag to pan, scroll to zoom, or jump between six preset views.
- **Smart pin clustering** — markers that would overlap collapse into a single badge showing the count. Click it to zoom into that group.
- **Four-axis filtering** — food vs. drinks, three price tiers, 11 cuisine categories and 7 geographic sectors, all cross-filtering the map and the roster together. Plus free-text search across names, dishes, tags and addresses.
- **Target dossiers** — click any pin or card for the address, grid coordinates, cost and drive-time bars, a recommended order list, attribute tags, field notes, and one-click links out to Maps and menus.
- **Original score** — a procedurally scheduled battle march (132 BPM, D minor, layered kick / snare / taiko / sub bass / brass stabs / lead) built entirely from Web Audio oscillators, plus hover blips, target-acquired chirps and zoom sweeps. **Muted by default**; the visitor opts in on the intro screen.

Everything runs client-side. No API keys, no map tiles, no fonts, no analytics — the page loads zero external resources. The only outbound URLs in the whole file are the two "Open in Maps" / "Menu & Hours" links in each dossier, and those only fire when a visitor clicks them.

---

## Quick start

Clone it and open the file:

```bash
git clone https://github.com/YOUR-USERNAME/eastside-eats-command.git
cd eastside-eats-command
open index.html          # macOS   (use `xdg-open` on Linux, `start` on Windows)
```

Or serve it locally if you prefer a real origin (needed if you want `localStorage` to persist reliably):

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

There is nothing to install and nothing to compile.

---

## Deploying to GitHub Pages

This repo is set up for the simplest possible deploy — no Actions workflow needed.

1. Push the repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Source**, choose **Deploy from a branch**.
4. Set the branch to **`main`** and the folder to **`/ (root)`**. Save.
5. Wait ~60 seconds. Your site is at `https://YOUR-USERNAME.github.io/eastside-eats-command/`.

The `.nojekyll` file is already included so GitHub serves the files as-is instead of running them through Jekyll.

---

## Project structure

```
eastside-eats-command/
├── index.html              # The entire application — HTML, CSS and JS inlined
├── data/
│   └── restaurants.json    # The dataset as portable JSON (see note below)
├── .nojekyll               # Tells GitHub Pages to skip Jekyll processing
├── .gitignore
├── LICENSE                 # MIT
└── README.md
```

`index.html` is fully self-contained and is the **source of truth** — the restaurant data lives in the `R` array inside its `<script>` block. `data/restaurants.json` is a machine-readable export of that same data, generated from the HTML, for anyone who wants the dataset without parsing the page. If you edit one, regenerate or update the other.

---

## The dataset

25 records across 7 sectors. Each entry looks like this:

```json
{
  "id": "santouka",
  "name": "Hokkaido Ramen Santouka",
  "category": "NOOD",
  "categoryName": "Noodle Houses",
  "sector": "DT",
  "sectorName": "Bellevue Downtown",
  "driveFromDowntownBellevue": "0–5 min",
  "priceTier": 2,
  "price": "$$",
  "priceRange": "$15-30",
  "primarilyDrinks": false,
  "address": "103 Bellevue Way NE, Suite 3, Bellevue",
  "lat": 47.61128,
  "lon": -122.20162,
  "description": "…",
  "recommendedOrders": ["Shio ramen", "Toroniku (pork cheek) add-on", "…"],
  "tags": ["Japanese", "Ramen", "Solo", "Rainy Day"],
  "notes": "…"
}
```

| Breakdown | |
|---|---|
| **By sector** | Bellevue Downtown 14 · Factoria 3 · Kent 3 · Bel-Red 2 · Kirkland 1 · Redmond 1 · Renton 1 |
| **By price** | `$` under $15 — 10 · `$$` $15–30 — 10 · `$$$` $30–60 — 5 |
| **By category** | Grill & BBQ 4 · Dumplings & Buns 3 · Street Eats 3 · Boba & Tea 3 · Noodle Houses 3 · Sushi & Poke 2 · Bakery & Sweets 2 · Coffee 2 · Middle Eastern 1 · Indian 1 · Buffet & AYCE 1 |

Coordinates were resolved through the U.S. Census Bureau's TIGER geocoder, so they are address-level accurate (correct building or block) rather than rooftop-exact.

---

## How the map works

There are no map tiles and no mapping library. Coastlines, lakes and highways are stored as arrays of `[lon, lat]` waypoints and projected into a fixed SVG coordinate space:

```js
const B  = { lonMin:-122.42, lonMax:-122.00, latMin:47.40, latMax:47.74 };
const PX = lon => (lon - B.lonMin) / (B.lonMax - B.lonMin) * 1000;
const PY = lat => (B.latMax - lat) / (B.latMax - B.latMin) * 1198;
```

The `1000 × 1198` base canvas is sized so one horizontal unit and one vertical unit cover the same real-world distance at this latitude (≈ 31.7 units per km), which keeps the geography from looking stretched.

Zooming animates the SVG `viewBox` rather than applying a CSS transform, so the vector terrain stays razor-sharp at any zoom. Two details make that work cleanly:

- **Strokes** use `vector-effect="non-scaling-stroke"`, so roads and shorelines stay a constant pixel width instead of ballooning as you zoom in.
- **Map text and the SVG chrome** (sector rings, the command-star crosshair) are counter-scaled every frame. An element of user-unit size `s` draws at `s × (containerWidth / viewBox.width)` pixels, so setting `s = base × K × (viewBox.width / 1000)` cancels the zoom out entirely and the label holds a fixed on-screen size. Labels also fade in and out by zoom tier — street names only appear past 3.2×, city names dim past 4.5×.

Restaurant pins are handled differently again: they are HTML elements in an overlay layer rather than SVG children, repositioned each frame from the current `viewBox`. That keeps them crisp, makes hover and click handling trivial, and allows CSS animation on the selection ring. Pins closer than 34px on screen collapse into a cluster badge via a greedy single-pass grouping.

---

## The audio

Every sound is generated at runtime by the Web Audio API — there are no audio files in this repo and nothing is streamed. A lookahead scheduler walks a 128-step (8-bar) pattern at 132 BPM. The harmony is D natural minor, built from the `i`, `VI`, `III` and `VII` chords in the bar order **i · i · VI · VII · i · III · VI · VII**. Voices:

- **Kick** — sine with a 150 → 42 Hz pitch envelope plus a triangle click transient
- **Snare** — bandpassed noise burst layered with a pitched triangle body
- **Hats** — highpassed noise on every even sixteenth, opening up on the last hit of each bar
- **Taiko** — low sine hit on the downbeat of bars 1 and 5
- **Bass** — saw plus a sub-octave square through a resonant lowpass, octave jumps on accents
- **Stabs & pad** — three-voice detuned saw chords with contrasting filter envelopes
- **Lead** — saw/square pair with a filter sweep, entering at bar 5

SFX are synthesized the same way. Audio starts muted and only resumes the `AudioContext` after an explicit user gesture, so it complies with browser autoplay policy and never ambushes a visitor.

---

## Adding or editing a restaurant

Open `index.html`, find the `const R = [` array (line 484), and add an object:

```js
{id:"unique-slug", n:"Restaurant Name", cat:"DUMP", sec:"DT", price:2, drink:false,
 lat:47.61021, lon:-122.20274, addr:"Street address, City",
 d:"One or two sentences on what makes it worth going.",
 orders:["Signature dish","Another dish"],
 tags:["Cuisine","Vibe","Occasion"],
 intel:"Anything worth knowing: closed Tuesdays, reserve ahead, easy to confuse with…"},
```

- `cat` must be one of the keys in `CAT` — `DUMP`, `NOOD`, `SUSHI`, `GRILL`, `BOBA`, `COFF`, `BAKE`, `ST`, `ME`, `IND`, `BUF`. Add a new key there (with a hex `color`) to create a category.
- `sec` must be one of the keys in `SECTOR` — `DT`, `FAC`, `BR`, `KIRK`, `RED`, `REN`, `KENT`. New sectors need a `name` and a `drive` estimate.
- `price` is `1`, `2` or `3`.
- `lat` / `lon` in decimal degrees. Anything roughly inside `47.40–47.74 N, 122.00–122.42 W` lands on the map; outside that it clips. Free geocoders: the [Census geocoder](https://geocoding.geo.census.gov/geocoder/) or Nominatim.

Filter chips, counts, legend, pins and clustering all rebuild from the array automatically. Nothing else to touch.

---

## Data accuracy

Every entry was verified against restaurant sites, Yelp, delivery platforms and local press in September 2026. Coordinates come from the U.S. Census Bureau geocoder rather than being eyeballed. Four corrections are worth flagging, because they are easy to get wrong:

1. **CHICHA San Chen** is a Taiwanese tea house in Bellevue Square — not a Colombian restaurant. No business named "Chibcha" exists in Bellevue.
2. **Mama Xita BBQ** is Chinese-Korean wagyu barbecue, despite the name. It serves no Mexican food.
3. **Din Tai Fung** moved out of Bellevue Square into Lincoln Square North in April 2024.
4. **Pho Bac Cafe** in Kent has no established connection to the well-known Phở Bắc chain in Seattle's Chinatown-ID. Same name, different business — no evidence of a link was found either way.

Also noted in the data: "Champion Gourmet" (Kent) is frequently mis-transcribed as "Champino"; Southern Spice brands itself as Bellevue but sits at a Redmond address on the Bel-Red line; and Happy Noodles is a rebrand of the former "Noodle House" at the same address and phone number.

One geographic quirk worth knowing: all three Kent entries — Champion Gourmet, Pho Bac Cafe and Happy Noodles — sit within about 200 metres of each other on East Valley Highway, with the latter two directly across the street. They render as a single cluster badge until you zoom in.

Hours, prices and menus change. **Verify before you drive.**

---

## Browser support

Any current version of Chrome, Edge, Firefox or Safari. Uses Pointer Events, CSS `clip-path`, `AudioContext`, `ResizeObserver` and `localStorage` — all baseline for several years now.

The last three degrade gracefully rather than being hard requirements: `ResizeObserver` falls back to a `window.resize` listener, `localStorage` reads and writes are wrapped in `try/catch` (you just lose remembered filters), and if `AudioContext` is missing *or* its constructor throws — enterprise policy, a privacy extension, a restrictive embed — the audio graph is marked dead, the toggle reports "Audio unavailable in this browser," and the rest of the page carries on untouched.

---

## License

[MIT](LICENSE) — do what you like with the code.

The restaurant write-ups are original editorial content. The visual design is an original sci-fi military HUD treatment and is not affiliated with, endorsed by, or derived from any game or film franchise.
