# Quality Assurance & Test Plan — Earth Monitor

## 1. Testing Philosophy & Quality Gates

Every feature branch and Pull Request in **Earth Monitor** must pass five sequential quality gates before merge:

1. **Gate 1 — Static Type Checking (`npm run typecheck`)**: `tsc --noEmit` with `"strict": true` and `"noUncheckedIndexedAccess": true` must exit with code `0`.
2. **Gate 2 — Linting & Formatting (`npm run lint`)**: ESLint (`eslint-config-next`, accessibility rules `jsx-a11y`) must report zero errors.
3. **Gate 3 — Unit & Component Integration Tests (`npm run test`)**: Vitest + React Testing Library suite covering pure domain logic, unit converters, provider adapters, and UI state transitions must pass 100%.
4. **Gate 4 — Production Build Verification (`npm run build`)**: Next.js production bundle compilation and static page generation must succeed without errors.
5. **Gate 5 — Browser E2E & REA Verification (`npm run test:e2e`)**: Playwright browser tests and REA `capture_browser_scenario` inspections verifying layout, map rendering, responsive breakpoints, and failure handling.

---

## 2. Unit & Integration Test Specifications (Vitest)

| Test Suite | Target Modules | Key Test Cases |
| :--- | :--- | :--- |
| **Coordinate & Unit Converters** | `src/settings/utils/unitConverters.ts` | - Convert decimal degrees (`22.21579, 78.49044`) to DMS (`22° 12′ N, 78° 29′ E`) across all 4 hemispheres (`N/S/E/W`) and `0°` / `180°` boundaries.<br>- Convert temperature (`°C` ↔ `°F`), wind speed (`km/h` ↔ `m/s`, `mph`, `knots`, `Beaufort`), pressure (`hPa` ↔ `mb`, `mmHg`, `inHg`), and distance (`km` ↔ `miles`, `NM`). |
| **Weather & Tile Adapters** | `nasaGibsAdapter.ts`, `rainViewerAdapter.ts`, `openMeteoAdapter.ts` | - Parse valid RainViewer `weather-maps.json` payload into ordered historical/nowcast frame URLs.<br>- Construct valid NASA GIBS WMTS/XYZ tile URLs with UTC date clamping.<br>- Reject malformed JSON payloads and throw typed `ProviderAdapterError` on HTTP 429/500 or `AbortError` timeout. |
| **Geocoding Adapter** | `geocodingAdapter.ts` | - Debounce rapid input changes (`300ms`) and abort superseded requests.<br>- Normalize Photon GeoJSON features into `GeocodingResult[]` with safe fallback when optional fields (`state`, `postcode`) are absent.<br>- Handle empty `features: []` and HTTP 5xx errors gracefully. |
| **Shell & Drawer State** | `AppShell`, `useActiveDrawer`, `useKeyboardShortcuts` | - Opening `Settings` automatically closes `Search` or `About` (mutual exclusion).<br>- Clicking the active right-toolbar icon button (`Search`, `Settings`, `About`) or pressing `Escape` closes the open drawer.<br>- Pressing hotkeys (`L`, `K`, `G`, `C`, `Z`, `T`) toggles the corresponding overlay/setting when focus is outside `<input>`, and is ignored when typing inside `<input name="q">`. |

---

## 3. Map Interaction Test Specifications

Map interactions are verified at both the controller integration level and in a real headless Chromium browser via Playwright / REA:

1. **Initial Viewport & URL Hydration (`TEST-MAP-01`)**:
   - Loading `/` without query parameters initializes the map to `22° 12′ N, 78° 29′ E` at zoom `5` and displays a `500 km` scale bar.
   - Loading `/?lat=28.6139&lon=77.2090&z=8&layer=radar` initializes the map directly over New Delhi at zoom `8` with the `Radar` layer active.
2. **Zoom Controls & Scale Bar Reactivity (`TEST-MAP-02`)**:
   - Clicking `button[aria-label="Zoom in"]` increments zoom (`z=5` → `z=6`), updates the URL query string after debounce, and decreases the scale bar distance (`500 km` → `200 km`).
   - Clicking `button[aria-label="Zoom out"]` decrements zoom (`z=6` → `z=5`).
3. **Layer Switching & Attribution Sync (`TEST-MAP-03`)**:
   - Selecting `Satellite` in `<LayerControlPanel />` activates the NASA GIBS raster source, hides the quantitative color bar legend, and updates `<AttributionFooter />` to include `NASA EOSDIS GIBS`.
   - Selecting `Radar` activates the RainViewer raster source, renders the bottom-left `Rain / Radar (dBZ / mm/h)` color bar legend, and updates `<AttributionFooter />` to include `RainViewer`.
4. **Map Overlays Toggling (`TEST-MAP-04`)**:
   - Toggling `Crosshair [C]` renders/hides the centered `<CrosshairOverlay />`.
   - Toggling `Coordinate Grid [G]` renders/hides the latitude/longitude `<GraticuleLayer />`.
   - Toggling `Place Labels [L]` and `Border Lines [K]` updates MapLibre layer visibility properties (`visible` vs `none`).

---

## 4. Responsive Layout Test Specifications (`Desktop` vs. `Mobile`)

Inspired directly by the verified REA observations (`ev_217069da...` at `1280×720` and `ev_c34d2162...` at `390×844`):

| Viewport Profile | Dimensions | Verified Layout Assertions |
| :--- | :--- | :--- |
| **Desktop Landscape** | `1280 × 720` (`deviceScaleFactor: 1`) | - Zero horizontal or vertical window scrollbars (`document.documentElement.scrollWidth === 1280`, `scrollHeight === 720`).<br>- Top-left brand pill displays both globe icon and `"EARTH MONITOR"` text.<br>- Left `<LayerControlPanel />` sidebar is persistently visible.<br>- Bottom-right displays `<ZoomControlGroup />` (`+`/`-`) and `<ScaleBar />`.<br>- Bottom-left displays `<CoordinateReadout />` and `<LayerLegend />`. |
| **Tablet Portrait** | `768 × 1024` (`deviceScaleFactor: 2`) | - All floating controls remain within viewport bounds with minimum `12px` edge padding.<br>- Right-side drawers (`Search`, `Settings`, `About`) open as floating cards (`max-width: 360px`) without overlapping the left layer panel. |
| **Mobile Portrait** | `390 × 844` (`deviceScaleFactor: 2`) | - Top-left brand header collapses to a compact `40×40px` circular globe button.<br>- Desktop left sidebar is hidden by default; a prominent **bottom-right circular blue Layers FAB** (`button[aria-label="Layers"]`) is visible.<br>- Clicking the mobile Layers FAB opens the layer selection sheet.<br>- Map Overlays button is accessible in the bottom-left corner.<br>- All touch targets measure at least `40×40px` bounding box. |

---

## 5. API Failure, Offline & Resilience Test Specifications

Using Playwright network interception (`page.route(...)`) and Vitest mocks, the following failure scenarios must be tested explicitly:

1. **Geocoding Search API Failure (`TEST-ERR-01`)**:
   - **Trigger**: Intercept `https://photon.komoot.io/api/**` and return HTTP `503 Service Unavailable`.
   - **Expected Behavior**: `<SearchDrawer />` stops its loading spinner, displays `"Unable to load search results"` and a `"Try Again"` button. Clicking `"Try Again"` re-issues the request.
2. **Geocoding Empty Results (`TEST-ERR-02`)**:
   - **Trigger**: Search for a nonsense string `"zzzzxxxxqqqq9999"` returning `{ "type": "FeatureCollection", "features": [] }`.
   - **Expected Behavior**: `<SearchDrawer />` displays a clean `"No results found"` empty state.
3. **Weather Radar Metadata / Tile Failure (`TEST-ERR-03`)**:
   - **Trigger**: Intercept `https://api.rainviewer.com/public/weather-maps.json` and return HTTP `500` or abort connection.
   - **Expected Behavior**: Map remains interactive on the base layer; a non-blocking `<BannerNotice />` appears (`"Live radar data is temporarily unavailable"`) with a `"Try Again"` action; zero unhandled promise rejections in console.
4. **Geolocation Permission Denied (`TEST-ERR-04`)**:
   - **Trigger**: Click `Show Your Location` in a browser context where `geolocation` permission is denied (`PERMISSION_DENIED = 1`).
   - **Expected Behavior**: Button resets from `"Locating…"` to `"Show Your Location"` and surfaces a clear notice explaining that location permission was denied.

---

## 6. Browser & Production Deployment Checks

Before marking Version 1 complete and deploying to Vercel:
1. **Console & Page-Error Cleanliness**: Run REA `capture_browser_scenario` with `events: ["console", "page-errors"]` against the local production server (`npm run build && npm run start`). Verify `0` uncaught `page-errors` and `0` React hydration warnings.
2. **Accessibility Audit**: Verify via REA `accessibility` capture that all interactive controls expose accessible roles and names (`button "Search"`, `button "Settings"`, `button "About"`, `button "Zoom in"`, `button "Zoom out"`, `button "Show your location"`).
3. **Vercel Preview & Production Verification**:
   - Confirm Vercel build logs pass `next build` cleanly.
   - Confirm HTTPS asset loading, CSP headers, and tile CORS requests succeed on the deployed `.vercel.app` URL.
