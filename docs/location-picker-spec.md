# Feature Spec: Configurable Location via URL + Searchable Picker

## Summary
Add a location picker to the calendar page so users can search for a place, select it, and have the selected location fully represented in URL query params. This makes views shareable/bookmarkable and keeps state across refresh/navigation.

## Goals
1. Users can change location from the page with a searchable dropdown.
2. Selected location persists in URL params.
3. Page load reads location from URL params and renders data for that location.
4. Month navigation preserves selected location params.
5. URLs are shareable and deterministic.

## Non-goals (v1)
- Browser geolocation permission flow ("use my current location").
- Multi-location compare view.
- Reverse geocoding from map click.
- Persistence outside URL (localStorage/cookies).

---

## UX Requirements

### Placement
- Add location control near the month nav/title area, above the calendar grid.
- Keep layout mobile-friendly (single column stack on small screens).

### Control behavior
- Search input with dropdown results.
- Results appear after user types at least 2 characters.
- Debounce API calls (~250ms).
- Keyboard support:
  - `ArrowUp/ArrowDown` move selection
  - `Enter` select
  - `Escape` close dropdown
- Clicking a result applies location immediately and refreshes data.

### Display format
- Selected location label shown in input:
  - `City, Admin, Country` when available.
- If URL has no location params, default to Victoria, BC (current behavior).

### Errors/empty states
- If search returns no results: show "No locations found".
- If location params are invalid: fallback to default location and normalize URL.
- If weather API fails for selected location: show existing error message area.

---

## URL State Design

### Existing params
- `month` (1-12)
- `year` (4-digit)

### New params (v1)
- `lat` (decimal)
- `lon` (decimal)
- `tz` (IANA timezone, URL-encoded, e.g. `America/Vancouver`)
- `name` (human label, URL-encoded)

### Example URL
`?year=2026&month=3&lat=48.4284&lon=-123.3656&tz=America%2FVancouver&name=Victoria%2C%20British%20Columbia%2C%20Canada`

### Validation rules
- `lat` in [-90, 90]
- `lon` in [-180, 180]
- `tz` non-empty string
- `name` non-empty string
- If any required location field missing/invalid, fallback to default location object.

### Normalization
On load, if fallback is used, update URL params to canonical values without adding history entries (`history.replaceState`).

---

## Data Source for Search

## API choice
Use Open-Meteo Geocoding API (keeps dependency aligned with existing weather provider):

`https://geocoding-api.open-meteo.com/v1/search?name=<query>&count=8&language=en&format=json`

### Result mapping
From each geocoding result, map to:
- `name`: `name, admin1, country` (best available)
- `lat`: `latitude`
- `lon`: `longitude`
- `tz`: `timezone` (fallback to `GMT` only if missing)

### Ranking
Use API result ordering as-is for v1.

---

## App State Model

```ts
type LocationState = {
  name: string;
  lat: number;
  lon: number;
  tz: string;
};

const DEFAULT_LOCATION: LocationState = {
  name: "Victoria, British Columbia, Canada",
  lat: 48.4284,
  lon: -123.3656,
  tz: "America/Vancouver"
};
```

State sources:
1. Initial: parse URL -> validated location or default.
2. User select from search: overwrite state + URL.
3. Month navigation: keep location unchanged.

---

## Rendering + Fetching Changes

Current code hardcodes:
- `lat`, `lon`, `timezone`

Replace with dynamic location state:
- All archive and forecast API URLs must use selected `lat/lon/tz`.
- `getTodayInTimezoneString()` must use selected `tz`.
- Month title remains unchanged format (`sunlight in feb 2026`).
- Add location subtitle (optional but recommended): selected location name under month title.

---

## Navigation Behavior

### Previous/Next month buttons
When updating month/year in URL, preserve `lat/lon/tz/name` params.

### Selection behavior
On location select:
1. Update URL params (`lat/lon/tz/name`) while preserving `month/year`.
2. Trigger `loadCalendar()` using new location.
3. Prefer `history.pushState` for explicit user selection.

### Back/forward browser buttons
- Listen to `popstate`.
- Re-parse URL and re-render with URL-derived state.

---

## Accessibility
- Input has visible label: `Location`.
- Dropdown uses ARIA roles:
  - input: `role="combobox"`, `aria-expanded`, `aria-controls`
  - list: `role="listbox"`
  - item: `role="option"`, `aria-selected`
- Maintain sufficient color contrast.

---

## Mobile/Responsive Requirements
- Location input full-width on small screens.
- Dropdown max height with scroll.
- Prevent overlap with month nav controls.

---

## Security/Abuse Considerations
- Sanitize displayed `name` using textContent only (no HTML injection).
- Rate-limit client requests via debounce.
- Cap results count (8-10).

---

## Acceptance Criteria
1. Opening page without location params uses Victoria defaults and renders calendar.
2. Typing `tor` shows search results; selecting Toronto updates URL with `lat/lon/tz/name`.
3. Refresh preserves selected location and month.
4. Shared URL opens same location/month on another device.
5. Prev/next month changes only month/year params; location stays unchanged.
6. Forecast/archive calls use selected location and timezone.
7. Mobile layout remains readable without overlapping text.

---

## Implementation Plan
1. **Refactor URL helpers**
   - Add `getSelectedLocationFromUrl()` + validation.
   - Add `updateUrlParams(partial)` helper preserving existing params.
2. **Add UI markup/CSS**
   - Label + input + dropdown container.
   - Responsive styles.
3. **Add geocoding module**
   - Debounced fetch, map results, keyboard interactions.
4. **Wire calendar fetches to location state**
   - Replace hardcoded `lat/lon/timezone` usage.
5. **State sync**
   - On selection update URL + re-render.
   - Handle `popstate`.
6. **QA**
   - Desktop/mobile checks.
   - URL edge cases.

---

## Open Questions
1. Should we include a country filter (e.g., prioritize Canada first) in v1?
2. Do we want compact URL keys (`la`,`lo`,`tz`,`n`) or readable keys (`lat`,`lon`,`tz`,`name`)?
3. Should selecting a location reset month/year to current month, or preserve current month (recommended: preserve)?

---

## Suggested Default Decisions
- Preserve current `month/year` when changing location.
- Use readable URL keys.
- Keep Victoria as fallback default.
