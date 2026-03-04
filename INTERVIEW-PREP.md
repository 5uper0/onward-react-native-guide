# Interview Prep: Mobile App Engineer @ Onward
# Підготовка до інтерв'ю: Mobile App Engineer @ Onward

> **Interview: Thursday | 60 min | 1 web dev + 1 RN dev as interviewers**
> **Структура: Intro (5 min) -> Code Review (5 min) -> Live Coding (35 min) -> Backend/API (10 min) -> Edge Cases -> Outro (5 min)**

---

## YOUR 2-DAY PLAN / ПЛАН НА 2 ДНІ

| Day | Focus | Time |
|-----|-------|------|
| **Tuesday** | Sections 1-3 (Intro + Code Walkthrough + Live Coding prep) | 4h |
| **Wednesday** | Sections 4-6 (Backend/API + Edge Cases + Rapid-fire Q&A) + full mock run | 3h |

---

## SECTION 1: INTRO (5 min)
## РОЗДІЛ 1: ВСТУП (5 хв)

### "Tell me about yourself" — Ready script

**EN:**
> "I'm Oleh, a mobile engineer with over 10 years in iOS — I've shipped apps in Swift, SwiftUI, and Objective-C across healthcare, fintech, and consumer products. Recently I've been expanding into React Native because I see the cross-platform direction the industry is moving in. For this take-home, I built a Football Teams Explorer using Expo with TypeScript — it was a good exercise in applying my native mobile thinking to the React Native paradigm. I'm excited about Onward because the healthcare rideshare space is where mobile engineering really matters — accessibility, reliability, real-time tracking."

**RU:**
> "Меня зовут Олег, я мобильный инженер с 10+ годами опыта в iOS — Swift, SwiftUI, Objective-C. Работал в healthcare, fintech, потребительских продуктах. Сейчас активно осваиваю React Native — вижу что индустрия движется в сторону кроссплатформы. Для тестового задания сделал Football Teams Explorer на Expo с TypeScript. Onward интересен тем, что healthcare rideshare — это где мобильная разработка реально важна: accessibility, надёжность, real-time."

---

## SECTION 2: CODE WALKTHROUGH & ARCHITECTURE (5 min)
## РОЗДІЛ 2: ОГЛЯД КОДУ ТА АРХІТЕКТУРИ (5 хв)

### Your project structure (know this cold):

```
app/
  _layout.tsx          # Root Stack navigator (= UINavigationController)
  index.tsx            # Main screen — FlatList + SearchBar + Filters
src/
  api/
    teams.ts           # API client — parallel page fetching
    geocoding.ts       # Nominatim geocoding with in-memory cache
  components/
    TeamCard.tsx        # Memoized card (= UITableViewCell)
    SearchBar.tsx       # Controlled TextInput
    FilterControls.tsx  # Expandable chip-based filters
    DistanceButton.tsx  # On-demand geocoding + Haversine
    EmptyState.tsx      # Empty list placeholder
    ErrorState.tsx      # Error with retry
  hooks/
    useTeams.ts         # Main data hook — fetch + filter + sort
    useDebounce.ts      # Generic debounce hook
    useLocation.ts      # expo-location wrapper
    useStadiumDistance.ts # Geocoding + distance calc
  types/index.ts        # All TypeScript interfaces
  utils/
    distance.ts         # Haversine formula
    format.ts           # Value formatting ($2.0B, $200M)
```

### Q: "Walk me through your solution — what are you most proud of?"
### В: "Розкажіть про ваше рішення — чим пишаєтесь найбільше?"

**Answer / Відповідь:**
> "Three things I'm particularly happy with:
>
> 1. **The `useTeams` custom hook** (`src/hooks/useTeams.ts`) — it encapsulates ALL data logic: fetching, filtering, sorting, error handling, retry. The screen component (`app/index.tsx`) is pure UI. In iOS terms, this is like having a clean ViewModel that the View just observes.
>    *В iOS це як чистий ViewModel який View просто спостерігає.*
>
> 2. **Parallel API fetching** (`src/api/teams.ts`) — the API is paginated, so I fetch page 1 to get `total_pages`, then fetch all remaining pages with `Promise.all`. This is like using a `TaskGroup` in Swift concurrency.
>    *Як `TaskGroup` у Swift concurrency — всі сторінки паралельно.*
>
> 3. **The geocoding architecture** — each `DistanceButton` calculates distance on-demand (not on load), with an in-memory `Map` cache in `geocoding.ts`. This prevents hitting the Nominatim rate limit and saves battery. In iOS I'd use `NSCache` for the same purpose.
>    *Кеш як `NSCache` — не навантажуємо API та економимо батарею.*"

### Q: "How did you structure your state management?"
### В: "Як ви організували управління станом?"

**Answer:**
> "I used **local state only** — no Redux, no Context, no Zustand. Here's why:
>
> - **`useTeams` hook** owns the teams data + filter state (`useState` for `teams`, `isLoading`, `error`, `filters`)
> - **`useDebounce`** in `index.tsx` debounces search input before passing to `useTeams.setSearch`
> - **`useLocation`** is independent — provides user coordinates
> - **`useStadiumDistance`** is per-card, triggered on tap
>
> For a single-screen app, lifting state to Context/Zustand would be over-engineering. If we added a detail screen or favorites, I'd introduce Zustand with AsyncStorage persistence.
>
> *Для одного екрану Context/Zustand — це over-engineering. Якщо додати деталі або обране — додав би Zustand з AsyncStorage.*"

**iOS comparison:** This is like having each screen own its own `@StateObject` ViewModel, no `@EnvironmentObject` needed yet.

### Q: "Why this component breakdown?"
### В: "Чому саме такий розподіл компонентів?"

**Answer:**
> "I followed the **Single Responsibility Principle**:
> - `TeamCard` — displays one team (memoized with `React.memo` = like conforming to `Equatable` in SwiftUI)
> - `SearchBar` — controlled input, knows nothing about teams
> - `FilterControls` — manages its own expand/collapse state, emits filter changes up via callbacks (like `@Binding` in SwiftUI)
> - `DistanceButton` — owns its own async state (loading/error/result) via `useStadiumDistance`
> - `EmptyState` / `ErrorState` — pure presentation components, zero logic
>
> *Кожен компонент відповідає за одну річ. Як Single Responsibility в SOLID.*"

### Q: "How does your filtering logic work — client-side or API-driven?"
### В: "Фільтрація на клієнті чи через API?"

**Answer:**
> "**Client-side**, and here's the deliberate trade-off:
>
> The API (`jsonmock.hackerrank.com`) doesn't support server-side filtering. So I load ALL teams on mount (parallel page fetch), store them in state, and filter/sort with `useMemo` in `useTeams.ts:57-82`.
>
> The `useMemo` dependency array is `[teams, filters]` — it only recalculates when the raw data or filters change. With ~50 teams this is instant. The sort is by `estimated_value_numeric` descending.
>
> *`useMemo` перераховує тільки коли `teams` або `filters` змінюються. Для 50 команд це миттєво.*"

### Q: "How would this hold up with 10,000 teams?"
### В: "Як це буде працювати з 10,000 командами?"

**Answer:**
> "With 10K teams, I'd change three things:
>
> 1. **Server-side filtering/sorting** — move the filter logic to the API with query params (`?league=Premier+League&min_value=1000000000&sort=value_desc`)
> 2. **Pagination with infinite scroll** — use `FlatList.onEndReached` + cursor-based pagination instead of loading everything
> 3. **`getItemLayout`** on FlatList — since cards have variable height, I'd either fix the height or use `FlashList` from Shopify (the modern high-perf alternative to FlatList)
>
> Currently I already have `initialNumToRender={10}`, `maxToRenderPerBatch={10}`, `windowSize={5}` in `index.tsx:113-115`. These control the virtualization window.
>
> In iOS terms: FlatList's virtualization is like `UITableView`'s cell reuse, and `windowSize` controls how many screens of content are kept in memory (like `estimatedRowHeight`).
>
> *FlatList = UITableView з cell reuse. `windowSize` контролює скільки екранів тримати в пам'яті.*"

---

## SECTION 3: LIVE CODING (35 min)
## РОЗДІЛ 3: LIVE CODING (35 хв)

> **They'll pick ONE of three options. You must be ready for all three.**
> **Виберуть ОДИН з трьох варіантів. Треба бути готовим до всіх.**

### Your live coding strategy (say this out loud at the start):

> "Let me start by clarifying the requirements, then I'll outline my approach before coding."

```
1. CLARIFY (2 min)  — ask scope questions
2. PLAN (3 min)     — describe component tree + state + types
3. IMPLEMENT (20 min) — code incrementally, talk through decisions
4. TEST (5 min)     — edge cases, optimizations
```

---

### OPTION A: Team Detail Screen (MOST LIKELY)
### ВАРІАНТ А: Екран деталей команди (НАЙІМОВІРНІШЕ)

**They look for:** React Navigation setup, param passing, avoiding unnecessary re-fetches.
**Шукають:** Налаштування навігації, передача параметрів, уникнення зайвих запитів.

**Step 1: Create the route file**
```tsx
// app/team/[name].tsx  — dynamic route (like /team/:name in web)
// iOS equivalent: pushing a detail VC onto UINavigationController
import { useLocalSearchParams, Stack } from 'expo-router';
import React from 'react';
import { View, Text, ScrollView, StyleSheet } from 'react-native';
import { FootballTeam } from '../../src/types';

export default function TeamDetailScreen() {
  // useLocalSearchParams = reading route params (like vc.navigationItem in UIKit)
  const { name, teamData } = useLocalSearchParams<{
    name: string;
    teamData: string;
  }>();

  // Parse the serialized team data — avoids re-fetching!
  const team: FootballTeam = JSON.parse(teamData ?? '{}');

  return (
    <>
      {/* Stack.Screen dynamically sets the header title */}
      <Stack.Screen options={{ title: team.name, headerShown: true }} />
      <ScrollView style={styles.container}>
        <Text style={styles.name}>{team.name}</Text>
        <Text style={styles.league}>{team.league} ({team.nation})</Text>

        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Team Info</Text>
          <DetailRow label="Manager" value={team.manager} />
          <DetailRow label="Captain" value={team.captain} />
          <DetailRow label="Stadium" value={`${team.stadium_name} (${team.stadium_capacity.toLocaleString()})`} />
          <DetailRow label="Value" value={team.estimated_value} />
        </View>

        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Honors</Text>
          <DetailRow label="League Titles" value={String(team.number_of_league_titles_won)} />
          <DetailRow label="Champions League" value={String(team.number_of_champions_league_won)} />
          <DetailRow label="Total Trophies" value={String(team.total_silverware_count)} />
        </View>

        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Key Players</Text>
          <DetailRow label="Top Scorer" value={team.highest_goalscorer} />
          <DetailRow label="Most Assists" value={team.highest_assist_provider} />
          <DetailRow label="Most Caps" value={`${team.most_capped_player} (${team.appearances_most_capped_player})`} />
        </View>
      </ScrollView>
    </>
  );
}

function DetailRow({ label, value }: { label: string; value: string }) {
  return (
    <View style={styles.row}>
      <Text style={styles.label}>{label}</Text>
      <Text style={styles.value}>{value}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f8f9fa', padding: 16 },
  name: { fontSize: 24, fontWeight: '800', color: '#1a1a1a' },
  league: { fontSize: 16, color: '#666', marginTop: 4, marginBottom: 20 },
  section: { marginBottom: 20 },
  sectionTitle: { fontSize: 18, fontWeight: '700', color: '#333', marginBottom: 10, borderBottomWidth: 1, borderBottomColor: '#eee', paddingBottom: 6 },
  row: { flexDirection: 'row', justifyContent: 'space-between', paddingVertical: 8 },
  label: { fontSize: 14, color: '#666' },
  value: { fontSize: 14, fontWeight: '600', color: '#333', textAlign: 'right', flex: 1, marginLeft: 16 },
});
```

**Step 2: Make TeamCard tappable — navigate with data**
```tsx
// In TeamCard.tsx — wrap in TouchableOpacity (or Pressable)
import { TouchableOpacity } from 'react-native';
import { router } from 'expo-router';

// In the component:
<TouchableOpacity
  onPress={() => router.push({
    pathname: '/team/[name]',
    params: {
      name: team.name,
      teamData: JSON.stringify(team),  // Pass data to avoid re-fetch
    },
  })}
>
  {/* existing card content */}
</TouchableOpacity>
```

**Step 3: Update _layout.tsx to show header on detail**
```tsx
// app/_layout.tsx — already has headerShown: false globally
// The detail screen overrides with Stack.Screen options={{ headerShown: true }}
// So no change needed in _layout.tsx!
```

**Key talking points:**
- "I serialize the team data to avoid re-fetching. In iOS, you'd just pass the model object directly via the segue/coordinator. In RN, route params must be serializable."
- "An alternative is to pass just the `name` and look up the team from a shared store (Zustand/Context). But since we already have the data, passing it is simpler and avoids an extra dependency."
- *"Серіалізую дані команди щоб не робити повторний запит. В iOS передали б модель через segue. В RN параметри маршруту мають бути серіалізовані."*

---

### OPTION B: Pagination / Infinite Scroll
### ВАРІАНТ Б: Пагінація / Нескінченний скрол

**They look for:** `FlatList.onEndReached`, loading state management, deduplication.
**Шукають:** `onEndReached`, управління станом завантаження, дедуплікація.

```tsx
// Modified useTeams.ts with pagination
export function useTeams() {
  const [teams, setTeams] = useState<FootballTeam[]>([]);
  const [page, setPage] = useState(1);
  const [totalPages, setTotalPages] = useState(1);
  const [isLoading, setIsLoading] = useState(true);
  const [isLoadingMore, setIsLoadingMore] = useState(false);

  const loadPage = useCallback(async (pageNum: number) => {
    const isFirst = pageNum === 1;
    if (isFirst) setIsLoading(true);
    else setIsLoadingMore(true);

    try {
      const response = await fetch(
        `https://jsonmock.hackerrank.com/api/football_teams?page=${pageNum}`
      );
      const data: TeamsApiResponse = await response.json();

      setTotalPages(data.total_pages);
      setTeams(prev => isFirst ? data.data : [...prev, ...data.data]);
      // Dedup: if worried about duplicates:
      // setTeams(prev => {
      //   const existing = new Set(prev.map(t => t.name));
      //   const newTeams = data.data.filter(t => !existing.has(t.name));
      //   return [...prev, ...newTeams];
      // });
    } catch (err) { /* error handling */ }
    finally {
      setIsLoading(false);
      setIsLoadingMore(false);
    }
  }, []);

  useEffect(() => { loadPage(1); }, [loadPage]);

  const loadMore = useCallback(() => {
    if (!isLoadingMore && page < totalPages) {
      const nextPage = page + 1;
      setPage(nextPage);
      loadPage(nextPage);
    }
  }, [isLoadingMore, page, totalPages, loadPage]);

  return { teams, isLoading, isLoadingMore, loadMore, hasMore: page < totalPages };
}

// In FlatList:
<FlatList
  data={teams}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}   // Trigger at 50% from bottom
  ListFooterComponent={isLoadingMore ? <ActivityIndicator /> : null}
/>
```

**Key talking points:**
- "`onEndReachedThreshold={0.5}` means trigger when 50% of remaining content is visible. In iOS this is like `scrollViewDidScroll` checking `contentOffset + bounds.height > contentSize.height - threshold`."
- "Deduplication: I use a `Set` of team names to prevent duplicates if the user scrolls fast and triggers multiple loads."
- *"`onEndReachedThreshold` — як перевірка `contentOffset` в `scrollViewDidScroll`."*

---

### OPTION C: Favorites with Persistence
### ВАРІАНТ В: Обране з збереженням

**They look for:** AsyncStorage read/write, lifting state or context, tab navigation.
**Шукають:** AsyncStorage, підняття стану, tab navigation.

**Step 1: Create a favorites context**
```tsx
// src/context/FavoritesContext.tsx
import React, { createContext, useContext, useState, useEffect, useCallback } from 'react';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface FavoritesContextType {
  favorites: Set<string>;
  toggleFavorite: (name: string) => void;
  isFavorite: (name: string) => boolean;
}

const FavoritesContext = createContext<FavoritesContextType | null>(null);

export function FavoritesProvider({ children }: { children: React.ReactNode }) {
  const [favorites, setFavorites] = useState<Set<string>>(new Set());

  // Load from AsyncStorage on mount
  useEffect(() => {
    AsyncStorage.getItem('favorites').then(stored => {
      if (stored) setFavorites(new Set(JSON.parse(stored)));
    });
  }, []);

  const toggleFavorite = useCallback((name: string) => {
    setFavorites(prev => {
      const next = new Set(prev);
      if (next.has(name)) next.delete(name);
      else next.add(name);
      // Persist (fire and forget)
      AsyncStorage.setItem('favorites', JSON.stringify([...next]));
      return next;
    });
  }, []);

  const isFavorite = useCallback((name: string) => favorites.has(name), [favorites]);

  return (
    <FavoritesContext.Provider value={{ favorites, toggleFavorite, isFavorite }}>
      {children}
    </FavoritesContext.Provider>
  );
}

export function useFavorites() {
  const ctx = useContext(FavoritesContext);
  if (!ctx) throw new Error('useFavorites must be inside FavoritesProvider');
  return ctx;
}
```

**Step 2: Tab navigation layout**
```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
export default function TabLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" options={{ title: 'Teams', tabBarIcon: /* football icon */ }} />
      <Tabs.Screen name="favorites" options={{ title: 'Favorites', tabBarIcon: /* heart icon */ }} />
    </Tabs>
  );
}
```

**iOS comparison:** `FavoritesProvider` wrapping the app in `_layout.tsx` is exactly like injecting an `@EnvironmentObject` at the App level in SwiftUI. `AsyncStorage` is like `UserDefaults` (or `NSUbiquitousKeyValueStore` for sync).

---

## SECTION 4: BACKEND/API EXTENSION (10 min)
## РОЗДІЛ 4: БЕКЕНД/API (10 хв)

### Q: "How would you structure a teams GraphQL query?"
### В: "Як би ви структурували GraphQL запит?"

```graphql
query Teams($filters: TeamsFilterInput!, $pagination: PaginationInput!) {
  teams(filters: $filters, pagination: $pagination) {
    edges {
      node {
        id
        name
        league
        estimatedValue
        stadiumName
        stadiumCapacity
        leagueTitlesWon
        championsLeagueWon
        manager
        captain
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
      totalCount
    }
  }
}

# Variables:
{
  "filters": {
    "search": "Real",
    "minValue": 1000000000,
    "leagues": ["La Liga", "Premier League"]
  },
  "pagination": {
    "first": 20,
    "after": "cursor_abc123"
  }
}
```

### Q: "Offset vs cursor-based pagination — and why?"
### В: "Offset чи cursor пагінація — і чому?"

| | Offset (`?page=2&per_page=10`) | Cursor (`?after=abc123&first=10`) |
|---|---|---|
| **Pros** | Simple, can jump to any page | Stable with real-time inserts/deletes |
| **Cons** | Breaks if data changes between pages | Can't jump to page N |
| **Use when** | Static data, admin dashboards | Feeds, real-time data, infinite scroll |
| **iOS equiv** | `NSFetchedResultsController` with offset | Core Data cursor / `fetchOffset` |

> "For Onward, I'd choose **cursor-based** because rides are created/completed constantly. Offset pagination would show duplicates or skip items when the underlying data changes between page loads."
>
> *"Для Onward обрав би cursor-based тому що поїздки постійно створюються/завершуються. При offset пагінації будуть дублі або пропуски."*

### Q: "How would you handle auth tokens in your fetch setup?"
### В: "Як обробляти auth tokens?"

```tsx
// services/api.ts — centralized fetch wrapper
const API_BASE = 'https://api.onward.app/graphql';

async function graphqlFetch<T>(query: string, variables?: object): Promise<T> {
  // Get token from secure storage (like iOS Keychain)
  const token = await SecureStore.getItemAsync('auth_token');

  const response = await fetch(API_BASE, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,  // JWT in header
    },
    body: JSON.stringify({ query, variables }),
  });

  if (response.status === 401) {
    // Token expired — try refresh
    const newToken = await refreshToken();
    // Retry with new token...
  }

  const json = await response.json();
  if (json.errors) throw new GraphQLError(json.errors);
  return json.data;
}
```

> "In iOS, I'd use `URLSession` with a custom `URLProtocol` or `RequestInterceptor` for the same token injection pattern. The key principles are the same: centralized auth, token refresh on 401, secure token storage."

---

## SECTION 5: EDGE CASES & QUALITY
## РОЗДІЛ 5: КРАЙНІ ВИПАДКИ ТА ЯКІСТЬ

### Q: "What happens if the geocoding API is slow or fails?"
### В: "Що станеться якщо geocoding API повільний або впаде?"

> "I designed for this specifically:
>
> 1. **On-demand calculation** — `DistanceButton` only geocodes when tapped, not on load. This means a slow geocoding API doesn't block the list rendering.
> 2. **Loading state** — the button shows `ActivityIndicator` while waiting
> 3. **Error state with retry** — if it fails, the button shows the error and tapping again retries
> 4. **In-memory cache** (`geocoding.ts:6`) — `Map<string, coords>` prevents re-hitting the API for already geocoded stadiums
> 5. **Cancelled flag** (`useStadiumDistance.ts:16`) — `mountedRef` prevents state updates after unmount (like checking `Task.isCancelled` in Swift)
>
> *Кнопка геокодує тільки при натисканні. Кеш `Map` запобігає повторним запитам. `mountedRef` запобігає оновленню стану після unmount.*"

### Q: "How would you test the filtering logic?"
### В: "Як би ви тестували логіку фільтрації?"

```tsx
// __tests__/useTeams.test.ts
import { renderHook, act, waitFor } from '@testing-library/react-native';

// Mock the API
jest.mock('../src/api/teams', () => ({
  fetchAllTeams: jest.fn().mockResolvedValue([
    { name: 'Real Madrid', estimated_value_numeric: 2_000_000_000, number_of_league_titles_won: 35, /* ... */ },
    { name: 'Arsenal', estimated_value_numeric: 500_000_000, number_of_league_titles_won: 13, /* ... */ },
    { name: 'Burnley', estimated_value_numeric: 50_000_000, number_of_league_titles_won: 2, /* ... */ },
  ]),
}));

describe('useTeams filtering', () => {
  it('filters by search term', async () => {
    const { result } = renderHook(() => useTeams());
    await waitFor(() => expect(result.current.isLoading).toBe(false));

    act(() => result.current.setSearch('Real'));
    expect(result.current.filteredTeams).toHaveLength(1);
    expect(result.current.filteredTeams[0].name).toBe('Real Madrid');
  });

  it('filters by minimum value', async () => {
    const { result } = renderHook(() => useTeams());
    await waitFor(() => expect(result.current.isLoading).toBe(false));

    act(() => result.current.setMinValue(1_000_000_000));
    expect(result.current.filteredTeams).toHaveLength(1); // Only Real Madrid
  });

  it('sorts by value descending', async () => {
    const { result } = renderHook(() => useTeams());
    await waitFor(() => expect(result.current.isLoading).toBe(false));

    const names = result.current.filteredTeams.map(t => t.name);
    expect(names).toEqual(['Real Madrid', 'Arsenal', 'Burnley']);
  });

  it('resets filters', async () => {
    const { result } = renderHook(() => useTeams());
    await waitFor(() => expect(result.current.isLoading).toBe(false));

    act(() => result.current.setSearch('xyz'));
    expect(result.current.filteredTeams).toHaveLength(0);

    act(() => result.current.resetFilters());
    expect(result.current.filteredTeams).toHaveLength(3);
  });
});
```

> "In iOS terms, this is like testing a ViewModel with `XCTestExpectation` — mock the network layer, verify the published outputs."

### Q: "Where would you add error boundaries?"
### В: "Де б ви додали error boundaries?"

```tsx
// Error Boundary = try/catch for React rendering errors
// (React Native doesn't have this built-in — need a class component)

// app/_layout.tsx
import { ErrorBoundary } from '../src/components/ErrorBoundary';

export default function RootLayout() {
  return (
    <ErrorBoundary fallback={<CrashScreen />}>
      <Stack screenOptions={{ headerShown: false }} />
    </ErrorBoundary>
  );
}
```

> "I'd wrap the root layout and also individual screens that might crash. Error boundaries catch rendering errors only — not event handlers or async code. For those, I use try/catch + error state.
>
> In iOS: Error boundaries are like `applicationDidBecomeActive` crash recovery or `NSSetUncaughtExceptionHandler`.
>
> *Error boundaries ловлять тільки помилки рендерингу. Для async — try/catch + error state.*"

---

## SECTION 6: RAPID-FIRE Q&A — HOOKS, OPTIMIZATION, TYPESCRIPT
## РОЗДІЛ 6: ШВИДКІ ЗАПИТАННЯ

### Hooks: When to use what?

| Hook | When / Коли | iOS Equivalent | In your project |
|------|-------------|----------------|-----------------|
| `useState` | Local component state | `@State` | `useTeams.ts:30-33` — teams, loading, error, filters |
| `useEffect` | Side effects (fetch, subscriptions, timers) | `onAppear` + `onDisappear` cleanup | `useTeams.ts:45-47` — fetch on mount; `useLocation.ts:23` — request permissions |
| `useCallback` | Memoize functions passed to children | No direct equivalent (Swift closures don't re-create) | `useTeams.ts:84-97` — setSearch, setMinValue, setMinTitles |
| `useMemo` | Memoize expensive computations | Cached computed property | `useTeams.ts:49-82` — filtering + sorting |
| `useRef` | Mutable value that doesn't trigger re-render | Instance variable (`var` in class) | `useStadiumDistance.ts:16` — mountedRef for cleanup |
| Custom Hook | Reusable stateful logic | ViewModel (ObservableObject) | `useTeams`, `useDebounce`, `useLocation`, `useStadiumDistance` |

### When would you use custom hooks? / Коли використовувати кастомні хуки?

> "I extract a custom hook when:
> 1. **Multiple components need the same stateful logic** (like `useDebounce` — generic, reusable)
> 2. **A component's logic is complex enough to separate** (like `useTeams` — keeps `index.tsx` focused on UI)
> 3. **Testing is easier in isolation** (hook can be tested with `renderHook` without mounting UI)
>
> In iOS terms: a custom hook is the ViewModel. The convention is the `use` prefix, like the `@Observable` macro in Swift.
>
> *Кастомний хук = ViewModel. Виділяю коли логіка складна або переиспользовується.*"

### Optimization: How to prevent double render?

| Technique | What it does | iOS Equivalent | Your project usage |
|-----------|-------------|----------------|-------------------|
| `React.memo` | Skip re-render if props unchanged | `Equatable` on SwiftUI View | `TeamCard.tsx:69` — `export const TeamCard = memo(TeamCardComponent)` |
| `useCallback` | Stable function reference | N/A (Swift closures are stable) | `index.tsx:50-55` — `renderItem` wrapped in `useCallback` |
| `useMemo` | Cache computed value | Lazy computed property with cache | `useTeams.ts:49` — filtered/sorted teams |
| `useDebounce` | Delay state updates | `Combine.debounce()` | `index.tsx:23` — 300ms search debounce |
| FlatList virtualization | Only render visible items | `UITableView` cell reuse | `index.tsx:112-116` — `windowSize={5}`, `initialNumToRender={10}` |
| `StyleSheet.create` | Pre-compute style objects | N/A (UIKit styles don't re-create) | Every component — styles defined outside render |

### Caching strategy:

> "Three caching layers in the app:
> 1. **In-memory geocoding cache** (`geocoding.ts:6`) — `Map<string, coords>` avoids repeated API calls
> 2. **useMemo** for filtered results — avoids re-filtering on every render
> 3. **React's reconciliation** — FlatList + `keyExtractor` with `team.name` enables efficient diffing
>
> If I had React Query, I'd add a 4th layer: `staleTime: 5 * 60 * 1000` to cache API responses for 5 minutes.
>
> *Три рівні кешування: geocoding Map, useMemo для фільтрів, React reconciliation.*"

### Battery optimization:

> "Key decisions for battery:
> 1. **On-demand geocoding** — not fetching coordinates for all 50 stadiums on load
> 2. **`Accuracy.Balanced`** in `useLocation.ts:32` — not GPS-level accuracy, which drains battery
> 3. **Cancellation** — `cancelled` flag in `useLocation.ts:24` prevents work after unmount
> 4. **No polling** — data loaded once, no timers/intervals running
>
> *`Accuracy.Balanced` замість GPS. Геокодинг тільки при натисканні. Немає polling.*"

---

## iOS -> React Native MENTAL MODEL (Your Secret Weapon)
## iOS -> React Native МЕНТАЛЬНА МОДЕЛЬ (Твоя секретна зброя)

Use this when explaining your decisions — it shows you truly understand both worlds:

| iOS Concept | React Native Equivalent | Key Difference |
|---|---|---|
| `@State var count = 0` | `const [count, setCount] = useState(0)` | Must use setter, immutable updates |
| `@Binding` | `props + callback` | No two-way binding — data flows down, events up |
| `@EnvironmentObject` | `Context` or `Zustand` | Context re-renders all consumers; Zustand is selective |
| `ObservableObject` + `@Published` | Custom Hook (e.g., `useTeams`) | Hook returns values + setters, like a ViewModel |
| `.onAppear { }` | `useEffect(() => {}, [])` | Dependency array controls WHEN it re-runs |
| `.onDisappear { }` | `useEffect` cleanup function | `return () => { cleanup() }` |
| `Combine.debounce()` | `useDebounce` custom hook | Same concept, different syntax |
| `UITableView` + `dequeueReusableCell` | `FlatList` + `keyExtractor` | FlatList virtualizes automatically |
| `UINavigationController.push` | `router.push('/team/name')` | File-based routing with Expo Router |
| `Codable` struct | TypeScript `interface` | TS interfaces have no runtime presence |
| `guard let` / `if let` | Optional chaining `?.` + nullish coalescing `??` | JS has no pattern matching (yet) |
| `NSCache` | `Map<string, T>` / React Query cache | In-memory; for persistence use AsyncStorage |
| `XCTest` + `XCTestExpectation` | `Jest` + `waitFor` | Async testing with `renderHook` for hooks |
| `DispatchQueue.main.async` | State updates already batched by React | React 18 auto-batches all state updates |
| `CLLocationManager` | `expo-location` | Same permission model, simpler API |
| `Info.plist` usage description | `app.json > ios > infoPlist` | Same concept, configured in Expo config |

---

## FINAL CHECKLIST (Thursday morning)
## ФІНАЛЬНИЙ ЧЕКЛИСТ (ранок четверга)

- [ ] Can explain every file in the project and why it exists
- [ ] Can walk through `useTeams.ts` line by line
- [ ] Can build a detail screen from scratch in <15 min
- [ ] Know `useState`, `useEffect`, `useCallback`, `useMemo`, `useRef` cold
- [ ] Can explain dependency arrays with examples
- [ ] Can explain `React.memo` vs `useMemo` vs `useCallback`
- [ ] Can discuss GraphQL query structure
- [ ] Can explain cursor vs offset pagination
- [ ] Can write a basic Jest test for the filtering logic
- [ ] Have iOS analogies ready for every RN concept
- [ ] Remember: **think out loud**, explain **why** not just what

---

> **Remember: You have 10+ years of iOS engineering. The concepts are the same — state management, component composition, async data loading, caching, testing. The syntax is just different. You already know this stuff.**
>
> **Пам'ятай: У тебе 10+ років iOS досвіду. Концепції ті ж — управління станом, композиція компонентів, асинхронне завантаження, кешування, тестування. Тільки синтаксис інший. Ти вже все це знаєш.**
