# Product Requirements Specification — Earth Monitor

## 1. Product Vision & Goals

**Earth Monitor** is an original, production-oriented, interactive Earth and weather monitoring web application inspired by the spatial user experience and clarity of Zoom Earth (`https://zoom.earth/`).

### 1.1 Core Product Goals
1. **Map-First Spatial Experience**: Dedicate `100%` of the viewport (`100dvw × 100dvh`) to an interactive, fluid Earth map with floating translucent glassmorphic controls that never obstruct geographic context.
2. **Scientific & Data Integrity**: Never represent simulated, mocked, or placeholder weather values as real-time meteorological observations. Every layer displayed to the user must clearly declare its upstream provider, observation/forecast timestamp, spatial resolution, and license attribution.
3. **Modular Extensibility**: Establish a clean architectural separation between the WebGL map rendering engine, UI control shell, and pluggable weather-data adapters so new environmental layers (satellite, radar, precipitation, temperature, wind, active fires, tropical storms) can be integrated incrementally without refactoring core components.
4. **Cross-Device Ergonomics**: Provide first-class desktop (persistent left layer panel, right utility stack, keyboard shortcuts, hover readouts) and mobile (compact bottom FABs, slide-up sheets, touch gestures) layouts.

---

## 2. Version 1 Scope (Foundation Milestone)

Version 1 prioritizes a rock-solid, responsive interactive Earth mapping foundation and modular layer architecture:

1. **Full-Screen Interactive Earth Map**:
   - Powered by MapLibre GL JS filling the entire browser viewport (`100dvw × 100dvh`).
   - Smooth pan, scroll-wheel/pinch zoom, double-click zoom, keyboard navigation, and URL/state-synchronized viewport coordinates (`lat`, `lon`, `zoom`).
2. **Responsive Application Shell & Floating Chrome**:
   - Top-left brand header pill (`Earth Monitor` logo + title on desktop; compact circular mark on mobile).
   - Top-right utility control stack (`Search`, `Settings`, `About`, `Geolocation`).
   - Bottom-right zoom control pill (`Zoom In +`, `Zoom Out -`, `Reset North / Bearing` when rotated) and dynamic metric/imperial scale bar (`ScaleControl`).
   - Bottom-left live coordinate readout (`DMS` or `Decimal Degrees`) and active layer legend slot.
   - Bottom-center attribution bar and time-status pill.
3. **Location Search & Browser Geolocation**:
   - Expandable right-side Search drawer (`Search` button or `/` / `Ctrl+K` shortcut).
   - Debounced geocoding search against a documented public geocoding API (e.g., OpenStreetMap Nominatim / Photon with proper User-Agent and rate-limit compliance, or MapTiler Geocoding) with loading, empty (`No results`), and error/retry states.
   - Selecting a search result smoothly flies (`flyTo`) the map to the target coordinates/bounding box and places a removable location marker.
   - `Show Your Location` button integrating the browser `navigator.geolocation` API with explicit permission-denied and timeout handling.
4. **Layer-Selection & Map-Overlay Interface**:
   - Desktop left-side floating `Layers` panel (`Live Maps` & `Forecast Maps` categories) and mobile bottom-right `Layers` FAB / sheet.
   - Support for switching base styles / initial verified public Earth layers (e.g., NASA EOSDIS GIBS True-Color / Geocolor Satellite WMTS raster layer, OpenFreeMap / CARTO Dark Matter vector basemap, and RainViewer live radar tile layer) plus clearly labeled adapter slots for upcoming forecast layers (`Precipitation`, `Wind`, `Temperature`, `Humidity`, `Pressure`).
   - Map Overlays quick-toggle menu (`Labels`, `Border Lines`, `Coordinate Grid / Graticule`, `Center Crosshair`, `Night Terminator`).
5. **Settings & Unit Preferences Drawer**:
   - Right-side Settings drawer persisting preferences in `localStorage` (`Coordinate System: DMS | Decimal`, `Distance Unit: km | miles | NM`, `Temperature Unit: °C | °F`, `Wind Speed Unit: km/h | m/s | mph | knots`, `Time Zone: Local | UTC`, `Time Format: 12-Hour | 24-Hour`).
6. **About & Data Attribution Drawer**:
   - Right-side About drawer documenting Earth Monitor's data sources, update intervals, licensing terms, and clickable attribution links, paired with a dynamic bottom-center attribution footer that updates when layers change.
7. **Comprehensive Loading, Empty, and Error States**:
   - Tile loading indicator, non-blocking network/tile error banner (`BannerNotice`) with retry capability, and drawer-level loading skeletons / retry buttons.

---

## 3. Explicit Exclusions (Out of Scope for Version 1)

To maintain strict scope discipline and prevent unverified complexity, the following are **explicitly excluded from Version 1**:

1. **No Fake or Simulated Weather Data**: Under no circumstances will random numbers, static mock JSON, or procedural noise be presented as live weather, radar, wind, or temperature readings.
2. **No Proprietary Zoom Earth API Consumption**: Earth Monitor will never call `tiles.zoom.earth` or `zoom.earth/data/*`.
3. **No User Authentication, Accounts, or Subscriptions**: No login system, OAuth, or paid `Pro` tier (`account.zoom.earth` equivalent is excluded).
4. **No Database or Custom Backend Server**: No PostgreSQL/MongoDB/Redis or standalone backend service; V1 runs as a stateless Next.js application using client-side storage (`localStorage`) for preferences and optional Next.js Route Handlers only if needed for API key proxying or caching.
5. **No Multi-Frame Forecast Scrubbing / WebGL Particle Streamlines in V1 Foundation**: Full multi-frame temporal scrubbing (`Play/Pause` loop across 24–120 forecast frames) and custom WebGL wind particle shaders are scheduled for **Phase 2** after the V1 map shell, layer registry, and data-provider adapters are verified.
6. **No Active Cyclone Cone-of-Uncertainty or Wildfire Perimeter Polygon Editors in V1**: Scheduled for Phase 3 environmental hazard overlays.

---

## 4. Functional Requirements

### 4.1 Map Engine & Viewport (`FR-MAP`)
- **`FR-MAP-01`**: The map canvas MUST occupy `100dvw × 100dvh` with `position: fixed` or `inset: 0` and zero page scrollbars (`overflow: hidden`).
- **`FR-MAP-02`**: The map MUST initialize to a configurable default viewport (`22.2158° N, 78.4904° E, zoom 5`) or restore coordinates from URL hash/query parameters (`?view=lat,lon,zoom`).
- **`FR-MAP-03`**: Panning or zooming the map MUST update the live coordinate readout (bottom-left) and scale bar (bottom-right) in real time, and debounce-sync the viewport state to the URL without polluting browser Back-button history (`history.replaceState`).
- **`FR-MAP-04`**: Zoom controls (`+` and `-`) MUST increment/decrement map zoom smoothly and disable themselves at `maxZoom` / `minZoom` boundaries.

### 4.2 Application Shell & Panels (`FR-SHELL`)
- **`FR-SHELL-01`**: Floating controls MUST use translucent dark glassmorphic styling (`backdrop-filter: blur(12px)`) with high-contrast text and visible focus rings (`:focus-visible`).
- **`FR-SHELL-02`**: Only one right-side utility drawer (`Search`, `Settings`, `About`) MAY be open at any time; opening one MUST automatically close any other open right-side drawer.
- **`FR-SHELL-03`**: When a right-side utility drawer is open, its corresponding circular toolbar button (`Search`, `Settings`, `About`) MUST display an active/close (`×`) state so clicking it again closes the drawer, in addition to an internal drawer close button and the `Escape` key.

### 4.3 Location Search & Geolocation (`FR-SEARCH`)
- **`FR-SEARCH-01`**: Typing at least 2 characters into the search input MUST trigger a debounced (`300ms`) request to the configured geocoding adapter and cancel any in-flight stale requests (`AbortController`).
- **`FR-SEARCH-02`**: Search results MUST display the primary place name, administrative region/country, and geographical coordinates, highlighting matching query substrings.
- **`FR-SEARCH-03`**: Selecting a result MUST animate the map to the location, drop a visual marker pin at the target coordinates, and record the item in `Recent Searches` (`localStorage`, capped at 5 entries with a `Clear` option).
- **`FR-SEARCH-04`**: Clicking `Show Your Location` MUST invoke `navigator.geolocation.getCurrentPosition`, display a `"Locating…"` state, and either pan/zoom to the user's coordinates or surface a clear, non-blocking message if permission is denied or unavailable.

### 4.4 Layer Controls & Overlays (`FR-LAYER`)
- **`FR-LAYER-01`**: The Layer Registry MUST support selecting one active primary base/weather layer at a time (`Satellite`, `Radar`, or `Standard Dark Basemap` in V1, with extensible definitions for `Precipitation`, `Wind`, `Temperature`, `Humidity`, `Pressure`).
- **`FR-LAYER-02`**: Each registered layer MUST define its id, label, category (`live` vs `forecast`), tile source configuration, attribution string, legend specification (unit + color stops), and availability status.
- **`FR-LAYER-03`**: Independent map overlays (`Place Labels`, `Border Lines`, `Coordinate Grid`, `Center Crosshair`) MUST be toggleable on/off via checkboxes and keyboard shortcuts (`L`, `K`, `G`, `C`) and persist across base layer switches.
- **`FR-LAYER-04`**: When a layer with a quantitative color scale is active, the bottom-left `<LayerLegend />` component MUST render the corresponding color gradient, unit label, and threshold values converted to the user's selected unit preference.

### 4.5 Attribution, Loading & Error States (`FR-STATUS`)
- **`FR-STATUS-01`**: The bottom-center attribution bar MUST always remain visible and dynamically reflect the legal attribution required by the currently active basemap and weather data provider(s).
- **`FR-STATUS-02`**: While map tiles or overlay data are actively fetching, a subtle non-blocking loading indicator MUST be shown.
- **`FR-STATUS-03`**: If a tile source or API request fails (network offline, HTTP 429/5xx, or timeout), the application MUST display a dismissible `<BannerNotice />` with a `"Try Again"` action and preserve the functioning base map without an unhandled runtime exception.

---

## 5. Nonfunctional Requirements (`NFR`)

1. **Type Safety (`NFR-TS`)**: 100% TypeScript with `"strict": true`, `"noUncheckedIndexedAccess": true`, and zero `any` types in domain models and API adapters. Runtime API payloads must be validated with schema guards (e.g., Zod or strict type predicates).
2. **Performance (`NFR-PERF`)**:
   - Initial First Contentful Paint (FCP) `< 1.5s` on desktop broadband.
   - MapLibre GL JS dynamically imported (`next/dynamic` with `ssr: false` or client boundary) so WebGL bundle weight does not block initial HTML shell rendering.
   - Smooth `60 FPS` pan/zoom interactions on WebGL2-capable browsers; automatic fallback warning if WebGL is disabled.
3. **Responsiveness (`NFR-RESP`)**:
   - Seamless layout adaptation across Desktop (`≥1024px`), Tablet (`768px–1023px`), and Mobile (`320px–767px`, verified at `390×844`).
   - All interactive touch targets MUST be at least `40×40px` (`44×44px` preferred on mobile) with safe-area-inset support (`env(safe-area-inset-*)`).
4. **Accessibility (`NFR-A11Y`)**:
   - Semantic landmarks (`<main>`, `<nav>`, `<aside>`, `<footer>`), explicit `aria-label` and `aria-expanded` attributes on all icon-only buttons, keyboard operability (`Tab`, `Enter`, `Space`, `Escape`, and single-key hotkeys when not focused inside an `<input>`), and `prefers-reduced-motion` support.
5. **Security & Privacy (`NFR-SEC`)**:
   - No client-side exposure of secret keys; public tile keys (if any) restricted by HTTP `Referer` origin.
   - Geolocation coordinates are processed strictly client-side and never transmitted to analytics or third-party tracking servers.

---

## 6. Version 1 Acceptance Criteria

| ID | Criterion | Verification Method |
| :--- | :--- | :--- |
| **AC-01** | Opening `/` renders a full-screen (`100dvw × 100dvh`) interactive Earth map with zero horizontal or vertical page scrollbars on both `1280×720` and `390×844` viewports. | Playwright E2E test + REA visual inspection. |
| **AC-02** | Clicking `+` and `-` zoom controls changes the map zoom level, updates the scale bar distance, and updates the coordinate/viewport state. | Playwright E2E test + unit test. |
| **AC-03** | Opening the Search drawer, typing `"New Delhi"`, and selecting the top result pans/zooms the map to New Delhi and places a location marker. | Playwright E2E test with mocked & live geocoding check. |
| **AC-04** | Switching layers in the Layer Control Panel updates the map tile source, active layer indicator, bottom-left legend (when applicable), and bottom-center attribution text. | Vitest integration test + Playwright E2E test. |
| **AC-05** | Toggling overlays (`Crosshair`, `Coordinate Grid`, `Border Lines`, `Labels`) immediately shows/hides the corresponding visual overlay on the map and persists in `localStorage`. | Vitest + Playwright E2E test. |
| **AC-06** | Changing units in the Settings drawer (e.g., `°C` ↔ `°F`, `km` ↔ `miles`, `DMS` ↔ `Decimal`) immediately updates the coordinate readout, scale bar, and active legend labels, and persists across page reload. | Vitest unit/integration test + Playwright reload test. |
| **AC-07** | Simulating a failed tile or geocoding request (`500` / network abort) displays an actionable `"Unable to load / Try Again"` error state without crashing the UI. | Playwright network interception test. |
| **AC-08** | `npm run typecheck`, `npm run lint`, `npm run test`, and `npm run build` all pass with zero errors. | CI / local CLI verification. |
