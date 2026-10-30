# Reference Website Analysis — Zoom Earth (`https://zoom.earth/`)

## 1. Executive Summary & REA Verification Ledger

Before writing any application code or specifications, the installed **Reverse Engineer Anything (REA)** skill (`reverse-engineer-anything`, v42, 207 tools) and its connected MCP server (`rea`, provider `rea-playwright-browser-scenario` v1 driving headless Chromium `139.0.7258.155`) were activated and executed directly against the public reference website `https://zoom.earth/`.

No private source code, credentials, or restricted APIs were extracted. All observations derive from controlled browser scenario captures, DOM/accessibility snapshots, viewport screenshots, storage key hashes, and passive network request metadata recorded as cryptographic REA Evidence records.

### 1.1 REA Evidence Ledger

| Evidence ID | Scenario Scope | Viewport | Actions Executed | Captured Artifacts & Key Findings |
| :--- | :--- | :--- | :--- | :--- |
| `ev_1034998339b9a357cd3849824167d4fb914a07a55e3969c4f72f8bb2a85e0a5c` | Initial Desktop Load & Baseline DOM/AX Inspection | `1280×720` (`dark`, `en-US`, `UTC`) | `goto_start` (`https://zoom.earth/`), `wait_for_timeout` (5000 ms) | Full DOM (`86,209` chars), Accessibility tree (`4,691` bytes, SHA-256 `e4054864...`), initial PNG screenshot (`1,290,483` bytes, SHA-256 `a89d23b3...`), `sessionStorage` keys (`ze_layersMenu`, `ze_measure`), welcome modal state, and inline `window._ZE` configuration metadata. |
| `ev_217069da414db3eca976e6e1b975a5242912bda1786d825f0b752e7d9cc40437` | Desktop Layer Switching, Legends, Models & Overlays Panel | `1280×720` (`dark`, `en-US`, `UTC`) | 10 steps: `wait_load`, `close_welcome`, `view_satellite_ui`, `select_precipitation`, `view_precipitation_ui`, `select_temperature`, `view_temperature_ui`, `open_more_overlays`, `view_overlays_panel` | Per-step screenshots, `1,263` network events, verification of left `LIVE MAPS` / `FORECAST MAPS` sidebar, contextual `layer-intro` dialogs, dynamic bottom-left color legends (`Rain/Snow` and `°C`), bottom-right `ICON 13 km / GFS 22 km` model toggle, dynamic attribution updates, and contextual `MAP OVERLAYS` popover menu with keyboard shortcut badges. |
| `ev_c23c7056350ec081f1c704874448a95bb16d1fa8881fbcb22ef40188e699d129` | Desktop Search Autocomplete, Settings Panel & About Panel | `1280×720` (`dark`, `en-US`, `UTC`) | 15 steps: dismiss welcome & layer intro, click `button.search`, fill `"New Delhi"`, wait 2500 ms, click `button.settings`, click `button.about`, attempt `.panel.about button.close` | Verified live autocomplete via `POST https://zoom.earth/data/search/`, highlighted matching tokens in result cards, Settings segmented controls (`Time Zone`, `Time Format`, `Time Control`, `Forecast Summary`, `Map & Display`, `Animation`, `Units`, `Language`), About panel (`Data Sources`, `Mobile App` QR code), and discovered that desktop side panels close by toggling their outer right-toolbar icon button (`button.about` morphs its icon to `×`) whereas `.panel.about button.close` is hidden on desktop viewports. |
| `ev_c34d21620e870d15bdc87d264d4e3a385c0e73316cbd95cfc55a48386c75a2f1` | Mobile Responsive Layout & Breakpoint Behavior | `390×844` (`dark`, `en-US`, `UTC`) | 8 steps: `mobile_initial_wait`, `mobile_close_welcome`, `mobile_view_satellite`, `mobile_close_intro`, `mobile_clean_ui`, attempt `button.layers-dropdown` | Verified mobile adaptation: top-left brand pill collapses to a circular globe icon; desktop left sidebar (`nav.panel.layers`) and `button.layers-dropdown` are hidden; a blue circular FAB (`button.layers`) appears in the bottom-right corner; Map Overlays button moves to bottom-left; `Zoom in/out`, `Measure`, coordinates, and scale bar are hidden on mobile to preserve map area. |
| `ev_ffd1f51db8a863d3428cf0c9acfcf01d2065c0b8a4a4bdfdca253817a073b60d` | Timeline Mode Switch, Wind Layer Animation & Zoom In | `1280×720` (`dark`, `en-US`, `UTC`) | 11 steps: dismiss welcome, open Settings, click `label[for="setting-time-control-timeline"]`, close Settings, click `label[for="category-wind"]`, dismiss wind intro, click `.group.zoom button.in` | Verified switching bottom time control from `Clock` spinner to horizontal scrollable `Timeline` ruler (`Now 03:55` with playhead and ticks); verified `Wind` layer (`Wind Speed` / `Wind Gusts`, animated wind particles, directional city wind badges e.g. `New Delhi 4 ↘`, `km/h` legend `0–120`); verified Zoom In (`z=5` to `z=6`, scale bar `500 km` to `200 km`, regional state/city labels loaded from `/static/labels/v12/en/6/{x}/{y}.json`); verified `localStorage` keys (`ze_welcome`, `ze_introsLayer`, `ze_layer`, `ze_timeControl`). |

---

## 2. Detailed Analysis by Functional Area

### A. Overall Layout

1. **Full-Screen Map and Page Composition**
   - **Observed Structure (`ev_10349983...`, `ev_217069da...`)**: The entire application renders inside a single `100vw × 100vh` `overflow: hidden` container (`<main class="layer-satellite ui-blur ui-blur-dark ui-clock ui-ad-none ui-time-24 ui-dialog-welcome ui-layers">`).
   - **Map Viewport (`<section id="map" tabindex="0">`)**: Contains `.viewport.notranslate` with a full-bleed WebGL `<canvas width="1280" height="720">` and synchronized HTML overlay layers:
     - `.map-crosshair` (center screen reticle when enabled).
     - `.map-graticule` (`.lon`, `.lat` coordinate grid labels).
     - `.overlays`: `.map-pressure-centers` (`H`/`L` symbols), `.map-marker` (selected location pin), `.map-favorites` (saved locations), and `.map-labels` (GPU-transformed `translate3d(x, y, 0)` interactive `<a class="label" href="/places/...">` city/country labels and value pills).
   - **State-Driven Root Classes**: `<main>` uses root modifier classes (`layer-satellite`, `ui-blur`, `ui-blur-dark`, `ui-clock`, `ui-time-24`, `ui-dialog-welcome`, `ui-layers`) to coordinate panel visibility and theme styles without layout thrashing.

2. **Header and Navigation**
   - **Observed Structure (`ev_217069da...`)**: There is no traditional full-width horizontal navbar. Instead, the top-left corner features a floating translucent pill button (`button.title`, `aria-label="Zoom Earth"`) containing a circular globe mark and `"ZOOM EARTH"` wordmark.
   - **Expandable Brand Menu (`nav.panel.title`, `width: 196px; max-height: 220px`)**: Clicking `button.title` toggles a compact dropdown menu offering `Settings`, `About`, `Share`, `Measure Distance`, and `Measure Area`.

3. **Floating Controls and Toolbars**
   - **Top-Right Vertical Action Stack (`ev_217069da...`, `ev_c23c7056...`)**:
     - `button.search` (40×40 px circular translucent dark button with magnifying glass icon; morphs to `×` when Search panel is open).
     - `button.settings` (circular button with gear icon; morphs to `×` when Settings panel is open).
     - `button.about` (circular button with `i` info icon; morphs to `×` when About panel is open).
     - `div.group.measure` (vertical 2-button pill containing `button.distance` "Measure distance" and `button.area` "Measure area"; hidden while welcome modal is active, revealed after dismissal).
     - `button.geolocation` (circular button with crosshair icon, `aria-label="Show your location"`).
   - **Bottom-Right Control Cluster (`ev_217069da...`, `ev_ffd1f51d...`)**:
     - `div.group.overlays`: Vertical quick-toggle pill along the right edge above the zoom buttons. Dynamically displays 2–3 contextual overlay toggles (`radar`, `wind-animation`, `heat`, `clouds`, etc.) plus `button.more` (`...` ellipsis icon, which morphs to `×` when `nav.panel.overlays` is open).
     - `div.group.zoom`: Vertical 2-button pill (`button.in` `+`, `button.out` `-`) positioned in the bottom-right corner above the scale bar.
     - `button.model`: Horizontal segmented pill (`ICON 13 km | GFS 22 km`) positioned to the left of the zoom buttons when a Forecast layer (`Precipitation`, `Wind`, `Temperature`, `Humidity`, `Pressure`) is active; hidden on `Satellite` and `Radar` layers.

4. **Side Panels and Expandable Menus**
   - **Left Layer Selector Panel (`nav.panel.layers`, `width: 174px; max-height: 460px`)**:
     - Anchored below `button.title` on the top-left on desktop (`ev_217069da...`).
     - Divided into two groups:
       - `LIVE MAPS` (with collapse chevron `button.arrow`): `Satellite` (expands sub-options `Live` and `HD`), `Radar`.
       - `FORECAST MAPS`: `Precipitation`, `Wind` (expands `Wind Speed`, `Wind Gusts`), `Temperature` (expands `Actual`, `Feels Like`), `Humidity` (expands `Relative`, `Dew Point`, `Wet Bulb`), `Pressure`.
   - **Map Overlays Popover (`nav.panel.overlays`, `width: 233px; max-height: 488px`)**:
     - Floating popover anchored to the left of `.group.overlays` in the bottom-right (`ev_217069da...`).
     - Contains checkboxes for up to 15 map overlays, each with an icon, label, and right-aligned `<kbd class="hotkey">` badge (`R`, `Q`, `O`, `W`, `I`, `H`, `F`, `S`, `L`, `V`, `E`, `K`, `N`, `G`, `C`).
   - **Right-Side Drawer Panels (`aside.panel.search`, `aside.panel.settings`, `aside.panel.about`, `aside.panel.weather`, `aside.panel.storm`)**:
     - Floating card panels (`~340–380px` wide on desktop) positioned to the left of the right-edge circular toolbar (`ev_c23c7056...`).
     - Opening one right-side panel automatically closes any previously open right-side panel (verified when clicking `button.settings` while `aside.panel.search` was open, and clicking `button.about` while `aside.panel.settings` was open in `ev_c23c7056...`).

5. **Information Panels and Legends**
   - **Color Bar Legend (`div.legend`)**: Positioned at bottom-left (`ev_217069da...`, `ev_ffd1f51d...`), directly above `.footer-coordinate`. Hidden (`class="legend none"`) on `Satellite` layer; visible on `Radar` and all `Forecast Maps` with unit label (e.g., `Rain`, `°C`, `km/h`) and color-stop values (`Light | Moderate | Heavy | Snow`, `-30` to `50 °C`, `0` to `120 km/h`).
   - **Footer Coordinates (`div.footer-coordinate`)**: Bottom-left readout (`22° 12′ N  78° 29′ E`) in DMS or decimal degrees.
   - **Attribution (`footer.attribution`)**: Bottom-center subtle text (`© Zoom Earth, EUMETSAT, Meteosat-IODC` or `© Zoom Earth, OpenStreetMap, DWD/ICON 29/18Z`).
   - **Scale Bar (`div.scale-line`)**: Bottom-right dynamic width bar (`500 km` at zoom 5, `200 km` at zoom 6).
   - **Onboarding & Alert Dialogs (`div.dialogs`, `aside.panel.dialog.*`)**: Centered glassmorphic modal cards for initial `welcome`, contextual first-visit `layer-intro` / `model-intro`, `heat-consent` disclaimer, and `outage` notifications.

6. **Typography, Spacing, Borders, and Visual Hierarchy**
   - **Typography (`ev_10349983...`)**: Preloads custom geometric sans-serif font `Museo Sans Cyrl` in weights `500` (regular UI text/labels), `700` (active buttons, city labels, headers), and `900` (heavy brand/badge titles).
   - **Surfaces & Glassmorphism**: Uses `backdrop-filter: blur(...)` (`ui-blur ui-blur-dark`) with dark translucent backgrounds (`rgba(30, 34, 42, 0.78)`), `12px–24px` border-radius pills, subtle `1px solid rgba(255, 255, 255, 0.12)` borders, and drop shadows so map imagery remains visible beneath floating chrome.
   - **Color Hierarchy**: Primary interactive CTA accent is vivid azure (`#0088ff`), active pill selection uses high-contrast white (`#ffffff` background with dark text in Settings/Model toggles) or translucent white pill highlight (`rgba(255, 255, 255, 0.18)` in Layer menu), and map city labels use dark halo text-shadows for legibility over bright clouds.

---

### B. Map Interactions

1. **Initial Map Position and Zoom**
   - **Verified (`ev_10349983...`)**: Without a URL place path or saved location, `window._ZE.place` initializes to `{ isDefault: true, isMarker: false, view: "22.21579,78.49044,5" }` (centered over India / South Asia based on IP geolocation `countryCode: "IN"` at zoom level `5`).
2. **Pan and Zoom Controls**
   - **Verified (`ev_ffd1f51d...`)**: Clicking `.group.zoom button.in` increments zoom from level `5` to `6`, updates the bottom-right `.scale-line` from `500 km` (`width: 110px`) to `200 km`, and triggers tile requests for `/static/labels/v12/en/6/{x}/{y}.json` and `/icon/v1/.../4/{x}/{y}.webp`.
3. **Location Search**
   - **Verified (`ev_c23c7056...`)**: Clicking `button.search` opens `aside.panel.search` and focuses `input[name="q"]` (`placeholder="Find a place, storm, or fire…"`). Typing `"New Delhi"` dispatches `POST https://zoom.earth/data/search/` and populates `.query .results` with matching location cards (`New Delhi, India`, `New Delhi, Delhi, India`, etc.) with `<u>` underlined query matches. Also provides a prominent `Show Your Location` geolocation trigger.
4. **Map Attribution**
   - **Verified (`ev_217069da...`, `ev_c23c7056...`)**: Displayed continuously in `footer.attribution` at bottom-center and dynamically updated whenever the base layer or forecast model changes. Full legal and scientific source credits (DWD ICON, NOAA GFS, NOAA GOES, EUMETSAT Meteosat, JMA Himawari, NASA EOSDIS GIBS Aqua/Terra, NASA LANCE FIRMS, NHC/JTWC/NRL/IBTrACS, NIFC/CIFFC/CEMS/EFFIS/GDACS, and OpenStreetMap contributors) are listed in `aside.panel.about`.
5. **Map Loading and Error Behavior**
   - **Observed in DOM (`ev_10349983...`)**:
     - Timeline/Clock play button wraps an SVG `<svg class="progress">` circular ring spinner (`class="panel time timeline loading"`) while animation frames load.
     - `aside.notices` displays non-blocking banner alerts (`No connection`, `Slow connection`, `Graphics error`, `Imagery is unavailable. More info`, `Maximum resolution`).
     - `aside.panel.dialog.outage` displays a modal explaining upstream satellite outages (`Live Satellite Outage: Imagery from {source} is currently unavailable.`).
     - Async drawers (`aside.panel.search`, `aside.panel.weather`, `article.panel.storm-details`) contain `.activity .dots` (3-dot pulse loader) and `.message` (`<p>Unable to load</p><button>Try Again</button>`).
6. **Available Map Styles**
   - **Verified (`ev_217069da...`)**: Rather than switching between street/topographic road maps, Zoom Earth renders meteorological imagery over a custom multi-layer canvas:
     - Natural Earth / bathymetric + shaded relief base tiles (`Satellite` Geocolor or `HD` NASA MODIS/VIIRS true-color).
     - Dark-Contoured land/ocean base under `Precipitation` / `Radar` (with selectable `Precipitation Theme: Light | Dark` in Settings).
     - Vector-raster hybrid border lines (`tiles.zoom.earth/static/line/v1/1x/webp/{z}/{x}/{y}.webp`) and JSON-positioned interactive HTML place labels (`tiles.zoom.earth/static/labels/v12/en/{z}/{x}/{y}.json`).

---

### C. Weather and Environmental Layers

All layers, sub-layers, models, and overlays observed and verified in `ev_10349983...`, `ev_217069da...`, and `ev_ffd1f51d...`:

| Category | Layer / Control | Sub-Options / Models | Tile / Data Endpoint Pattern Observed | Legend & Visual Representation |
| :--- | :--- | :--- | :--- | :--- |
| **Live Maps** | **Satellite** | `Live` (Geostationary), `HD` (Polar-orbiting Aqua/Terra) | `/times/geocolor.json`, `/geocolor/{sat}/{YYYY-MM-DD}/{HHmm}/{z}/{x}/{y}.jpg` (`msg-iodc`, `himawari`), `/times/gibs.json` | No color bar legend; full-color composite cloud & surface imagery updated every 10–15 min (`Live`) or twice daily (`HD`). |
| **Live Maps** | **Radar** | — | `/times/radar.json` | Bottom-left `Rain / Snow` color bar (`mm/h`, `in/h`, or `dBZ`); high-frequency radar reflectivity over radar coverage zones. |
| **Forecast Maps** | **Precipitation** | Models: `ICON 13 km`, `GFS 22 km` | `/times/icon.json`, `/times/gfs.json` | Bottom-left `Rain: Light | Moderate | Heavy | Snow` gradient legend; animated rain/snow + cloud cover. |
| **Forecast Maps** | **Wind** | `Wind Speed`, `Wind Gusts` (`ICON 13 km`, `GFS 22 km`) | `/icon/v1/wind-speed/...` | Bottom-left `km/h` (`m/s`, `mph`, `knots`, `Bft`) scale `0–120`; animated white streamline particles + city pill badges with speed and direction arrow (`New Delhi 4 ↘`). |
| **Forecast Maps** | **Temperature** | `Actual` (2m), `Feels Like` (`ICON 13 km`, `GFS 22 km`) | `/icon/v1/temperature/webp/2m/{date}/{run}/{fHour}/{z}/{x}/{y}.webp` | Bottom-left `°C` (`°F`) color ramp (`-30` to `50 °C`); smooth thermal raster + city temperature pills (`Mumbai 29°`). |
| **Forecast Maps** | **Humidity** | `Relative`, `Dew Point`, `Wet Bulb` | Model raster tiles (`ICON` / `GFS`) | Percentage (`%`) or temperature (`°C`/`°F`) color ramp + heat-stress classifications (`No Heat Stress` to `Lethal Heat Stress`). |
| **Forecast Maps** | **Pressure** | Mean Sea Level Pressure (`ICON` / `GFS`) | Model raster tiles + `.map-pressure-centers` | Atmospheric pressure ramp (`hPa`, `mb`, `mmHg`, `inHg`) + `H` / `L` pressure center markers and isobars. |
| **Map Overlays** | **15 Toggleable Overlays** | `Radar [R]`, `Radar Coverage [Q]`, `Clouds [O]`, `Wind Animation [W]`, `Pressure Isobars [I]`, `Heat Spots [H]`, `Active Fires [F]`, `Tropical Systems [S]`, `Labels [L]`, `Values [V]`, `Temperatures [E]`, `Border Lines [K]`, `Night Boundary [N]`, `Coordinate Grid [G]`, `Crosshair [C]` | `/data/storms/?date=...`, `/data/storms/?id=...`, `/data/fires/latest.json` | Composable vector/canvas overlays rendered on top of the active base weather layer. |

---

### D. Time-Based Interactions

1. **Dual Time Controls (`Clock` vs. `Timeline`)**:
   - **Clock Mode (`div.panel.time.clock`, default in `ev_217069da...`)**: Compact bottom-center pill displaying `Play/Pause` button, `Date` (`30 Oct` with `^`/`v` day steppers), `Hour` (`03` with `^`/`v` hour steppers), blinking `:`, `Minute` (`15` with `^`/`v` 15-min steppers), and `Latest time / Reset to now` (`>|`) button. Also includes `.clock-hd` (`Year`, `Month`, `Day`, `AM/PM`) when viewing archival `HD Satellite` imagery (`since 2000`).
   - **Timeline Mode (`div.panel.time.timeline`, verified in `ev_ffd1f51d...`)**: Toggled via Settings (`Time Control: Timeline` or hotkey `J`). Renders a wide (`~600px`) bottom-center pill with current timestamp label (`Now 03:55`) above a horizontally draggable tick-mark timeline ruler with a centered blue vertical indicator bar, `Play/Pause` on the left, and `Reset to now` (`>|`) on the right.
2. **Play, Pause, and Frame Navigation**:
   - `button.play` / `button.pause` starts/stops frame-by-frame temporal playback across cached satellite/radar frames or forecast steps.
   - `button.recent.latest` / `button.recent.now` jumps immediately back to the latest observation or current forecast hour (`disabled` when already at `Now`/`Latest`).
3. **Animation Configuration (`aside.panel.settings`)**:
   - `Animation Speed [A]`: `Slow` | `Medium` (default) | `Fast`.
   - `Satellite Loop Duration`: `3 Hours` (default) | `6 Hours` | `12 Hours` | `24 Hours`.
   - `Satellite Animation Style`: `Fast` (default) | `Smooth`.

---

### E. User Experience & Responsive Behavior

1. **Desktop (`1280×720`) vs. Mobile (`390×844`) Layout Comparison (Verified in `ev_217069da...` vs. `ev_c34d2162...`)**:
   - **Desktop (`1280×720`)**:
     - Left sidebar (`nav.panel.layers`) stays persistently open below the top-left `ZOOM EARTH` brand pill.
     - Right edge shows all utility buttons (`Search`, `Settings`, `About`, `Measure`, `Geolocation`), quick overlay toggles (`.group.overlays`), Forecast Model toggle (`ICON / GFS`), and `.group.zoom` (`+`/`-`).
     - Bottom bar displays `.legend` (left), `.footer-coordinate` (bottom-left), `.panel.time` (center), `.attribution` (bottom-center), and `.scale-line` (bottom-right).
   - **Mobile (`390×844`)**:
     - Top-left brand pill collapses into a single `40×40px` circular globe button (`button.title`).
     - Left sidebar (`nav.panel.layers`) is hidden by default and replaced by a prominent **blue circular FAB (`button.layers`)** in the bottom-right corner displaying the active layer icon.
     - Map Overlays button (`button.more`) relocates to the **bottom-left corner** as a circular icon button.
     - `Search` and `Geolocation` stack vertically above the bottom-right blue Layers FAB.
     - `Zoom in/out` (`+`/`-`), `Measure`, `.footer-coordinate`, and `.scale-line` are hidden on mobile (relying on pinch-to-zoom).
2. **Persistence & Storage (`ev_10349983...`, `ev_ffd1f51d...`)**:
   - Uses `localStorage` keys prefixed with `ze_` (`ze_welcome`, `ze_introsLayer`, `ze_layer`, `ze_timeControl`, plus unit preferences) and `sessionStorage` (`ze_layersMenu`, `ze_measure`) so user state persists across reloads without requiring an account.

---

## 3. Comprehensive 5-Point Feature Matrix

For each inspected feature, the table below records: (1) Visible presence, (2) Trigger action, (3) Post-action behavior, (4) Verification status with REA Evidence ID, and (5) Earth Monitor equivalent implementation.

| # | Feature | 1. Visibly Present | 2. Trigger Action | 3. Post-Action Behavior | 4. Verification Status | 5. Earth Monitor Original Implementation |
| :- | :--- | :--- | :--- | :--- | :--- | :--- |
| **A1** | **Full-Viewport Map Canvas** | `100vw × 100vh` WebGL `<canvas>` with border lines, place labels, and storm markers (`<section id="map">`). | Initial page navigation to `/`. | Renders interactive Earth view centered on default or URL coordinates (`22.22°N, 78.49°E, z=5`). | **Directly Verified** (`ev_10349983...`) | MapLibre GL JS full-viewport `<MapCanvas />` (`100dvw × 100dvh`) with dark/satellite basemap and vector place/border layers. |
| **A2** | **Welcome / Location Onboarding Modal** | Centered dark glass dialog (`aside.panel.dialog.welcome`) with `Show Your Location`, `Search...`, and `Close` (`×`). | First visit with empty `localStorage.ze_welcome`. | Clicking `×` sets `localStorage.ze_welcome`, hides welcome dialog, and reveals left layer panel, bottom time bar, and measure tools. | **Directly Verified** (`ev_10349983...`, `ev_217069da...`) | Non-intrusive, dismissible welcome card or direct-to-map experience with persistent `localStorage` onboarding state. |
| **A3** | **Top-Left Brand Pill & App Menu** | Floating pill (`button.title`) on desktop; compact circular globe icon on mobile (`390×844`). | Click `button.title`. | Toggles `nav.panel.title` dropdown (`Settings`, `About`, `Share`, `Measure Distance`, `Measure Area`). | **DOM & Visually Verified** (`ev_10349983...`, `ev_c34d2162...`); click expansion **Inferred from DOM** | Top-left `<AppBrandHeader />` with Earth Monitor logo, status badge, and quick-access menu. |
| **A4** | **Left Weather Layer Sidebar (Desktop) / Bottom-Right FAB (Mobile)** | Desktop: `nav.panel.layers` (`174px` wide) with `LIVE MAPS` and `FORECAST MAPS`. Mobile: blue circular FAB (`button.layers`) at bottom-right. | Click any layer radio label (e.g., `Precipitation`, `Temperature`, `Wind`) or mobile FAB. | Switches active weather layer, fetches corresponding tile set, expands sub-layer options (`Actual`/`Feels Like`, `Wind Speed`/`Wind Gusts`), and updates legend & attribution. | **Directly Verified** (`ev_217069da...`, `ev_c34d2162...`, `ev_ffd1f51d...`) | Responsive `<LayerControlPanel />`: persistent glass sidebar on desktop (`≥768px`) and bottom sheet / FAB trigger on mobile (`<768px`). |
| **B1** | **Zoom Controls & Scale Bar** | Vertical `+` / `-` pill (`.group.zoom`) and dynamic scale bar (`500 km` at `z=5`). | Click `.group.zoom button.in` (`+`) or `button.out` (`-`), or scroll/pinch map. | Increments/decrements zoom level (`z=5` → `z=6`), updates scale bar (`500 km` → `200 km`), and loads higher-detail regional labels. | **Directly Verified** (`ev_ffd1f51d...`) | MapLibre navigation controller + custom `<ZoomControlGroup />` (`+`/`-`/compass reset) and reactive metric/imperial `<ScaleBar />`. |
| **B2** | **Coordinate Readout** | Bottom-left `.footer-coordinate` (`22° 12′ N 78° 29′ E`). | Move cursor / pan map or toggle `DMS` vs `decimal degrees` in Settings. | Updates latitude/longitude display dynamically. | **Visually & DOM Verified** (`ev_10349983...`, `ev_217069da...`) | `<CoordinateReadout />` displaying cursor/viewport-center coordinates with one-click toggle between DMS and decimal degrees. |
| **B3** | **Location Search & Autocomplete** | Top-right `button.search` opening `aside.panel.search` with search input and `Show Your Location` button. | Click `button.search` and type query (`"New Delhi"`). | Sends search request, displays autocomplete list with highlighted matches; selecting a result pans/zooms map to location. | **Directly Verified** (`ev_c23c7056...`); result selection navigation **Partially Verified / DOM Inspected** | `<LocationSearchDrawer />` powered by a documented geocoding provider (Nominatim / Photon / MapTiler Geocoding) with debounce, keyboard navigation, and browser Geolocation API support. |
| **B4** | **Dynamic Map & Data Attribution** | Bottom-center `footer.attribution` and `aside.panel.about` `Data Sources` list. | Switch between Satellite (`EUMETSAT, Meteosat-IODC`) and Forecast (`OpenStreetMap, DWD/ICON`). | Updates bottom attribution string in real time to credit active basemap and meteorological provider. | **Directly Verified** (`ev_217069da...`, `ev_c23c7056...`) | `<AttributionFooter />` + `<DataSourcesModal />` dynamically rendering required legal attribution for active base tiles and weather providers. |
| **C1** | **Satellite Layer (`Live` & `HD`)** | Full-Earth cloud/surface satellite composite (`Live` geostationary & `HD` NASA GIBS polar-orbiting). | Default layer or click `Satellite` (`Live` / `HD`). | Renders time-stamped satellite raster tiles (`geocolor` or `gibs`) across the map canvas. | **Directly Verified** (`ev_10349983...`, `ev_217069da...`) | Raster tile adapter supporting NASA EOSDIS GIBS WMTS/XYZ tiles (MODIS/VIIRS/GOES/Himawari) with explicit timestamp metadata. |
| **C2** | **Precipitation & Radar Layers** | Rain/snow intensity overlay with `Rain: Light | Moderate | Heavy | Snow` color legend. | Click `Precipitation` or `Radar`. | Renders precipitation/radar raster tiles, displays bottom-left rain/snow legend, and shows `ICON / GFS` model selector (for forecast). | **Directly Verified** (`ev_217069da...`) | Modular weather layer interface supporting RainViewer Radar API (live radar tiles) and Open-Meteo / NOAA GFS precipitation layers. |
| **C3** | **Temperature Layer (`Actual` & `Feels Like`)** | Thermal color-ramp raster + city temperature pill badges (`20°`, `28°`) + `-30 to 50 °C` legend. | Click `Temperature` in left sidebar. | Expands `Actual` and `Feels Like` sub-options, renders 2m temperature raster and city value pills, and displays `°C`/`°F` color bar legend. | **Directly Verified** (`ev_217069da...`) | Temperature layer adapter with configurable color ramp legend (`°C`/`°F`) and point-value markers for major cities/viewport grid points. |
| **C4** | **Wind Layer (`Wind Speed` & `Wind Gusts`)** | Animated white wind particles over wind-speed color field + directional city wind pills (`4 ↘`) + `0–120 km/h` legend. | Click `Wind` in left sidebar. | Expands `Wind Speed` / `Wind Gusts`, starts particle streamline canvas animation, renders speed raster and `km/h` legend. | **Directly Verified** (`ev_ffd1f51d...`) | Pluggable wind visualization layer (raster speed ramp + optional WebGL/Canvas2D streamline particle layer) fed by real forecast grid data. |
| **C5** | **Map Overlays Menu (`nav.panel.overlays`)** | Bottom-right quick-toggle pill (`.group.overlays`) + expandable `MAP OVERLAYS` checklist with hotkeys (`W`, `F`, `S`, `L`, `V`, `N`, `G`, `C`). | Click `.group.overlays button.more` (`...`). | Opens `MAP OVERLAYS` popover filtered to overlays compatible with the active layer; button icon changes to `×`. | **Directly Verified** (`ev_217069da...`) | `<MapOverlaysPopover />` with toggleable vector/canvas overlays (labels, borders, coordinate grid, crosshair, night terminator) and keyboard shortcuts. |
| **D1** | **Time Control: Clock vs. Timeline Bar** | `Clock` spinner (`30 Oct 03:15`) or horizontal `Timeline` tick ruler (`Now 03:55`) with `Play/Pause` and `Reset to Now` (`>|`). | Toggle `Time Control` (`Timeline` / `Clock`) in Settings (`label[for="setting-time-control-timeline"]`). | Switches bottom-center bar from stepper clock to scrollable tick ruler with blue playhead indicator and saves `localStorage.ze_timeControl`. | **Directly Verified** (`ev_217069da...`, `ev_ffd1f51d...`) | `<TimelineControlBar />` supporting frame scrubbing, play/pause loop, step forward/backward, `Now` reset, and UTC/Local time formatting (scoped for Phase 2 once multi-frame time series are connected; clean time badge/placeholder in V1 foundation). |
| **E1** | **Settings Drawer & Unit Preferences** | Right-side `aside.panel.settings` with segmented radio pills for Time, Display, Animation, Units, and Language. | Click `button.settings` (top-right gear icon). | Opens Settings drawer, morphs gear icon to `×`, and persists changes to `localStorage`. | **Directly Verified** (`ev_c23c7056...`, `ev_ffd1f51d...`) | `<SettingsDrawer />` managing typed user preferences (`temperatureUnit`, `windUnit`, `distanceUnit`, `coordinateFormat`, `timeZone`) persisted in `localStorage`. |
| **E2** | **About Drawer & Data Source Transparency** | Right-side `aside.panel.about` listing app overview, mobile QR code, and scientific data citations. | Click `button.about` (top-right `i` icon). | Opens About drawer and morphs `i` icon to `×`. | **Directly Verified** (`ev_c23c7056...`) | `<AboutDrawer />` detailing Earth Monitor architecture, data provider licenses, update frequencies, and links to upstream scientific agencies. |
| **E3** | **Loading, Empty, and Error States** | SVG ring spinner on Play button; `.activity .dots` loader and `"Unable to load"` + `"Try Again"` button in panels; `aside.notices` banners; `outage` modal. | Slow network, tile failure, or search with no results. | Displays non-blocking loading indicators or actionable retry UI without crashing the map canvas. | **DOM Verified** (`ev_10349983...`); runtime outage modal **Not Triggered Live (Inferred from DOM)** | Standardized `<LoadingSpinner />`, `<BannerNotice />` (offline/tile error warnings), and panel error boundaries with retry actions. |

---

## 4. Documented Limitations & Unverified Items

In accordance with strict engineering and REA evidence rules, the following items were observed in the DOM or static configuration but were **not directly exercised as live runtime failures or full end-to-end flows**:

1. **Interactive Measurement Drawing (`Measure Distance` / `Measure Area`)**: The buttons (`.group.measure button.distance`, `button.area`) and action dialog (`div.panel.measure` with `Cancel`, `Undo`, `Done`) were verified in the DOM and screenshots (`ev_217069da...`), but multi-point canvas polygon drawing was not executed.
2. **Active Storm Track & Fire Incident Drill-Down Panels (`aside.panel.storm`, `article.panel.storm-details`)**: Storm endpoints (`/data/storms/?id=in-fa-2026`, `91b-2026`, `vance-2026`) and purple track markers near Phuket were observed in network logs and screenshots (`ev_217069da...`), and panel templates were verified in the DOM, but clicking an active cyclone marker to populate the storm table was not executed.
3. **Simulated Upstream Outage & Offline Banner States (`aside.panel.dialog.outage`, `aside.notices`)**: The exact DOM templates and localized strings (`Live Satellite Outage`, `No connection`, `Graphics error`) were verified in `window._ZE.strings` (`ev_10349983...`), but upstream tile servers (`tiles.zoom.earth`) responded with HTTP 200 throughout testing, so outage modals were not triggered live.
4. **Zoom Earth Internal APIs Are Proprietary**: Network inspection confirmed that `tiles.zoom.earth` and `zoom.earth/data/*` are first-party endpoints operated by Neave Interactive Ltd. **Earth Monitor must not consume or scrape `zoom.earth` endpoints** and will instead use documented, appropriately licensed third-party map tile and meteorological data providers.
