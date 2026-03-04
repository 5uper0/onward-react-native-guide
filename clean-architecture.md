# Clean Architecture — iOS Interview Cheatsheet

## Core Concept
Clean Architecture (Uncle Bob) = **separation of concerns through layers with strict dependency rules**.

```
┌─────────────────────────────────────────┐
│           UI / Presentation             │  ← Frameworks & Drivers
├─────────────────────────────────────────┤
│         Interface Adapters              │  ← ViewModels, Presenters
├─────────────────────────────────────────┤
│            Use Cases                    │  ← Application Business Rules
├─────────────────────────────────────────┤
│             Entities                    │  ← Enterprise Business Rules
└─────────────────────────────────────────┘
```

**The Dependency Rule:** Dependencies point INWARD only. Inner layers know nothing about outer layers.

---

## The 4 Layers

### 1. Entities (Domain)
- Pure business objects, no frameworks
- `struct User`, `struct Order`
- Protocols for repositories: `protocol UserRepository`

### 2. Use Cases (Application)
- Single responsibility: one business action
- Orchestrate entities, call repositories
- Input → Process → Output

```swift
protocol FetchUserUseCaseProtocol {
    func execute(userId: String) async throws -> User
}

final class FetchUserUseCase: FetchUserUseCaseProtocol {
    private let repository: UserRepository
    
    init(repository: UserRepository) {
        self.repository = repository
    }
    
    func execute(userId: String) async throws -> User {
        try await repository.getUser(by: userId)
    }
}
```

### 3. Interface Adapters (Presentation)
- ViewModels, Presenters, Mappers
- Convert domain models ↔ view models
- Handle UI logic, NOT business logic

```swift
@MainActor
final class UserViewModel: ObservableObject {
    @Published var userName: String = ""
    @Published var isLoading = false
    
    private let fetchUserUseCase: FetchUserUseCaseProtocol
    
    func load(userId: String) async {
        isLoading = true
        defer { isLoading = false }
        
        if let user = try? await fetchUserUseCase.execute(userId: userId) {
            userName = user.fullName // Map domain → presentation
        }
    }
}
```

### 4. Frameworks & Drivers (Infrastructure)
- UIKit/SwiftUI views
- Network clients, databases, third-party SDKs
- Repository implementations

```swift
final class UserRepositoryImpl: UserRepository {
    private let apiClient: APIClient
    
    func getUser(by id: String) async throws -> User {
        let dto = try await apiClient.fetch(UserDTO.self, from: "/users/\(id)")
        return dto.toDomain() // Map DTO → Entity
    }
}
```

---

## Clean Architecture vs VIPER vs VIP

| Aspect | Clean Architecture | VIPER | VIP (Clean Swift) |
|--------|-------------------|-------|-------------------|
| **Scope** | App-wide layers | Per-module | Per-scene |
| **Router** | Optional/flexible | Mandatory Router | Mandatory Router |
| **Data flow** | Flexible | Presenter ↔ View | Unidirectional cycle |
| **Interactor** | = Use Case | 1 per module | 1 per scene |
| **Presenter** | In adapters layer | Separate component | Separate component |

### VIPER Components
```
View ↔ Presenter ↔ Interactor → Entity
         ↓
       Router
```

### VIP Cycle (Unidirectional)
```
View → Interactor → Presenter → View
         ↓
    Worker/Service
```

**Key insight:** VIPER/VIP are *implementations* of Clean Architecture principles for iOS modules.

---

## Common Interview Questions

### Q: Why use Clean Architecture?
- **Testability**: Inject mocks at any layer boundary
- **Maintainability**: Changes isolated to specific layers
- **Scalability**: Teams can work on different layers
- **Framework independence**: Swap UIKit↔SwiftUI without touching domain

### Q: What goes in a Use Case vs ViewModel?
- **Use Case**: Business rules, data transformation, orchestration
- **ViewModel**: UI state, formatting, user interaction handling

### Q: How do you handle dependency injection?
```swift
// Composition Root (AppDelegate/SceneDelegate)
let apiClient = APIClient()
let userRepo = UserRepositoryImpl(apiClient: apiClient)
let fetchUserUC = FetchUserUseCase(repository: userRepo)
let viewModel = UserViewModel(fetchUserUseCase: fetchUserUC)
```

### Q: Where does navigation logic live?
- **Coordinator pattern** at the presentation layer
- Or **Router** (VIPER-style) injected into ViewModel

### Q: How to avoid over-engineering?
- Don't create Use Case for every screen — only for reusable business logic
- Small apps: skip Interface Adapters, go Domain → View
- Pragmatic approach: apply where complexity justifies it

---

## Red Flags to Avoid
❌ ViewModels calling network directly  
❌ Entities importing UIKit  
❌ Use Cases knowing about ViewModels  
❌ Circular dependencies between layers  
❌ "Manager" classes doing everything  

## Quick Wins to Mention
✅ Protocol-driven boundaries  
✅ Repository pattern for data access  
✅ Mappers between layers (DTO → Entity → ViewModel)  
✅ Dependency injection at composition root  
