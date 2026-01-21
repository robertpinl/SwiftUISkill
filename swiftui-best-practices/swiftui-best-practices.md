# SwiftUI Best Practices Skill

Use this skill when writing or reviewing SwiftUI code to ensure best practices are followed.

## Data Flow and State Management

### Property Wrapper Selection Guide

| Wrapper | Use When |
|---------|----------|
| `@State` | Internal view state that triggers updates; should be `private` |
| `@Binding` | Child view needs to modify parent's state |
| `@StateObject` | View owns an `ObservableObject` instance (pre-iOS 17) |
| `@ObservedObject` | View receives an `ObservableObject` from outside |
| `@Bindable` | iOS 17+: View receives `@Observable` object and needs bindings |
| `let` | Read-only value passed from parent |
| `var` | Read-only value that child observes via `.onChange()` |

### @State Rules

- Always mark `@State` properties as `private`
- Use for internal view state that triggers UI updates
- iOS 17+: Can use with `@Observable` classes instead of `@StateObject`

```swift
// Correct
@State private var isAnimating = false

// iOS 17+ with @Observable
@Observable
final class DataModel {
    var name = "Some Name"
}

struct MyView: View {
    @State private var model = DataModel()  // Use @State, not @StateObject
    var body: some View {
        TextField("Name", text: $model.name)
    }
}
```

### @Binding Rules

- Use only when child view needs to **modify** parent's state
- If child only reads the value, use `let` instead

```swift
struct ChildView: View {
    @Binding var isSelected: Bool  // Will modify this
    var body: some View {
        Button("Select") { isSelected = true }
    }
}
```

### @StateObject vs @ObservedObject

- `@StateObject`: View **creates** the object
- `@ObservedObject`: View **receives** the object from outside

```swift
// View creates it → @StateObject
@StateObject private var viewModel = MyViewModel()

// View receives it → @ObservedObject
@ObservedObject var viewModel: MyViewModel
```

### @Bindable (iOS 17+)

Use when receiving an `@Observable` object from outside and needing bindings:

```swift
struct MyView: View {
    @Bindable var model: DataModel  // Received from parent, needs bindings
    var body: some View {
        TextField("Name", text: $model.name)
    }
}
```

### let vs var for Passed Values

- `let`: Child only reads the value
- `var`: Child needs `.onChange()` to react to changes

```swift
// Read-only display
struct DisplayView: View {
    let title: String  // Never modified
}

// Needs to react to changes
struct ReactiveView: View {
    var externalValue: Int  // Watch with .onChange()
    @State private var internalState = ""

    var body: some View {
        Text(internalState)
            .onChange(of: externalValue) { newValue in
                internalState = "Value: \(newValue)"
            }
    }
}
```

## View Structure and Composition

### Extract Subviews, Not Computed Properties

**Bad**: Using `@ViewBuilder` functions or computed properties for complex views:

```swift
// Bad - re-executes on every parent state change
struct ParentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Button("Tap") { count += 1 }
            complexSection()  // Re-executes every tap!
        }
    }

    @ViewBuilder
    func complexSection() -> some View {
        // 100+ lines of views...
    }
}
```

**Good**: Extract to separate `struct` views:

```swift
// Good - subview body skipped when its inputs don't change
struct ParentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Button("Tap") { count += 1 }
            ComplexSection()  // Body SKIPPED during re-execution
        }
    }
}

struct ComplexSection: View {
    var body: some View {
        // 100+ lines of views...
    }
}
```

### When to Use @ViewBuilder Functions

Only for simple, readable sections that don't affect performance:

```swift
struct SimpleView: View {
    var body: some View {
        VStack {
            headerSection()  // OK for small, simple sections
            mainContent()
        }
    }

    @ViewBuilder
    func headerSection() -> some View {
        HStack {
            Text("Title")
            Spacer()
            Button("Back") { }
        }
    }
}
```

### Rule of Thumb

> When a view exceeds 100 lines of primitive views, extract subviews for better performance.

## Performance Optimization

### 1. Avoid Redundant State Updates

SwiftUI doesn't compare values before triggering updates:

```swift
// Bad - triggers update even if value unchanged
.onReceive(publisher) { value in
    self.currentValue = value
}

// Good - only update when different
.onReceive(publisher) { value in
    if self.currentValue != value {
        self.currentValue = value
    }
}
```

### 2. Avoid Closure-Based Content Properties

Closures can't be compared, causing unnecessary re-renders:

```swift
// Bad - closure can't be compared
struct MyContainer<Content: View>: View {
    let content: () -> Content  // Causes re-render on every parent update
}

// Good - view can be compared
struct MyContainer<Content: View>: View {
    @ViewBuilder let content: Content  // Only re-renders when content changes
}
```

### 3. Optimize Hot Paths

Hot paths (frequently executed code) must be efficient:

```swift
// Bad - updates state on every scroll position change
.onPreferenceChange(ScrollOffsetKey.self) { offset in
    shouldShowTitle = offset.y <= -32  // Fires constantly!
}

// Good - only update when threshold crossed
.onPreferenceChange(ScrollOffsetKey.self) { offset in
    let shouldShow = offset.y <= -32
    if shouldShow != shouldShowTitle {
        shouldShowTitle = shouldShow  // Fires only on threshold change
    }
}
```

## Quick Reference Checklist

When reviewing SwiftUI code, verify:

- [ ] `@State` properties are `private`
- [ ] `@Binding` is only used when child modifies parent state
- [ ] `@StateObject` for owned objects, `@ObservedObject` for injected
- [ ] Complex views (100+ lines) are extracted to separate structs
- [ ] `@ViewBuilder` functions only used for simple, non-performance-critical sections
- [ ] State updates check for value changes before assigning
- [ ] Container views use `@ViewBuilder let content: Content` not closures
- [ ] Hot paths (scroll handlers, etc.) minimize state updates
