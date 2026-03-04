# Football Teams Explorer — Code Walkthrough
# Пошаговый разбор кода / Line-by-line code companion

> Open this side-by-side with VS Code. Each section = one file.
> Откройте рядом с VS Code. Каждая секция = один файл.

---

## Table of Contents / Содержание

| # | File | What it does / Что делает |
|---|------|---------------------------|
| 0 | [Project Config](#0-project-config) | package.json, app.json, tsconfig.json |
| 1 | [Types](#1-types--srctypesindexts) | TypeScript interfaces |
| 2 | [API Layer](#2-api--srcapiteamsts) | Data fetching |
| 3 | [Geocoding API](#3-geocoding--srcapigeocodingts) | Stadium coordinates + caching |
| 4 | [Utils](#4-utils--srcutilsdistancets--formatts) | Haversine formula + formatting |
| 5 | [useDebounce](#5-usedebounce--srchooksusedebouncets) | Generic debounce hook |
| 6 | [useTeams](#6-useteams--srchooksuseteamsts) | Core data hook |
| 7 | [useLocation](#7-uselocation--srchooksuselocationts) | GPS permissions + coordinates |
| 8 | [useStadiumDistance](#8-usestadiumdistance--srchooksusestadiumdistancets) | On-demand distance calculation |
| 9 | [SearchBar](#9-searchbar--srccomponentssearchbartsx) | Text input component |
| 10 | [FilterControls](#10-filtercontrols--srccomponentsfiltercontrolstsx) | Expandable chip filters |
| 11 | [TeamCard](#11-teamcard--srccomponentsteamcardtsx) | Memoized card component |
| 12 | [DistanceButton](#12-distancebutton--srccomponentsdistancebuttontsx) | 3-state distance UI |
| 13 | [EmptyState + ErrorState](#13-emptystate--errorstate) | Edge case screens |
| 14 | [Root Layout](#14-root-layout--app_layouttsx) | Navigation setup |
| 15 | [Main Screen](#15-main-screen--appindextsx) | Everything wired together |

---

## 0. Project Config

### package.json

```
"main": "expo-router/entry"     ← Entry point. Expo Router takes over from here.
                                   Swift: like @main App {} but router-based.

"expo": "~52.0.40"              ← Expo SDK 52 = build tools + native modules pre-configured
"react": "18.3.1"               ← React 18 = hooks, concurrent features
"react-native": "0.76.7"        ← RN 0.76 = New Architecture (JSI, Fabric, TurboModules)
"expo-router": "~4.0.19"        ← File-based routing (like Next.js, NOT like UINavigationController)
"expo-location": "~18.0.10"     ← Native GPS module (wraps CLLocationManager / FusedLocationProvider)
"react-native-reanimated": ...  ← Native-thread animations (like Core Animation but JS-driven)
```

**Swift parallel / Аналогия со Swift:**
- `expo` = Xcode project + CocoaPods + build config all-in-one
- `expo-router` = Coordinator pattern, but file-system based
- `expo-location` = CLLocationManager wrapped as a JS module

### app.json — lines that matter:

```jsonc
"newArchEnabled": true           // ← Enables JSI (no more bridge serialization!)
                                 // Swift: like switching from ObjC message passing to direct C++ calls

"NSLocationWhenInUseUsageDescription": "..."  // ← Same as Info.plist key. Expo injects it at build time.

"experiments": { "typedRoutes": true }        // ← Route params get TypeScript types. Catches broken links at compile time.
```

### tsconfig.json

```jsonc
"strict": true                   // ← Full type checking (like Swift's type system strictness)
"paths": { "@/*": ["./*"] }      // ← Import alias: @/src/hooks/useTeams instead of ../../../src/hooks/useTeams
                                 //   Swift: no equivalent needed because Xcode resolves by module name
```

---

## 1. Types — `src/types/index.ts`

> Open: `src/types/index.ts`

This file defines the shape of all data in the app. In Swift this would be `struct` + `Codable`.

```typescript
// LINE 1-28: FootballTeam interface
export interface FootballTeam {
  name: string;                           // Swift: let name: String
  captain: string;
  // ... 26 more fields
  estimated_value_numeric: number;        // Used for filtering/sorting
  number_of_league_titles_won: number;    // Used for filtering
  stadium_name: string;                   // Used for geocoding
  stadium_capacity: number;               // Displayed in card
}
```

**Key difference from Swift:**
- `interface` = compile-time only. Erased at runtime. No `Codable` conformance needed.
- `number` = both Int and Double. No Int/Double distinction in JS.
- `number | null` = Swift's `Optional<Int>` / `Int?`

```typescript
// LINE 30-36: API response shape — matches the JSON structure exactly
export interface TeamsApiResponse {
  page: number;          // current page number
  per_page: number;      // items per page
  total: number;         // total item count
  total_pages: number;   // how many pages exist ← we use this to know how many parallel fetches to make
  data: FootballTeam[];  // the actual teams array
}

// LINE 38-42: Geocoding result from Nominatim API
export interface NominatimResult {
  lat: string;           // Note: string not number! API returns strings, we parseFloat later.
  lon: string;
}

// LINE 44-48: Filter state shape
export interface Filters {
  search: string;
  minValue: number | null;    // null = "any" (no filter active)
  minTitles: number | null;
}

// LINE 50-53: GPS coordinates
export interface Location {
  latitude: number;
  longitude: number;
}
```

**Why separate interfaces?**
Same principle as Swift — each type has one responsibility. `Filters` is separate from `FootballTeam` because it represents UI state, not API data.

---

## 2. API — `src/api/teams.ts`

> Open: `src/api/teams.ts`

This is the network layer. Fetches all football teams from a paginated REST API.

```typescript
// LINE 1: Import types for type safety
import { TeamsApiResponse, FootballTeam } from '../types';

// LINE 3: Base URL constant
const BASE_URL = 'https://jsonmock.hackerrank.com/api/football_teams';
// HackerRank's mock API. Paginated, 10 teams per page.
```

```typescript
// LINE 5: The function signature
export async function fetchAllTeams(): Promise<FootballTeam[]> {
//     ^^^^^ = available to other files
//          ^^^^^ = returns a Promise (Swift: async throws -> [FootballTeam])
```

**Step 1 — Fetch page 1 to learn total pages:**
```typescript
  // LINE 7-8: fetch() is JS's built-in URLSession equivalent
  const firstResponse = await fetch(BASE_URL);
  // Swift equivalent: let (data, response) = try await URLSession.shared.data(from: url)

  if (!firstResponse.ok) {
    throw new Error(`API error: ${firstResponse.status}`);
    // Swift: throw URLError(.badServerResponse)
  }

  // LINE 12-13: Parse JSON + type it
  const firstPage: TeamsApiResponse = await firstResponse.json();
  // Swift: let firstPage = try JSONDecoder().decode(TeamsApiResponse.self, from: data)
  // KEY DIFFERENCE: No Codable needed. .json() returns `any`, TypeScript annotation adds compile-time checks only.

  const allTeams: FootballTeam[] = [...firstPage.data];
  // Spread operator (...) = copies array. Swift: var allTeams = firstPage.data
```

**Step 2 — Fetch remaining pages IN PARALLEL:**
```typescript
  // LINE 16-20: Build array [2, 3, 4, ...totalPages]
  if (firstPage.total_pages > 1) {
    const pageNumbers = Array.from(
      { length: firstPage.total_pages - 1 },   // e.g., 3 elements for 4 total pages
      (_, i) => i + 2                            // maps to [2, 3, 4]
    );
    // Swift equivalent: let pageNumbers = Array(2...firstPage.totalPages)
```

```typescript
    // LINE 22-31: Promise.all = fire ALL requests simultaneously, wait for ALL to finish
    const responses = await Promise.all(
      pageNumbers.map(async (page) => {
        const response = await fetch(`${BASE_URL}?page=${page}`);
        if (!response.ok) {
          throw new Error(`API error on page ${page}: ${response.status}`);
        }
        const data: TeamsApiResponse = await response.json();
        return data.data;           // return just the teams array, not the wrapper
      })
    );
    // Swift equivalent: try await withThrowingTaskGroup(of: [FootballTeam].self) { group in
    //     for page in pageNumbers { group.addTask { try await fetchPage(page) } }
    //     ...
    // }
```

```typescript
    // LINE 33: Flatten results into single array
    responses.forEach((teams) => allTeams.push(...teams));
    // Swift: responses.forEach { allTeams.append(contentsOf: $0) }
  }

  return allTeams;  // All teams from all pages, in one array
```

**Why this pattern matters for the interview:**
- Shows you understand pagination
- Shows you can optimize with parallel fetching (not sequential!)
- `Promise.all` is the #1 way to do concurrent async work in JS (like TaskGroup in Swift)

---

## 3. Geocoding — `src/api/geocoding.ts`

> Open: `src/api/geocoding.ts`

Converts stadium names to GPS coordinates using OpenStreetMap's free Nominatim API.

```typescript
// LINE 3: API endpoint
const NOMINATIM_URL = 'https://nominatim.openstreetmap.org/search';

// LINE 6: In-memory cache — a JS Map (Swift: Dictionary / NSCache)
const geocodeCache = new Map<string, { lat: number; lon: number } | null>();
// Key: "Camp Nou" → Value: { lat: 41.38, lon: 2.12 } or null if not found
// This is MODULE-LEVEL — survives across renders, shared by all components.
// Swift equivalent: private static var cache: [String: CLLocationCoordinate2D?] = [:]
```

```typescript
// LINE 8-10: Function signature
export async function geocodeStadium(
  stadiumName: string
): Promise<{ lat: number; lon: number } | null> {
  // Returns coordinates or null if stadium can't be found

  // LINE 11-13: Cache check — O(1) lookup
  if (geocodeCache.has(stadiumName)) {
    return geocodeCache.get(stadiumName) ?? null;
  }
  // Swift: if let cached = cache[stadiumName] { return cached }
```

```typescript
  // LINE 15-23: Build URL and fetch
  const query = encodeURIComponent(`${stadiumName} stadium`);
  // encodeURIComponent = Swift's addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)
  // We append " stadium" to improve geocoding accuracy

  const response = await fetch(
    `${NOMINATIM_URL}?q=${query}&format=json&limit=1`,
    {
      headers: {
        'User-Agent': 'OnwardClubsList/1.0 (oleh@veheria.tech)',
        // Nominatim REQUIRES a User-Agent. Without it, requests get 403'd.
        // This is API etiquette, like setting URLRequest.setValue for User-Agent
      },
    }
  );
```

```typescript
  // LINE 29-33: Parse results
  const results: NominatimResult[] = await response.json();

  if (results.length === 0) {
    geocodeCache.set(stadiumName, null);  // Cache the "not found" too! Prevents repeated failed requests.
    return null;
  }

  // LINE 36-42: Extract coordinates, cache, return
  const coords = {
    lat: parseFloat(results[0].lat),   // Remember: API returns strings, we need numbers
    lon: parseFloat(results[0].lon),
  };

  geocodeCache.set(stadiumName, coords);  // Cache for next time
  return coords;
```

**Interview talking point:** "I cache both successful AND failed lookups to avoid hammering the API. The cache lives at module scope so it persists across re-renders without needing React state."

---

## 4. Utils — `src/utils/distance.ts` + `format.ts`

> Open: `src/utils/distance.ts`

### Haversine Formula — distance between two GPS points on a sphere

```typescript
// LINE 5-24: Pure math function, no React involved
export function haversineDistance(
  lat1: number, lon1: number,    // User's location
  lat2: number, lon2: number     // Stadium's location
): number {
  const R = 6371;                 // Earth's radius in km (known constant)
  const dLat = toRadians(lat2 - lat1);   // Delta latitude in radians
  const dLon = toRadians(lon2 - lon1);   // Delta longitude in radians

  // The Haversine formula accounts for Earth's curvature
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRadians(lat1)) *
      Math.cos(toRadians(lat2)) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);

  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;    // Result in kilometers
}
// Swift: CLLocation.distance(from:) does this for you, but knowing the math shows depth.
```

```typescript
// LINE 33-41: Human-readable formatting
export function formatDistance(km: number): string {
  if (km < 1)   return `${Math.round(km * 1000)} m`;     // 0.5 → "500 m"
  if (km < 10)  return `${km.toFixed(1)} km`;             // 3.7 → "3.7 km"
  return `${Math.round(km).toLocaleString()} km`;         // 1234 → "1,234 km"
}
// Template literals (`${}`) = Swift's string interpolation ("\(value)")
// toLocaleString() adds thousand separators based on user's locale
```

> Open: `src/utils/format.ts`

```typescript
// LINE 5-16: Currency formatting
export function formatValue(valueNumeric: number): string {
  if (valueNumeric >= 1_000_000_000) {
    return `$${(valueNumeric / 1_000_000_000).toFixed(1)}B`;   // 2000000000 → "$2.0B"
  }
  if (valueNumeric >= 1_000_000) {
    return `$${Math.round(valueNumeric / 1_000_000)}M`;        // 200000000 → "$200M"
  }
  // ... K and raw number cases
}
// Note: 1_000_000_000 = numeric separator (same as Swift's 1_000_000_000)
// .toFixed(1) = one decimal place. Math.round = no decimals.
```

---

## 5. useDebounce — `src/hooks/useDebounce.ts`

> Open: `src/hooks/useDebounce.ts`

This is a **custom hook** — a reusable piece of stateful logic. Swift has no direct equivalent; closest is a Combine pipeline.

```typescript
// LINE 7: Generic hook — works with any type T
export function useDebounce<T>(value: T, delayMs: number = 300): T {
//                          ^^ TypeScript generic (Swift: <T>)
//                                        ^^^^^^^^^^^^^^^^^^^ default parameter

  // LINE 8: Internal state holds the "delayed" value
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  // useState returns [currentValue, setterFunction]
  // Swift: @State private var debouncedValue: T = value
```

```typescript
  // LINE 10-16: The debounce logic
  useEffect(() => {
    // This runs every time `value` or `delayMs` changes

    const timer = setTimeout(() => {
      setDebouncedValue(value);     // After 300ms of NO changes, update the debounced value
    }, delayMs);

    return () => clearTimeout(timer);
    // ^^^^^^ CLEANUP FUNCTION — runs BEFORE next effect, or on unmount
    // If user types again within 300ms, the old timer is cancelled and a new one starts.
    // Swift equivalent: Combine's .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
  }, [value, delayMs]);
  //  ^^^^^^^^^^^^^^^^ dependency array — only re-run when these change

  return debouncedValue;
}
```

**How it works step by step:**
1. User types "R" → `value` = "R" → timer starts (300ms)
2. User types "e" (50ms later) → cleanup cancels old timer → new timer starts (300ms)
3. User types "a" (50ms later) → cleanup cancels → new timer (300ms)
4. User stops typing → 300ms passes → `setDebouncedValue("Rea")` fires
5. Component re-renders with debounced value "Rea" → search executes

**Why not just search on every keystroke?**
Each search triggers `useMemo` recalculation on the full teams array. With debounce, you recalculate once after the user stops typing, not on every keystroke. Saves CPU cycles + prevents UI jank.

---

## 6. useTeams — `src/hooks/useTeams.ts`

> Open: `src/hooks/useTeams.ts`

This is the **core hook** — manages data loading, filtering, and sorting. It's what interviewers will ask about most.

```typescript
// LINE 5-16: Return type interface — explicitly typed for consumers
interface UseTeamsReturn {
  filteredTeams: FootballTeam[];            // The filtered + sorted list to display
  isLoading: boolean;                       // Show spinner?
  error: string | null;                     // Show error screen?
  filters: Filters;                         // Current filter state
  hasActiveFilters: boolean;                // Any filters active?
  setSearch: (search: string) => void;      // Filter setters
  setMinValue: (value: number | null) => void;
  setMinTitles: (titles: number | null) => void;
  resetFilters: () => void;
  retry: () => void;                        // Re-fetch after error
}
// Swift parallel: This is like a ViewModel's public interface
// protocol TeamsViewModelProtocol { var filteredTeams: [FootballTeam] { get } ... }
```

```typescript
// LINE 18-22: Default filter state (module-level constant)
const INITIAL_FILTERS: Filters = {
  search: '',
  minValue: null,      // null = "Any" (no filter)
  minTitles: null,
};
```

### State declarations:

```typescript
// LINE 24-28: Four pieces of state
export function useTeams(): UseTeamsReturn {
  const [teams, setTeams] = useState<FootballTeam[]>([]);      // Raw API data
  const [isLoading, setIsLoading] = useState(true);             // Start loading immediately
  const [error, setError] = useState<string | null>(null);      // null = no error
  const [filters, setFilters] = useState<Filters>(INITIAL_FILTERS);
  // Swift: 4x @State or @Published properties
```

### Data loading:

```typescript
  // LINE 30-44: loadTeams wrapped in useCallback
  const loadTeams = useCallback(async () => {
    // useCallback = memoize this function so it doesn't get recreated on every render
    // Swift: no equivalent needed because closures don't cause re-renders in SwiftUI

    setIsLoading(true);
    setError(null);          // Clear previous error

    try {
      const allTeams = await fetchAllTeams();    // Call API layer
      setTeams(allTeams);                        // Store in state
    } catch (err) {
      const message =
        err instanceof Error ? err.message : 'Failed to load teams';
        // instanceof = Swift's `if let error = err as? Error`
      setError(message);
    } finally {
      setIsLoading(false);   // Always stop loading, success or fail
      // Swift: defer { isLoading = false }
    }
  }, []);
  //   ^^ empty dependency array = function never changes = created once
```

### Trigger loading on mount:

```typescript
  // LINE 46-48: useEffect with [loadTeams] dependency
  useEffect(() => {
    loadTeams();
  }, [loadTeams]);
  // Runs once on mount because loadTeams never changes (empty deps)
  // Swift equivalent: .onAppear { await loadTeams() }
  // BUT: useEffect runs AFTER first render, not before (like viewDidAppear, not viewWillAppear)
```

### Filtering + Sorting (the most important part):

```typescript
  // LINE 50-82: useMemo = computed property that caches its result
  const filteredTeams = useMemo(() => {
    // This function ONLY re-runs when `teams` or `filters` change.
    // If neither changed, returns cached result. No computation.
    // Swift: like a computed property but with automatic caching

    let result = [...teams];    // Copy array (never mutate original state!)

    // Filter by search term
    if (filters.search.trim()) {
      const searchLower = filters.search.toLowerCase().trim();
      result = result.filter((team) =>
        team.name.toLowerCase().includes(searchLower)
      );
      // Swift: result = result.filter { $0.name.localizedCaseInsensitiveContains(searchLower) }
    }

    // Filter by minimum estimated value
    if (filters.minValue !== null) {
      result = result.filter(
        (team) => team.estimated_value_numeric >= (filters.minValue ?? 0)
      );
    }

    // Filter by minimum league titles
    if (filters.minTitles !== null) {
      result = result.filter(
        (team) => team.number_of_league_titles_won >= (filters.minTitles ?? 0)
      );
    }

    // Sort by valuation (highest first)
    result.sort(
      (a, b) => b.estimated_value_numeric - a.estimated_value_numeric
    );
    // Swift: result.sort { $0.estimatedValueNumeric > $1.estimatedValueNumeric }
    // NOTE: JS sort is in-place AND returns the array. Negative = a first, positive = b first.

    return result;
  }, [teams, filters]);
  //  ^^^^^^^^^^^^^^^^ dependency array — recalculate when either changes
```

### Filter setters (stable references):

```typescript
  // LINE 84-98: Each setter wrapped in useCallback
  const setSearch = useCallback((search: string) => {
    setFilters((prev) => ({ ...prev, search }));
    //         ^^^^^^ functional update — receives previous state, returns new state
    //                    ^^^^^^^^^^^ spread operator: copy all fields, override `search`
    // Swift: filters.search = search (but in React, state is immutable!)
  }, []);

  const setMinValue = useCallback((value: number | null) => {
    setFilters((prev) => ({ ...prev, minValue: value }));
  }, []);

  // ... setMinTitles, resetFilters same pattern

  // LINE 100-103: Derived boolean (no need for useMemo — cheap comparison)
  const hasActiveFilters =
    filters.search !== '' ||
    filters.minValue !== null ||
    filters.minTitles !== null;
```

### Return object:

```typescript
  // LINE 105-116
  return {
    filteredTeams,     // The pre-filtered, pre-sorted list
    isLoading,
    error,
    filters,
    hasActiveFilters,
    setSearch,         // Stable callback references
    setMinValue,
    setMinTitles,
    resetFilters,
    retry: loadTeams,  // Expose loadTeams as "retry"
  };
```

**Interview talking point:** "useTeams is essentially a ViewModel. It owns the data, exposes filtered/sorted output via useMemo (like a computed property), and provides stable setter callbacks that child components can call without causing unnecessary re-renders."

---

## 7. useLocation — `src/hooks/useLocation.ts`

> Open: `src/hooks/useLocation.ts`

Wraps `expo-location` to request GPS permission and get coordinates.

```typescript
// LINE 2: Import the native module
import * as ExpoLocation from 'expo-location';
// This bridges to CLLocationManager (iOS) / FusedLocationProviderClient (Android)
```

```typescript
// LINE 21-63: The effect
useEffect(() => {
  let cancelled = false;   // ← Cleanup flag to prevent state updates after unmount
  // Swift: like a Task { } with Task.isCancelled checks

  async function requestLocation() {
    setIsLoading(true);

    // LINE 27: Request permission (shows native dialog)
    const { status } = await ExpoLocation.requestForegroundPermissionsAsync();
    // Swift: CLLocationManager().requestWhenInUseAuthorization()
    // Returns 'granted', 'denied', or 'undetermined'

    if (status !== 'granted') {
      if (!cancelled) {                    // Only update state if component still mounted
        setError('Location permission denied');
        setIsLoading(false);
      }
      return;
    }

    try {
      // LINE 37-39: Get current position
      const position = await ExpoLocation.getCurrentPositionAsync({
        accuracy: ExpoLocation.Accuracy.Balanced,
        // Balanced = GPS + Wi-Fi + cell towers. Good accuracy, reasonable battery usage.
        // Swift: CLLocationAccuracy.kCLLocationAccuracyHundredMeters (similar)
      });

      if (!cancelled) {
        setLocation({
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
        });
      }
    } catch (err) {
      if (!cancelled) {
        setError(err instanceof Error ? err.message : 'Failed to get location');
      }
    }
  }

  requestLocation();    // Fire immediately on mount

  // LINE 60-62: Cleanup function
  return () => {
    cancelled = true;    // If component unmounts mid-request, don't update state
    // This prevents the React warning: "Can't perform a React state update on an unmounted component"
    // Swift equivalent: task.cancel() in .onDisappear
  };
}, []);
//  ^^ empty deps = run once on mount, clean up on unmount
```

**Why `cancelled` flag instead of AbortController?**
`expo-location` doesn't support AbortController. The `cancelled` boolean is the standard React pattern for non-abortable async operations. You'll see this pattern in many React Native codebases.

---

## 8. useStadiumDistance — `src/hooks/useStadiumDistance.ts`

> Open: `src/hooks/useStadiumDistance.ts`

Calculates distance to a stadium **on demand** (user taps button), not automatically.

```typescript
// LINE 20: useRef for mounted tracking (persists across renders without causing re-renders)
const mountedRef = useRef(true);
// useRef = like a stored property that doesn't trigger re-renders when changed
// Swift: no exact equivalent. Closest is a regular property on a class-based ViewModel

// LINE 22-26: Track mount status
useEffect(() => {
  return () => {
    mountedRef.current = false;   // Set to false on unmount
  };
}, []);
```

```typescript
// LINE 28-63: The calculate function — triggered by user tap
const calculate = useCallback(async () => {
  if (!userLocation) {
    setError('Location not available');
    return;
  }

  setIsLoading(true);
  setError(null);

  try {
    // Step 1: Geocode stadium name → coordinates
    const coords = await geocodeStadium(stadiumName);
    // This call may hit cache (instant) or network (slow)

    if (!mountedRef.current) return;    // Component unmounted during await

    if (!coords) {
      setError('Stadium not found');
      return;
    }

    // Step 2: Calculate distance using Haversine
    const km = haversineDistance(
      userLocation.latitude,
      userLocation.longitude,
      coords.lat,
      coords.lon
    );

    setDistance(formatDistance(km));     // "1,234 km"
  } catch {
    if (!mountedRef.current) return;
    setError('Failed to calculate distance');
  } finally {
    if (mountedRef.current) {
      setIsLoading(false);
    }
  }
}, [stadiumName, userLocation]);
// Dependency array: recreate function if stadium or user location changes
```

**Why on-demand (lazy) instead of automatic?**
- 24+ teams × geocoding API call = 24 network requests on load
- Nominatim has rate limits (1 req/sec for free tier)
- User may not care about distance for most teams
- This is a **battery-conscious** choice — only fetch when needed

**Swift parallel:** Like a `@MainActor func calculateDistance()` method on a ViewModel that you call from a button's action.

---

## 9. SearchBar — `src/components/SearchBar.tsx`

> Open: `src/components/SearchBar.tsx`

A **controlled input** — the parent owns the state, this component just renders it.

```typescript
// LINE 10-14: Props interface (Swift: init parameters)
interface SearchBarProps {
  value: string;                        // Current text (owned by parent)
  onChangeText: (text: string) => void; // Callback when user types (Swift: @Binding)
  placeholder?: string;                 // Optional with default
}
```

```typescript
// LINE 16-20: Destructuring props in function signature
export function SearchBar({
  value,
  onChangeText,
  placeholder = 'Search teams...',     // Default value if not provided
}: SearchBarProps) {
// Swift equivalent:
// struct SearchBar: View {
//     @Binding var text: String
//     var placeholder: String = "Search teams..."
// }
```

```typescript
// LINE 24-35: TextInput — RN's equivalent of UITextField
<TextInput
  style={styles.input}
  value={value}                        // Controlled: React owns the value
  onChangeText={onChangeText}          // Fires on every keystroke
  placeholder={placeholder}
  placeholderTextColor="#999"
  autoCapitalize="none"                // Don't capitalize search terms
  autoCorrect={false}                  // Don't autocorrect team names
  returnKeyType="search"               // Show "Search" on keyboard
  accessibilityLabel="Search teams by name"        // VoiceOver
  accessibilityHint="Type a team name to filter"   // VoiceOver hint
/>
// Swift: TextField("Search teams...", text: $text)
//   .textInputAutocapitalization(.never)
//   .autocorrectionDisabled()
```

```typescript
// LINE 36-44: Conditional clear button
{value.length > 0 && (          // Only show when there's text
  <TouchableOpacity
    onPress={() => onChangeText('')}     // Clear by setting value to empty string
    hitSlop={{ top: 12, bottom: 12, left: 12, right: 12 }}
    // hitSlop = expand tap area beyond visible bounds. Like override point(inside:with:) in UIKit.
    // Makes small buttons easier to tap (Apple HIG: 44pt minimum)
  >
    <Text style={styles.clearIcon}>✕</Text>
  </TouchableOpacity>
)}
```

---

## 10. FilterControls — `src/components/FilterControls.tsx`

> Open: `src/components/FilterControls.tsx`

Expandable filter panel with preset chips.

```typescript
// LINE 18-32: Preset arrays — defined OUTSIDE the component (module-level)
const VALUE_PRESETS = [
  { label: 'Any', value: null },           // null = no filter
  { label: '$100M+', value: 100_000_000 },
  { label: '$500M+', value: 500_000_000 },
  { label: '$1B+', value: 1_000_000_000 },
  { label: '$2B+', value: 2_000_000_000 },
];
// WHY outside component? These arrays never change. If inside, they'd be recreated every render.
// Swift: static let constants on the struct
```

```typescript
// LINE 42: Local state for expand/collapse — only this component cares about it
const [isExpanded, setIsExpanded] = useState(false);
// This is NOT in useTeams because it's purely visual state, not data state.
// Principle: keep state as close to where it's used as possible.
```

```typescript
// LINE 63-84: Rendering chips with .map() — React's version of ForEach
{VALUE_PRESETS.map((preset) => (
  <TouchableOpacity
    key={preset.label}                    // ← REQUIRED: unique key for list rendering
    // Swift ForEach needs `id:` — same concept. React uses it to track which items changed.
    style={[
      styles.chip,
      minValue === preset.value && styles.chipActive,
      // Conditional styling: if this preset is active, add blue background
      // Swift: .background(isActive ? Color.blue : Color.gray)
    ]}
    onPress={() => onMinValueChange(preset.value)}
    accessibilityState={{ selected: minValue === preset.value }}
    // ^^^ Tells VoiceOver this chip is currently selected
  >
    <Text style={[styles.chipText, minValue === preset.value && styles.chipTextActive]}>
      {preset.label}
    </Text>
  </TouchableOpacity>
))}
```

---

## 11. TeamCard — `src/components/TeamCard.tsx`

> Open: `src/components/TeamCard.tsx`

The most important component to understand. Uses **React.memo** for performance.

```typescript
// LINE 1: Import memo from React
import React, { memo } from 'react';
```

```typescript
// LINE 7-11: Props interface
interface TeamCardProps {
  team: FootballTeam;
  userLocation: Location | null;  // Passed through to DistanceButton
  rank: number;                   // Position in the sorted list
}

// LINE 13: Define as regular function first
function TeamCardComponent({ team, userLocation, rank }: TeamCardProps) {
```

```typescript
// LINE 15-19: Root View with accessibility
<View
  style={styles.card}
  accessibilityLabel={`${team.name}, valued at ${team.estimated_value}, ${team.number_of_league_titles_won} league titles`}
  accessibilityRole="summary"
  // VoiceOver reads: "Real Madrid, valued at $6.07B, 36 league titles"
  // accessibilityRole="summary" = groups child elements for VoiceOver
>
```

```typescript
// LINE 32: Using utility function in JSX
<Text style={styles.value}>{formatValue(team.estimated_value_numeric)}</Text>
// Calls formatValue(6070000000) → "$6.1B"
```

```typescript
// LINE 60-63: DistanceButton receives stadium name + user location
<DistanceButton
  stadiumName={team.stadium_name}
  userLocation={userLocation}
/>
// Each card has its own DistanceButton with its own useStadiumDistance hook
// = each button manages its own loading/distance/error state independently
```

```typescript
// LINE 69: THE KEY LINE — memo() wrapping
export const TeamCard = memo(TeamCardComponent);
// memo = "only re-render if props changed" (shallow comparison)
// Without memo: FlatList re-renders ALL visible cards when ANY state changes
// With memo: only cards whose `team`, `userLocation`, or `rank` changed re-render
//
// Swift equivalent: Equatable conformance on SwiftUI View
// struct TeamCard: View, Equatable {
//     static func == (lhs: TeamCard, rhs: TeamCard) -> Bool { ... }
// }
```

**Why memo matters here:** FlatList renders ~10 visible cards. When user types in search, `filteredTeams` changes → FlatList re-renders → without memo, all 10 cards re-render even if their specific team data didn't change. With memo, only cards that actually received different props re-render.

---

## 12. DistanceButton — `src/components/DistanceButton.tsx`

> Open: `src/components/DistanceButton.tsx`

A **3-state component**: default → loading → result/error. Each state renders completely different UI.

```typescript
// LINE 21-24: Use the custom hook
const { distance, isLoading, error, calculate } = useStadiumDistance(
  stadiumName,
  userLocation
);
// The hook manages all async state. This component just renders based on that state.
```

```typescript
// LINE 26-37: STATE 1 — Result exists → show green badge
if (distance) {
  return (
    <TouchableOpacity
      style={[styles.button, styles.buttonResult]}    // Green background
      onPress={calculate}                              // Tap to recalculate
      accessibilityLabel={`Distance to ${stadiumName}: ${distance}`}
    >
      <Text style={styles.resultText}>📍 {distance}</Text>
    </TouchableOpacity>
  );
}

// LINE 39-49: STATE 2 — Error → show red badge with retry
if (error) {
  return (
    <TouchableOpacity
      style={[styles.button, styles.buttonError]}
      onPress={calculate}                              // Tap to retry!
    >
      <Text style={styles.errorText}>⚠️ {error}</Text>
    </TouchableOpacity>
  );
}

// LINE 52-72: STATE 3 — Default/Loading → blue badge or spinner
return (
  <TouchableOpacity
    style={styles.button}
    onPress={calculate}
    disabled={isLoading || !userLocation}
    // Disabled when: actively loading OR location not available yet
  >
    {isLoading ? (
      <ActivityIndicator size="small" color="#1a73e8" />   // Native spinner
    ) : (
      <Text style={[styles.text, !userLocation && styles.textDisabled]}>
        📍 Distance
      </Text>
    )}
  </TouchableOpacity>
);
```

**Pattern: early returns for different states.** This is idiomatic React. Instead of one JSX tree with lots of ternaries, you return early for each state. Cleaner to read, easier to maintain.

Swift parallel: `switch viewModel.state { case .loaded: ... case .error: ... case .loading: ... }` in a SwiftUI `body`.

---

## 13. EmptyState + ErrorState

> Open: `src/components/EmptyState.tsx` + `ErrorState.tsx`

These are simple **presentational components** — no hooks, no state, just render props.

### EmptyState:
```typescript
// Smart conditional: different message depending on whether filters are active
{hasFilters
  ? 'Try adjusting your filters to see more results.'    // User filtered too aggressively
  : 'Something went wrong loading teams.'}                // No data at all

// Reset button only shows when filters are active (when it can actually help)
{hasFilters && (
  <TouchableOpacity onPress={onReset}>
```

### ErrorState:
```typescript
// Takes message + retry callback. That's it.
export function ErrorState({ message, onRetry }: ErrorStateProps) {
// Single responsibility: display error, offer retry. No business logic.
```

---

## 14. Root Layout — `app/_layout.tsx`

> Open: `app/_layout.tsx`

The **simplest file** in the project but architecturally important.

```typescript
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack
      screenOptions={{
        headerShown: false,    // We handle our own header in the screen
      }}
    />
  );
}
```

**How Expo Router works (file-based routing):**
- `app/_layout.tsx` = defines the **navigator** (Stack, Tabs, etc.)
- `app/index.tsx` = the **default screen** (like "/" in web)
- `app/team/[id].tsx` = would be a **dynamic route** (like "/team/123")

**Swift parallel:**
```swift
// This entire file is equivalent to:
NavigationStack {
    TeamsScreen()
        .navigationBarHidden(true)
}
```

**Why `headerShown: false`?** We build a custom header inside `index.tsx` instead of using the default navigation bar. More control over design.

---

## 15. Main Screen — `app/index.tsx`

> Open: `app/index.tsx`

This is where **everything comes together**. The conductor of the orchestra.

### Imports (lines 1-19):
```typescript
import React, { useState, useCallback } from 'react';
import { View, FlatList, Text, StyleSheet, ActivityIndicator, SafeAreaView, StatusBar } from 'react-native';
// SafeAreaView = respects notch/Dynamic Island (like .safeAreaInset in SwiftUI)
// StatusBar = control status bar appearance (like .preferredStatusBarStyle)

import { useTeams } from '../src/hooks/useTeams';
import { useDebounce } from '../src/hooks/useDebounce';
import { useLocation } from '../src/hooks/useLocation';
// Three hooks = three concerns: data, input optimization, GPS
// Each is independent and testable in isolation
```

### State & Hooks (lines 21-38):
```typescript
export default function TeamsScreen() {
  // LOCAL state: the raw text input value (changes on every keystroke)
  const [searchInput, setSearchInput] = useState('');

  // DERIVED state: debounced version (changes 300ms after last keystroke)
  const debouncedSearch = useDebounce(searchInput, 300);

  // DATA hook: teams, filters, loading, error — everything from the "ViewModel"
  const {
    filteredTeams, isLoading, error, filters, hasActiveFilters,
    setSearch, setMinValue, setMinTitles, resetFilters, retry,
  } = useTeams();

  // LOCATION hook: user's GPS coordinates (or null if denied/loading)
  const { location: userLocation } = useLocation();
  // Rename: `location` from hook → `userLocation` in this scope (avoids shadowing)
```

### Syncing debounced search to filter (lines 41-43):
```typescript
  React.useEffect(() => {
    setSearch(debouncedSearch);
  }, [debouncedSearch, setSearch]);
  // WHY two-step? searchInput updates on every keystroke (for responsive TextInput)
  // but setSearch (which triggers useMemo filtering) only gets the debounced value.
  // This decouples input responsiveness from filtering performance.
  //
  // Flow: keystroke → searchInput → [300ms] → debouncedSearch → setSearch → useMemo → filteredTeams
```

### Memoized callbacks (lines 45-60):
```typescript
  // handleReset: clear both local input and all filters
  const handleReset = useCallback(() => {
    setSearchInput('');         // Clear local TextInput
    resetFilters();             // Clear all filters in useTeams
  }, [resetFilters]);

  // renderItem: how to render each team in the list
  const renderItem = useCallback(
    ({ item, index }: { item: FootballTeam; index: number }) => (
      <TeamCard team={item} userLocation={userLocation} rank={index + 1} />
    ),
    [userLocation]
    // Only recreate when userLocation changes. If we didn't wrap in useCallback,
    // FlatList would think renderItem changed on every render → re-render all items.
  );

  // keyExtractor: unique key for each item (React needs this for efficient list diffing)
  const keyExtractor = useCallback(
    (item: FootballTeam) => item.name,
    []    // Never changes — team names are unique and stable
  );
  // Swift: ForEach(teams, id: \.name) { team in ... }
```

### Error early return (lines 62-68):
```typescript
  if (error) {
    return (
      <SafeAreaView style={styles.container}>
        <ErrorState message={error} onRetry={retry} />
      </SafeAreaView>
    );
  }
  // Pattern: handle error state before the main render.
  // If there's an error, we don't show the list at all — just the error screen with retry.
```

### The main render — FlatList (lines 70-118):
```typescript
  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" />
      {/* barStyle="dark-content" = dark text on light background */}

      {/* Custom header */}
      <View style={styles.header}>
        <Text style={styles.title}>Football Teams</Text>
        <Text style={styles.subtitle}>
          {isLoading ? 'Loading...' : `${filteredTeams.length} teams · sorted by value`}
          {/* Ternary: condition ? ifTrue : ifFalse */}
          {/* Template literal with dynamic count */}
        </Text>
      </View>

      {/* THE LIST — the most important component */}
      <FlatList
        data={filteredTeams}              // What to render
        renderItem={renderItem}           // How to render each item
        keyExtractor={keyExtractor}       // Unique ID per item

        contentContainerStyle={styles.list}   // Padding inside scroll area

        ListHeaderComponent={             // Rendered ABOVE the list, scrolls with it
          <>                              {/* Fragment: group without extra View */}
            <SearchBar value={searchInput} onChangeText={setSearchInput} />
            <FilterControls
              minValue={filters.minValue}
              minTitles={filters.minTitles}
              onMinValueChange={setMinValue}
              onMinTitlesChange={setMinTitles}
              onReset={handleReset}
              hasActiveFilters={hasActiveFilters}
            />
          </>
        }

        ListEmptyComponent={             // Shown when data=[] (no items)
          isLoading ? (
            <View style={styles.loadingContainer}>
              <ActivityIndicator size="large" color="#1a73e8" />
              <Text style={styles.loadingText}>Loading teams...</Text>
            </View>
          ) : (
            <EmptyState hasFilters={hasActiveFilters} onReset={handleReset} />
          )
        }

        {/* Performance tuning */}
        showsVerticalScrollIndicator={false}
        initialNumToRender={10}           // Render 10 items on first paint
        maxToRenderPerBatch={10}          // Render max 10 items per scroll frame
        windowSize={5}                    // Keep 5 "screens" of items in memory
        removeClippedSubviews={true}      // Detach off-screen items from native view hierarchy
        {/*
          Swift parallel: UITableView with estimatedRowHeight + prefetchDataSource
          These props control FlatList's virtualization engine.
          Without them: all 24 items render at once (fine for 24, bad for 10,000)
          With them: only ~10 visible + 20 buffered items exist in memory
        */}
      />
    </SafeAreaView>
  );
```

---

## Data Flow Summary / Полная схема потоков данных

```
┌──────────────────────────────────────────────────────────────────┐
│                        TeamsScreen                                │
│                                                                   │
│  searchInput ──→ useDebounce ──→ debouncedSearch ──→ setSearch   │
│       ↑                                                  ↓        │
│  SearchBar                                          useTeams      │
│                                                     ├─ teams[]   │
│  FilterControls ──→ setMinValue/setMinTitles ──→    ├─ filters   │
│                                                     ├─ useMemo ──→ filteredTeams
│                                                     ↓            │
│  useLocation ──→ userLocation ──→ FlatList          │            │
│                       ↓              ↓               │            │
│                  TeamCard ←── renderItem              │            │
│                       ↓                              │            │
│              DistanceButton                          │            │
│                       ↓                              │            │
│              useStadiumDistance                       │            │
│                  ├─ geocodeStadium (+ cache)         │            │
│                  └─ haversineDistance                 │            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: React Concepts Used / Шпаргалка

| React Concept | Where Used | Swift Equivalent |
|---------------|-----------|-----------------|
| `useState` | useTeams (4x), useLocation (3x), FilterControls (1x), TeamsScreen (1x) | `@State` |
| `useEffect` | useTeams (load), useLocation (GPS), useDebounce (timer), useStadiumDistance (cleanup), TeamsScreen (sync search) | `.onAppear` / `.onChange` |
| `useCallback` | useTeams (all setters), TeamsScreen (renderItem, keyExtractor, handleReset) | Not needed in Swift |
| `useMemo` | useTeams (filteredTeams) | Computed property with cache |
| `useRef` | useStadiumDistance (mountedRef) | Stored property (no re-render) |
| `memo()` | TeamCard | `Equatable` conformance |
| `FlatList` | TeamsScreen | `UITableView` / `List` |
| Props | Every component | `init` parameters |
| Conditional rendering | `{condition && <Component />}` | `if condition { View() }` |
| Destructuring | Every file | `let (a, b) = tuple` |
| Template literals | `` `${count} teams` `` | `"\(count) teams"` |
| `async/await` | teams.ts, geocoding.ts, hooks | `async throws` |
| `Promise.all` | teams.ts (parallel fetch) | `TaskGroup` |

---

**You're ready. Open VS Code and walk through file by file. Start from Types → API → Utils → Hooks → Components → Main Screen. Bottom-up, just like building a Swift app.**
