# Swift Macros (5.9+) — iOS Interview Cheatsheet

## What Are Swift Macros?
Compile-time code generation that transforms syntax → expanded code.  
**Key benefit**: Reduce boilerplate while keeping generated code visible (right-click → "Expand Macro").

---

## Two Types of Macros

### 1. Freestanding Macros
Start with `#` — standalone expressions or declarations.

```swift
// Expression macro — returns a value
let url = #URL("https://api.example.com")  // Validated at compile time!

// Declaration macro — generates new declarations
#warning("TODO: Implement this")
#Preview { ContentView() }
```

### 2. Attached Macros
Start with `@` — attached to declarations, modify/add to them.

```swift
@Observable        // Adds observation infrastructure
class UserModel {
    var name: String = ""  // Becomes tracked property
}

@Model             // SwiftData: adds persistence
class Item { ... }
```

---

## Built-in Macros You Must Know

### @Observable (Observation framework, iOS 17+)
Replaces Combine's `ObservableObject` + `@Published`.

```swift
// OLD (Combine)
class UserModel: ObservableObject {
    @Published var name: String = ""
}

// NEW (Observation)
@Observable
class UserModel {
    var name: String = ""     // Automatically observed
    var age: Int = 0          // Automatically observed
}

// Usage in SwiftUI — no @ObservedObject needed!
struct ContentView: View {
    var user: UserModel  // Just a regular property
    
    var body: some View {
        Text(user.name)  // View updates when name changes
    }
}
```

**What @Observable generates:**
- `@ObservationTracked` on each stored property
- Conformance to `Observable` protocol
- Internal tracking machinery

### #Preview (Xcode 15+)
Replaces `PreviewProvider` structs.

```swift
// OLD
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}

// NEW — much cleaner!
#Preview {
    ContentView()
}

// Named preview
#Preview("Dark Mode") {
    ContentView()
        .preferredColorScheme(.dark)
}

// UIKit preview
#Preview {
    let vc = MyViewController()
    return vc
}
```

---

## Creating Custom Macros

### Macro Roles (Types)

| Role | What it does | Example |
|------|--------------|---------|
| `@freestanding(expression)` | Returns a value | `#stringify(x)` |
| `@freestanding(declaration)` | Creates declarations | `#makeEnum(...)` |
| `@attached(peer)` | Adds sibling declarations | Add async version of func |
| `@attached(member)` | Adds members to type | Add init, properties |
| `@attached(accessor)` | Adds get/set/willSet | Custom property wrappers |
| `@attached(memberAttribute)` | Adds attributes to members | Add @Published to all |
| `@attached(conformance)` | Adds protocol conformance | Auto-conform to Codable |

### Basic Custom Macro Structure

```swift
// 1. DECLARATION (in main module)
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) = 
    #externalMacro(module: "MyMacros", type: "StringifyMacro")

// 2. IMPLEMENTATION (in macro plugin target)
import SwiftSyntax
import SwiftSyntaxMacros

public struct StringifyMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) throws -> ExprSyntax {
        guard let argument = node.argumentList.first?.expression else {
            throw CustomError.noArgument
        }
        return "(\(argument), \(literal: argument.description))"
    }
}

// 3. USAGE
let (value, code) = #stringify(2 + 3)
// Expands to: (2 + 3, "2 + 3")
```

### Project Structure for Custom Macros
```
MyApp/
├── Sources/
│   ├── MyApp/           # Main app
│   └── MyMacros/        # Macro implementations (Swift Package Plugin)
├── Tests/
│   └── MyMacroTests/    # Test expansions
└── Package.swift
```

---

## Common Interview Questions

### Q: What's the difference between macros and property wrappers?
| Macros | Property Wrappers |
|--------|-------------------|
| Compile-time | Runtime |
| Generate real code | Wrap values |
| Can see expanded code | Implementation hidden |
| More flexible | Simpler for basic cases |

### Q: When would you create a custom macro?
- Reducing repetitive boilerplate (init generation, Codable keys)
- Compile-time validation (URL strings, regex patterns)
- Adding conformances with custom logic
- Code generation that property wrappers can't handle

### Q: How does @Observable improve over @Published?
- **No inheritance required** (no `ObservableObject`)
- **Granular tracking** — only views using changed properties update
- **Simpler syntax** — no `@Published` on every property
- **Better performance** — less overhead than Combine

### Q: Can macros access runtime values?
**No!** Macros only see syntax (AST) at compile time. They can't:
- Read variable values
- Access network/database
- Use runtime reflection

### Q: How do you test macros?
```swift
import SwiftSyntaxMacrosTestSupport

func testStringify() {
    assertMacroExpansion(
        #"#stringify(2 + 3)"#,
        expandedSource: #"(2 + 3, "2 + 3")"#,
        macros: ["stringify": StringifyMacro.self]
    )
}
```

---

## Red Flags
❌ Using macros for simple things (property wrapper suffices)  
❌ Not understanding that macros are compile-time only  
❌ Thinking macros can do reflection  

## Points to Impress
✅ Explain macro roles clearly  
✅ Know that expanded code is inspectable  
✅ Understand SwiftSyntax basics (AST manipulation)  
✅ Mention hygiene (macros create unique names to avoid conflicts)  
