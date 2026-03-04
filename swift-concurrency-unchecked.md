# Swift Concurrency: Unchecked Escape Hatches — iOS Interview Cheatsheet

## Why These Exist
Swift 6 enforces **strict concurrency checking**. Sometimes you need escape hatches for:
- Legacy code migration
- Interfacing with C/Objective-C
- Performance-critical code where you *know* it's safe
- Types that are thread-safe but compiler can't verify

**⚠️ Key principle:** These are **last resorts**. Prefer proper Sendable conformance.

---

## @unchecked Sendable

### What It Does
Tells the compiler: "Trust me, this type is safe to share across concurrency domains."

```swift
// Compiler can't verify this is thread-safe
final class Cache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Data] = [:]
    
    func get(_ key: String) -> Data? {
        lock.lock()
        defer { lock.unlock() }
        return storage[key]
    }
    
    func set(_ key: String, value: Data) {
        lock.lock()
        defer { lock.unlock() }
        storage[key] = value
    }
}
```

### When to Use
✅ **Appropriate:**
- Types protected by locks/queues that compiler can't verify
- Immutable reference types (compiler doesn't know they're immutable)
- Wrappers around thread-safe C libraries
- `@MainActor`-bound classes used only from main thread

```swift
// Example: Immutable class
final class Config: @unchecked Sendable {
    let apiURL: URL      // let = immutable
    let timeout: Int     // Compiler doesn't know class is immutable
    
    init(apiURL: URL, timeout: Int) {
        self.apiURL = apiURL
        self.timeout = timeout
    }
}
```

❌ **Wrong:**
- "I don't understand the error, let me silence it"
- Mutable state without proper synchronization
- Quick fix to make code compile

---

## nonisolated(unsafe)

### What It Does
Allows access to mutable state from any isolation domain without compiler checks.
**Swift 5.10+ / Swift 6**

```swift
actor DataManager {
    // This property can be accessed from anywhere — YOU guarantee safety
    nonisolated(unsafe) var cachedValue: String?
    
    func updateCache(_ value: String) {
        cachedValue = value  // No actor isolation enforced
    }
}
```

### When to Use
✅ **Appropriate:**
- Global mutable state you're migrating incrementally
- Properties accessed from C callbacks
- Interfacing with legacy frameworks

```swift
// Legacy global state during migration
nonisolated(unsafe) var legacySharedState: [String: Any] = [:]

// C interop — callback doesn't respect Swift concurrency
nonisolated(unsafe) var callbackResult: UnsafeMutablePointer<CChar>?
```

❌ **Wrong:**
- Avoiding actor isolation because it's "inconvenient"
- Any new code — use proper actors instead

---

## @preconcurrency

### What It Does
Suppresses Sendable warnings when importing modules not yet updated for concurrency.

```swift
@preconcurrency import SomeOldFramework

// Use types from SomeOldFramework without Sendable warnings
func process(_ item: OldFrameworkType) async {
    // Compiler trusts this won't cause data races
}
```

### When to Use
- Third-party dependencies not updated for Swift 6
- System frameworks with missing Sendable conformances
- Gradual migration of large codebases

---

## Comparison Table

| Escape Hatch | Scope | Use Case |
|--------------|-------|----------|
| `@unchecked Sendable` | Type conformance | Thread-safe types compiler can't verify |
| `nonisolated(unsafe)` | Property/variable | Mutable state outside isolation |
| `@preconcurrency` | Import statement | Legacy modules without Sendable |

---

## The Risks (Interview Gold!)

### Data Races
```swift
// DANGEROUS — no synchronization!
final class BadCache: @unchecked Sendable {
    var storage: [String: Data] = [:]  // 💥 Data race
}

Task { cache.storage["a"] = data1 }
Task { cache.storage["b"] = data2 }  // 💀 Undefined behavior
```

### Silent Corruption
- No compiler warnings after you opt out
- Bugs may appear intermittently (race conditions)
- Extremely hard to debug

### Audit Liability
- Every `@unchecked` is a promise YOU maintain
- Code changes can break thread-safety assumptions
- Document WHY it's safe!

```swift
/// Thread-safe: All access synchronized via `lock`.
/// Safe to mark @unchecked Sendable.
final class DocumentedCache: @unchecked Sendable {
    // ...
}
```

---

## Common Interview Questions

### Q: When would you use @unchecked Sendable?
"When I have a type that IS thread-safe through mechanisms the compiler can't verify — like locks, atomics, or immutability in a reference type. I'd document WHY it's safe and consider it tech debt to revisit."

### Q: What's the difference between @unchecked Sendable and nonisolated(unsafe)?
- `@unchecked Sendable` — marks a **type** as safely shareable
- `nonisolated(unsafe)` — marks a **property** as accessible outside isolation

### Q: How do you make a class properly Sendable without @unchecked?
```swift
// Option 1: Make it immutable (all let properties)
final class Config: Sendable {
    let value: Int
}

// Option 2: Use actor
actor SafeCache {
    var storage: [String: Data] = [:]
}

// Option 3: Use Mutex (Swift 6) or OSAllocatedUnfairLock
final class ModernCache: Sendable {
    let storage = Mutex<[String: Data]>([:])
}
```

### Q: What happens if you lie with @unchecked Sendable?
- **Undefined behavior** — data races, crashes, corruption
- **Silent bugs** — may work in testing, fail in production
- **No compiler help** — you're on your own

---

## Best Practices

1. **Exhaust safe options first**
   - Can you use an actor?
   - Can you make it immutable?
   - Can you use `Mutex` or `OSAllocatedUnfairLock`?

2. **Document heavily**
   ```swift
   /// SAFETY: All mutable state accessed only from main thread.
   /// Verified via @MainActor on all entry points.
   final class ViewModel: @unchecked Sendable { }
   ```

3. **Centralize escape hatches**
   - Keep them in dedicated files
   - Makes auditing easier

4. **Plan to remove them**
   - Track as tech debt
   - Revisit when dependencies update

---

## Red Flags
❌ Using `@unchecked Sendable` to "make it compile"  
❌ No documentation on WHY it's safe  
❌ `nonisolated(unsafe)` in new code  
❌ Multiple escape hatches scattered everywhere  

## Points to Impress
✅ Explain the risks clearly  
✅ Show you know proper alternatives (actors, Mutex)  
✅ Mention documentation and code review  
✅ Discuss migration strategy for legacy code  
