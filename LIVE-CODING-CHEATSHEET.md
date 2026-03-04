# Onward Rides — Live Coding Cheatsheet

> 📱 Healthcare rideshare для пожилых. Accessibility = критично!

---

## 1️⃣ React Hooks

### useState
```tsx
const [rides, setRides] = useState<Ride[]>([]);
const [loading, setLoading] = useState(false);
```

### useEffect (с cleanup!)
```tsx
useEffect(() => {
  fetchRides();
  const unsubscribe = onLocationChange(...);
  return () => unsubscribe(); // ← cleanup!
}, [userId]);
```

### useCallback (для props в children)
```tsx
const handleRideAccept = useCallback((rideId: string) => {
  api.acceptRide(rideId);
}, [api]);

<RideCard onAccept={handleRideAccept} />
```

### useMemo (expensive calculations)
```tsx
const sortedRides = useMemo(() => {
  return rides
    .filter(r => r.status === 'pending')
    .sort((a, b) => a.eta - b.eta);
}, [rides]);
```

### useRef (no re-render)
```tsx
const mapRef = useRef<MapView>(null);
const scrollPos = useRef(0);

if (mapRef.current) {
  mapRef.current.fitToElements();
}
```

❌ **НЕ делай:**
- Hooks в `if`/`for`
- Пропущенные deps в useEffect
- useCallback без React.memo у ребёнка

---

## 2️⃣ Архитектура (Expo Router)

### Файловая структура
```
app/
├── (tabs)/
│   ├── index.tsx      # Home
│   ├── rides.tsx      # My Rides
│   └── profile.tsx
├── ride/[id].tsx      # Dynamic route
└── _layout.tsx        # Root Stack
```

### React Query (fetching)
```tsx
const { data: rides, isLoading } = useQuery({
  queryKey: ['rides'],
  queryFn: () => api.getRides(),
  staleTime: 5 * 60 * 1000  // 5 min cache
});
```

### Mutation + invalidate
```tsx
const { mutate: bookRide } = useMutation(
  api.bookRide,
  {
    onSuccess: () => {
      queryClient.invalidateQueries(['rides']);
      router.push('/rides');
    }
  }
);
```

---

## 3️⃣ Performance

### React.memo
```tsx
const RideCard = React.memo(({ ride, onSelect }) => {
  return (
    <TouchableOpacity onPress={() => onSelect(ride.id)}>
      <Text>{ride.destination}</Text>
    </TouchableOpacity>
  );
}, (prev, next) => prev.ride.id === next.ride.id);
```

### FlatList оптимизация
```tsx
<FlatList
  data={rides}
  renderItem={renderRide}
  keyExtractor={(item) => item.id}  // НЕ index!
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index
  })}
  windowSize={10}
/>
```

### StyleSheet (ОБЯЗАТЕЛЬНО)
```tsx
const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  text: { fontSize: 16, fontWeight: '600' }
});

// ✅ Правильно:
<View style={styles.container}>

// ❌ Неправильно:
<View style={{ flex: 1 }}>  // inline = re-create every render
```

---

## 4️⃣ TypeScript Patterns

### Discriminated Unions (state machine)
```tsx
type RideStatus =
  | { status: 'requested'; requestedAt: Date }
  | { status: 'assigned'; driver: Driver; eta: number }
  | { status: 'in_progress'; location: LatLng }
  | { status: 'completed'; summary: RideSummary }
  | { status: 'cancelled'; reason: string };

// Narrowing:
if (ride.status === 'assigned') {
  console.log(ride.driver);  // ✅ TypeScript знает!
}
```

### Branded Types (strong IDs)
```tsx
type RideId = string & { __brand: 'RideId' };
type DriverId = string & { __brand: 'DriverId' };

const createRideId = (id: string): RideId => id as RideId;

function acceptRide(rideId: RideId, driverId: DriverId) {
  // Нельзя перепутать!
}
```

### Generic API Response
```tsx
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: ApiError };

async function fetchRides(): Promise<ApiResponse<Ride[]>> {
  // ...
}
```

### Type Guard
```tsx
function isAssignedRide(ride: RideStatus):
  ride is Extract<RideStatus, { status: 'assigned' }> {
  return ride.status === 'assigned';
}
```

---

## 5️⃣ Real-Time

### WebSocket
```tsx
useEffect(() => {
  const ws = new WebSocket('wss://api.onward.app/rides');
  
  ws.onmessage = (event) => {
    const { rideId, location, eta } = JSON.parse(event.data);
    setRide(prev => ({ ...prev, location, eta }));
  };
  
  return () => ws.close();  // cleanup!
}, []);
```

### Offline Queue
```tsx
const { mutate: bookRide } = useMutation(
  async (rideData) => {
    if (isOnline) {
      return api.bookRide(rideData);
    } else {
      await queue.add(rideData);  // SQLite
    }
  }
);
```

### Push Notifications
```tsx
messaging().onMessage(async (remoteMessage) => {
  if (remoteMessage.data.type === 'driver_eta') {
    showNotification({
      title: 'Driver Arriving',
      body: remoteMessage.data.eta
    });
  }
});
```

---

## 6️⃣ Accessibility (CRITICAL для Onward!)

> 👴 Seniors = твои пользователи. VoiceOver должен работать!

```tsx
<TouchableOpacity
  accessible={true}
  accessibilityLabel="Accept ride to Downtown"
  accessibilityRole="button"
  accessibilityHint="Double tap to confirm"
/>
```

**Чеклист:**
- ✅ `accessibilityLabel` на всех кнопках
- ✅ Достаточный contrast ratio
- ✅ Large touch targets (44x44 pt min)
- ✅ Тестируй с VoiceOver/TalkBack

---

## 🚨 Типичные Ошибки (Live Coding)

| Ошибка | Как избежать |
|--------|--------------|
| Забыл cleanup в useEffect | Всегда `return () => ...` |
| index как key в FlatList | Использовать `item.id` |
| Inline styles | `StyleSheet.create()` |
| Пропустил deps в useEffect | ESLint + думать |
| any вместо типа | Создай type/interface |
| Нет error handling | try/catch + isError state |
| Нет loading state | `isLoading` перед data |

---

## 💡 Interview Tips

1. **Начинай с простого** — skeleton компонента
2. **Инкрементально** — state → fetch → display → errors → performance
3. **Проговаривай** свои решения вслух
4. **Accessibility** — упомяни сразу (Onward = healthcare)
5. **TypeScript** — покажи discriminated unions
6. **Тестируй** — напиши хотя бы 1 тест

**Порядок сборки компонента:**
```
1. Types/interfaces
2. useState/useRef
3. useEffect (fetch)
4. Handlers (useCallback)
5. JSX return
6. Styles
```

---

## 🎯 Quick Wins для Впечатления

- **Type guard** для API response
- **React.memo** с custom comparator
- **getItemLayout** в FlatList
- **Cleanup** в useEffect
- Упомянуть **offline-first** подход
- **accessibilityLabel** на кнопках

---

*Создано: 2026-03-02 | Onward Rides Live Coding Interview*
