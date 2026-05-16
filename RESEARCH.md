# Swift 6 Concurrency: What's Most Likely to Bite a SwiftUI App

A research note focused on the concurrency changes in Swift 6 that tend to
break existing SwiftUI codebases when you flip the language mode. Scope: an
app that already exists, compiles under Swift 5, and uses SwiftUI plus the
typical `ObservableObject` / `@StateObject` / `Task { }` patterns.

## TL;DR

The big shift is that data-race checks that used to be warnings (or off
entirely) are now compile errors. Most of the pain in a SwiftUI app
clusters around four things:

1. The whole `View` protocol is now `@MainActor`, but the types and
   closures it touches often aren't — so isolation mismatches surface at
   the boundary.
2. `@StateObject` / `@ObservedObject` no longer drag `@MainActor` onto
   their enclosing type (SE-0401). View models that used to get main-actor
   isolation "for free" no longer do.
3. Anything captured by a `Task { }` or passed across an `async` boundary
   has to be `Sendable`, and most existing model/view-model classes are
   not.
4. Closures lose their isolation when they cross into a `@Sendable`
   context, producing the now-infamous "main actor-isolated property can
   not be referenced from a Sendable closure" diagnostic.

The rest of this doc walks through each in more detail with the actual
proposal/source links.

## 1. Strict concurrency is on by default

Under Swift 6, `-strict-concurrency=complete` is effectively the floor.
Diagnostics that were warnings under Swift 5 (even with the flag) become
errors when `swift-tools-version` / `SWIFT_VERSION` is 6. The recommended
migration path is to leave the project on Swift 5 but turn on Complete
strict-concurrency checking, fix the warnings module by module, and only
then flip the language mode.

- [Apple — Adopting strict concurrency in Swift 6 apps](https://developer.apple.com/documentation/swift/adoptingswift6)
- [Swift.org — Data Race Safety (migration guide)](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/dataracesafety/)
- [SwiftLee — Swift 6: What's New and How to Migrate](https://www.avanderlee.com/concurrency/swift-6-migrating-xcode-projects-packages/)

## 2. `View` is `@MainActor` — and that's contagious

In Xcode 16 / Swift 6, the entire `SwiftUI.View` protocol is annotated
`@MainActor`. Every conforming type, every `body`, every method on the
view, and every stored property accessed from `body` is now main-actor
isolated. This is mostly a good thing — it matches reality — but it
creates a sharp edge: anything the view *calls* must either also be
main-actor or be reachable across an `await`.

What this means in practice for an existing app:

- Helper methods on a view that did `DispatchQueue.global().async { ... }`
  and then touched `self` need to be reworked. The closure escapes the
  main actor, so touching `self` is a data race.
- A view that calls into a non-isolated singleton (e.g. a `Logger.shared`
  or a `NetworkClient` that isn't actor-isolated) usually still works,
  but anything that mutates view state from inside that singleton's
  callbacks now needs `await MainActor.run { ... }` or a `@MainActor`
  closure.
- Computed properties on a view that wrap calls to non-isolated
  synchronous APIs are fine; computed properties that *await* something
  are not allowed and were never allowed — but the error message is
  clearer now.

References:
- [fatbobman — SwiftUI Views and @MainActor](https://fatbobman.com/en/posts/swiftui-views-and-mainactor/)
- [Apple Developer Forums — Task Isolation Inheritance and SwiftUI](https://developer.apple.com/forums/thread/761150)

## 3. SE-0401: property wrappers no longer infer actor isolation

This is the change most likely to surprise a maintainer who hasn't
followed the evolution proposals.

Under Swift 5.5+, using a property wrapper whose `wrappedValue` was
`@MainActor` would silently confer `@MainActor` on the enclosing type.
So this view model:

```swift
class CounterModel: ObservableObject {
    @Published var count = 0
    func increment() { count += 1 }
}
```

…was *not* main-actor-isolated itself, but a `View` that held it via
`@StateObject` got `@MainActor` for free because `@StateObject`'s wrapped
value is `@MainActor`. SE-0401 removes that inference. The view is still
`@MainActor` in Swift 6 (see §2, because `View` itself is annotated), but
classes that relied on the same trick — anything using `@StateObject`,
`@ObservedObject`, or other `@MainActor` property wrappers in a
non-`View` context — lose it.

The fix is mechanical: annotate the type explicitly.

```swift
@MainActor
class CounterModel: ObservableObject { ... }
```

References:
- [SE-0401 — Remove Actor Isolation Inference caused by Property Wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md)
- [Massicotte — SE-0401 walkthrough](https://www.massicotte.org/concurrency-swift-6-se-401)
- [Hacking with Swift — Complete concurrency enabled by default](https://www.hackingwithswift.com/swift/6.0/concurrency)

## 4. Sendable: model types that cross actors

`Sendable` was opt-in advice in Swift 5 and is enforced in Swift 6. The
rules that bite hardest in a SwiftUI app:

- **Reference types**: a `class` must be `final` and have only immutable
  stored properties (or be an `actor`) to conform to `Sendable`. Most
  view models don't qualify. Common workaround: isolate them to a global
  actor (usually `@MainActor`) so the compiler knows access is
  serialized, in which case they get `Sendable` "for free" via actor
  isolation.
- **Closures crossing task boundaries**: completion handlers stored on a
  non-`Sendable` class and then invoked from a `Task` will fail to
  compile. Either make the handler `@Sendable` (and then it can't
  capture non-`Sendable` state), or wrap the entry point in
  `Task { @MainActor in ... }`.
- **Views are not `Sendable`**: passing a `View` value across a task
  boundary is rejected. This usually only hurts code that built views in
  background work — uncommon, but it happens with snapshotting /
  rendering pipelines.
- **`@unchecked Sendable` is the escape hatch**, not the answer. It
  silences the compiler and re-introduces the data race. Reserve it for
  types you've genuinely synchronized by hand (e.g. an internal lock).

References:
- [SwiftLee — Sendable and @Sendable closures explained](https://www.avanderlee.com/swift/sendable-protocol-closures/)
- [Wesley Matlock — Mastering Sendable in Swift 6](https://medium.com/@wesleymatlock/mastering-sendable-in-swift-6-e13d04d86820)
- [fatbobman — Sendable, @unchecked Sendable, @Sendable, sending and nonsending](https://fatbobman.com/en/posts/sendable-sending-nonsending/)

## 5. The "main actor-isolated property can not be referenced from a Sendable closure" error

This is the diagnostic everyone hits. The setup: you're in a `@MainActor`
view, you start a `Task { }`, and inside it you touch `self.something`.
Under Swift 5 this worked because `Task` inherited the enclosing
isolation. Under Swift 6 the closure passed to `Task` is `@Sendable`,
which strips main-actor isolation, so the capture is a race.

The fixes, ordered by preference:

1. `Task { @MainActor in ... }` — declare the closure main-actor-isolated.
2. `await MainActor.run { ... }` for a tighter scope inside a non-isolated
   `Task`.
3. `MainActor.assumeIsolated { ... }` if you can prove statically that
   you're already on the main thread (e.g. inside a UIKit delegate
   callback) and want to avoid the implicit hop.
4. Annotate the *captured method* itself with `@MainActor` so the
   compiler propagates isolation through the call.

Avoid: dropping the work into a `@Sendable` closure and silencing
warnings — that's the actual race the compiler is warning about.

References:
- [Donny Wals — Solving "Main actor-isolated property can not be referenced from a Sendable closure"](https://www.donnywals.com/solving-main-actor-isolated-property-can-not-be-referenced-from-a-sendable-closure-in-swift/)
- [Jesse Squires — Swift concurrency hack for passing non-sendable closures](https://www.jessesquires.com/blog/2024/06/05/swift-concurrency-non-sendable-closures/)
- [Donny Wals — Solving "Capture of non-sendable type in @Sendable closure"](https://www.donnywals.com/solving-capture-of-non-sendable-type-in-sendable-closure-in-swift/)

## 6. Less obvious sharp edges

A grab bag of issues that show up in real migrations but get less
attention than the headline ones:

- **`onAppear` / `task` modifiers**: `.task { ... }` inherits the view's
  main-actor isolation; a raw `Task { ... }` inside `onAppear` does not.
  Mixed usage in one file is a common source of confusion.
- **Protocol conformances**: if a protocol method is non-isolated and
  your conforming type is `@MainActor`, the conformance is rejected.
  Either mark the protocol requirement `@MainActor`, mark the
  implementation `nonisolated`, or hop inside.
- **Delegate callbacks from UIKit/AppKit**: many delegate methods are
  declared non-isolated. If your delegate is `@MainActor`, you'll need
  `MainActor.assumeIsolated` or a non-isolated thunk.
- **Singletons**: `static let shared` properties on non-`Sendable`
  classes are now an error. The two clean fixes are to make the class
  `@MainActor` or to make it an `actor`.
- **`@Published` and background mutation**: any code that mutated a
  `@Published` property off the main thread "worked" in Swift 5 (with a
  runtime purple warning in debug). In Swift 6 it won't compile if the
  enclosing type is `@MainActor`, which is now the expected shape.
- **Tests**: `XCTestCase` is not `@MainActor`. Test methods that touch
  main-actor types need `@MainActor` on the method or the whole class.

References:
- [Quality Coding — XCTest Meets @MainActor](https://qualitycoding.org/xctest-mainactor/)
- [jano.dev — More Swift 6 Migration Errors](https://jano.dev/apple/macos/swift/2025/03/09/Swift-6-Migration-Errors.html)
- [BrightDigit — Swift 6 Incomplete Migration Guide for Dummies](https://brightdigit.com/tutorials/swift-6-async-await-actors-fixes/)
- [TelemetryDeck — Migrating the TelemetryDeck SDK to Swift 6](https://telemetrydeck.com/blog/migrating-to-swift-6/)

## 7. Swift 6.2 / Xcode 26: "Approachable Concurrency"

Worth knowing about because it changes the recommended defaults rather
than the language semantics.

- **SE-0466 — Default Actor Isolation**: lets a module declare that
  unannotated code runs on `MainActor` by default. New Xcode 26 projects
  enable this; existing projects keep `nonisolated` as the default to
  preserve behavior.
- **SE-0461 — Nonisolated Async Functions**: non-isolated `async`
  functions now run on the caller's actor instead of hopping to the
  cooperative pool, which removes a whole class of accidental hops in
  SwiftUI code.
- **"Approachable Concurrency" build setting**: a single switch that
  turns on a curated bundle of the friendlier defaults.

If you're starting the migration today, decide early whether you'll
opt in to default `MainActor` isolation per module. Turning it on later
flips the meaning of every unannotated function, which is a much bigger
diff than the original Swift 6 migration.

References:
- [SwiftLee — Default Actor Isolation in Swift 6.2](https://www.avanderlee.com/concurrency/default-actor-isolation-in-swift-6-2/)
- [Donny Wals — Setting default actor isolation in Xcode 26](https://www.donnywals.com/setting-default-actor-isolation-in-xcode-26/)
- [Donny Wals — Should you opt-in to Swift 6.2's Main Actor isolation?](https://www.donnywals.com/should-you-opt-in-to-swift-6-2s-main-actor-isolation/)
- [Hacking with Swift — What's new in Swift 6.2?](https://www.hackingwithswift.com/articles/277/whats-new-in-swift-6-2)
- [NativeFirst — Swift 6.2 Approachable Concurrency migration guide](https://nativefirstapp.com/blog/swift-6-2-approachable-concurrency-migration-guide/)

## Suggested migration order for a SwiftUI app

1. Stay on Swift 5; set Strict Concurrency Checking to **Complete**.
   Fix all warnings.
2. Annotate view models, routers, and other UI-adjacent classes with
   `@MainActor` explicitly (don't rely on SE-0401-removed inference).
3. Audit `Task { }` call sites. Replace bare `Task { }` inside views
   with `.task { }` or `Task { @MainActor in ... }`.
4. Make model value types `Sendable`. For reference types, prefer
   `@MainActor` or `actor` over `@unchecked Sendable`.
5. Move utility / networking modules to Swift 6 first; flip UI targets
   last.
6. Decide separately whether to adopt Swift 6.2's default `MainActor`
   isolation — and if you do, do it per-module, not project-wide in one
   PR.

## Sources

- [Apple — Adopting strict concurrency in Swift 6 apps](https://developer.apple.com/documentation/swift/adoptingswift6)
- [Apple — Migrate your app to Swift 6 (WWDC24)](https://developer.apple.com/videos/play/wwdc2024/10169/)
- [Swift.org — Data Race Safety migration guide](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/dataracesafety/)
- [SE-0401 — Remove Actor Isolation Inference caused by Property Wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md)
- [Hacking with Swift — Complete concurrency enabled by default](https://www.hackingwithswift.com/swift/6.0/concurrency)
- [Hacking with Swift — What's new in Swift 6.2?](https://www.hackingwithswift.com/articles/277/whats-new-in-swift-6-2)
- [SwiftLee — Swift 6: What's New and How to Migrate](https://www.avanderlee.com/concurrency/swift-6-migrating-xcode-projects-packages/)
- [SwiftLee — Default Actor Isolation in Swift 6.2](https://www.avanderlee.com/concurrency/default-actor-isolation-in-swift-6-2/)
- [SwiftLee — Sendable and @Sendable closures](https://www.avanderlee.com/swift/sendable-protocol-closures/)
- [Donny Wals — Solving "Main actor-isolated property can not be referenced from a Sendable closure"](https://www.donnywals.com/solving-main-actor-isolated-property-can-not-be-referenced-from-a-sendable-closure-in-swift/)
- [Donny Wals — Setting default actor isolation in Xcode 26](https://www.donnywals.com/setting-default-actor-isolation-in-xcode-26/)
- [Donny Wals — What is Approachable Concurrency in Xcode 26?](https://www.donnywals.com/what-is-approachable-concurrency-in-xcode-26/)
- [Massicotte — SE-0401 walkthrough](https://www.massicotte.org/concurrency-swift-6-se-401)
- [fatbobman — SwiftUI Views and @MainActor](https://fatbobman.com/en/posts/swiftui-views-and-mainactor/)
- [fatbobman — Sendable, @unchecked Sendable, @Sendable, sending and nonsending](https://fatbobman.com/en/posts/sendable-sending-nonsending/)
- [Quality Coding — XCTest Meets @MainActor](https://qualitycoding.org/xctest-mainactor/)
- [Quality Coding — A Conversation With Swift 6 About Data Race Safety](https://qualitycoding.org/conversation-swift6-data-race-safety/)
- [Jesse Squires — Swift concurrency hack for passing non-sendable closures](https://www.jessesquires.com/blog/2024/06/05/swift-concurrency-non-sendable-closures/)
- [TelemetryDeck — Migrating the TelemetryDeck SDK to Swift 6](https://telemetrydeck.com/blog/migrating-to-swift-6/)
- [BrightDigit — Swift 6 Incomplete Migration Guide for Dummies](https://brightdigit.com/tutorials/swift-6-async-await-actors-fixes/)
- [jano.dev — More Swift 6 Migration Errors](https://jano.dev/apple/macos/swift/2025/03/09/Swift-6-Migration-Errors.html)
- [Apple Developer Forums — Task Isolation Inheritance and SwiftUI](https://developer.apple.com/forums/thread/761150)
