# Swift Concurrency Q&A

**Source:** [Q&A: Swift Concurrency, Formatted — Anton Gubarenko](https://antongubarenko.substack.com/p/q-and-a-swift-concurrency-formatted?utm_source=fatbobman%20weekly%20issue%20134&utm_medium=web)

---

## 1. `nonisolated` vs `nonisolated(nonsending)`

### Q
What's the difference between `nonisolated` and `nonisolated(nonsending)`, and why was the latter introduced?

### A
Nonisolated means that a declaration has no preference for isolation. It does not have a fixed isolation, so it can be used in multiple isolation contexts. Nonisolated non-sending is about asynchronous functions. There was a behavior originally implemented for asynchronous functions in Swift concurrency where asynchronous functions would move to the concurrent thread pool, the shared executor, by default. We found that this was not the right trade-off as people adopted Swift concurrency and we learned more. So nonisolated non-sending, which is now the default behavior if you opt in using the approachable concurrency settings, is the new behavior where your asynchronous task or asynchronous function will not leave the isolation it was called from right away, unless there is an explicit reason to.

### Explain
The key thing to grasp: this only matters for **async** functions. For sync functions, `nonisolated` behaves the same in both worlds.

**The old behavior (plain `nonisolated async`)**

When you call a `nonisolated` async function, Swift hops off your actor to the global concurrent pool, runs the function there, then hops back.

```swift
@MainActor
class ViewModel {
    var items: [String] = []

    func load() async {
        // We're on MainActor here.
        let data = await fetcher.fetch()  // ← hops OFF MainActor
        // Back on MainActor here.
        items = data
    }
}

class Fetcher {
    nonisolated func fetch() async -> [String] {
        // This runs on the global concurrent pool, NOT MainActor.
        return ["a", "b"]
    }
}
```

Why this was painful:
1. **Sendable headaches** — anything you pass into `fetch()` had to be Sendable, because it was crossing isolation boundaries.
2. **Surprising hops** — looks like a normal call, but actually causes a thread switch.
3. **Worse perf in many cases** — you didn't need the work off the main actor, but you paid the hop cost anyway.

**The new behavior (`nonisolated(nonsending)`)**

The async function stays on the caller's actor unless it explicitly opts out.

```swift
class Fetcher {
    nonisolated(nonsending) func fetch() async -> [String] {
        // If called from MainActor → runs on MainActor.
        // If called from some other actor → runs on that actor.
        // If called from nowhere (a Task) → runs nonisolated.
        return ["a", "b"]
    }
}
```

Now `await fetcher.fetch()` from a `@MainActor` context does not leave MainActor. No hop, no Sendable check on the arguments, no surprise.

**When you actually want to hop off**

If `fetch()` does heavy CPU work and you want it off MainActor, be explicit:

```swift
class Fetcher {
    @concurrent
    func heavyWork() async -> [String] {
        // Explicitly runs on the global concurrent pool.
    }
}
```

Or use `Task.detached { ... }` at the call site.

**TL;DR**

| Form | Async function runs on… | Use when |
|---|---|---|
| `nonisolated` (sync) | caller's context (no hop) | normal sync helpers |
| `nonisolated(nonsending)` async | caller's actor (no hop) | new default — most async APIs |
| `@concurrent` async | global concurrent pool (hop) | you genuinely want to leave the actor |

The reason for the change: the old default was the wrong default. Most async functions don't need to leave the caller's actor, and forcing the hop made Sendable errors and threading behavior much harder to reason about. Under "Approachable Concurrency" (Swift 6.x), `nonisolated(nonsending)` becomes the default for async, so you usually don't even write it — you write `nonisolated` and get the new behavior.

---

## 2. Heavy I/O Inside a `@MainActor` ViewModel

### Q
When marking a ViewModel with `@MainActor`, how should we handle a method inside it that performs heavy I/O processing? How do we process outside of MainActor / main thread?

### A
Yes, if you have a particular operation on any MainActor-isolated type and you want only that operation to be offloaded, you can annotate just that method with a different isolation. For example, if you want to offload it to the concurrent thread pool, you can now use the `@concurrent` attribute. That is the right explicit spelling for it. The compiler will prevent you from directly accessing any MainActor mutable state on that view model, or whatever your MainActor-isolated type is, to make sure you are not accessing that state at the same time as another operation on that type is accessing the same mutable state from the MainActor. So take a look at `@concurrent`. There are also a lot of references from last year's WWDC, where many uses of `@concurrent` were covered as well.

### Explain
**The problem**

You have a `@MainActor` ViewModel. Every method runs on the main thread. But one method does something heavy — image processing, parsing a big JSON, hashing a file. You don't want that blocking the UI.

```swift
@MainActor
final class ImageViewModel {
    var thumbnail: UIImage?

    func makeThumbnail(from data: Data) async {
        // ⚠️ Runs on MainActor → blocks the UI while it crunches.
        let image = expensiveResize(data)
        thumbnail = image
    }
}
```

**The fix: `@concurrent`**

`@concurrent` is the explicit opt-out. It tells the compiler: "this one method should run on the global concurrent pool, not on MainActor."

```swift
@MainActor
final class ImageViewModel {
    var thumbnail: UIImage?

    @concurrent
    func makeThumbnail(from data: Data) async {
        // ✅ Runs OFF MainActor, on the global pool.
        let image = expensiveResize(data)

        // ❌ Compiler error: can't touch MainActor state from here.
        // thumbnail = image

        // ✅ Hop back to MainActor to write state.
        await MainActor.run {
            self.thumbnail = image
        }
    }
}
```

**What the WWDC quote was trying to say**

Three pieces:

1. **Default** — methods on a `@MainActor` type run on MainActor.
2. **`@concurrent` opts ONE method off** — without removing `@MainActor` from the whole class.
3. **The compiler protects you** — once that method is off-actor, it physically cannot read or write `self.thumbnail` directly. You must `await MainActor.run { ... }` (or call back into a MainActor method). This prevents data races on the ViewModel's mutable state.

**Why this is better than the old tricks**

Old patterns people used:

```swift
// Old: Task.detached
func makeThumbnail(from data: Data) async {
    let image = await Task.detached {
        expensiveResize(data)  // data must be Sendable
    }.value
    thumbnail = image
}

// Old: a separate nonisolated helper
nonisolated func resize(_ data: Data) -> UIImage { ... }
```

Both work, but they push the off-actor work into a closure or a separate function. `@concurrent` lets the whole method body be the off-actor work, while still being a method on your ViewModel — cleaner shape, same safety.

**The mental model**

| Annotation on a method of `@MainActor` class | Where the body runs |
|---|---|
| (none) | MainActor (main thread) |
| `nonisolated` (sync) | Caller's context |
| `nonisolated` async (Swift 6.2 default) | Caller's actor — usually still MainActor |
| `@concurrent` async | Global concurrent pool — definitely off MainActor |

Rule of thumb: for a heavy async method on a MainActor type, mark it `@concurrent`, do the work, then `await MainActor.run { ... }` to write results back.

---

## 3. Task Created From `@MainActor` and UI Hitches

### Q
When a Task is created from `@MainActor`, will that block the main thread and cause UI hitches?

### A
A task being created from MainActor is something you probably encounter often when working on a UI framework. I think that when a task is created on MainActor, there is still a suspension point that the compiler creates, so while that work is being performed, the main thread can still continue working on UI updates and avoid causing hitches. In general, the task machinery tries to schedule your work on the concurrent thread pool so that it does not block excessively. That is one of the nice behaviors that tasks provide. I think it depends on what the task does. If you create a task that is isolated to MainActor and that task starts running on MainActor, and you are doing some very expensive work that takes a long time, then it might block MainActor, and you might notice that. In that case, you may want to profile that code in Instruments and then decide whether to insert some suspension points or offload parts of that expensive task from MainActor. I really recommend that you do not worry about proactively offloading everything you can from MainActor. There are some things that are totally fine to run on MainActor. But if you notice hangs in your app and identify that piece of code through Instruments or your profiling tool of choice, that is when you should make targeted changes to move the expensive work off.

### Explain
**Short answer**

Creating a `Task` is cheap and never blocks. What's *inside* it can block — but only if it does heavy synchronous work without `await` points.

**What actually happens**

```swift
@MainActor
final class VM {
    var label = ""

    func tapped() {
        Task {
            // This Task is MainActor-isolated (inherits from VM).
            // It will RUN on the main thread.
        }
    }
}
```

Three cases for what's inside:

**Case 1 — Has `await` points → fine ✅**

```swift
Task {
    let data = await api.fetch()   // ← suspends here
    label = "got \(data.count)"
}
```

At the `await`, the main thread is released. UI keeps animating, scrolling, responding to taps. When `fetch()` returns, the task resumes on main, sets `label`, done. No hitch.

**Case 2 — Pure heavy sync work, no `await` → blocks ❌**

```swift
Task {
    // No suspension points. Main thread is held the whole time.
    let result = primesUpTo(10_000_000)   // 2 seconds of CPU
    label = "\(result.count)"
}
```

This is identical to just running the code on main directly. The `Task { }` wrapper does not magically move it off the main thread. UI freezes for 2 seconds.

**Case 3 — Want to offload the heavy bit → use `@concurrent` or `Task.detached`**

```swift
@concurrent
func compute() async -> Int {
    primesUpTo(10_000_000).count    // runs on global pool
}

Task {
    let count = await compute()     // suspends main, main stays responsive
    label = "\(count)"              // back on main, fast
}
```

**What the WWDC speaker meant**

- "Task machinery tries to schedule your work on the concurrent thread pool" → true for `Task.detached` and `@concurrent`/`nonisolated async` calls. Not true for plain `Task { }` from MainActor — that one stays on main.
- "Compiler creates a suspension point" → he means the `await` points you write. The compiler doesn't insert them for you.
- "Don't proactively offload everything" → most code on main is fine (setting properties, small calculations, layout work). Only profile-then-fix when you see actual hangs.

**Rule of thumb**

| What's in the Task | Blocks main? |
|---|---|
| `await someAsyncCall()` | No |
| Tiny sync work (set a property, format a string) | No, imperceptible |
| Tight loop / parsing big JSON / image work / hashing | Yes — offload it |

So: don't be afraid of `Task { ... }` from MainActor. Be afraid of heavy sync code with no `await`, wherever it lives.

---

## 4. `@concurrent` on a Custom Actor

### Q
We've focused on `@concurrent` inside `@MainActor` so far. Is using `@concurrent` inside a custom actor meaningful?

### Explain
Yes — `@concurrent` works on any actor (custom actor, custom global actor, `@MainActor`). It's the same mechanism. The interesting question is **when** it's worth doing.

**What makes a custom actor different from MainActor**

| | MainActor | Custom actor |
|---|---|---|
| Backed by | Main thread | A serial executor (some background queue) |
| Blocking it freezes UI? | Yes | No |
| Blocking it hurts throughput of other callers? | Yes | Yes — other actor calls queue up |

So the urgency for offloading is lower for custom actors, but the same kind of problem exists: a sync-heavy method monopolizes the actor and stalls everyone else waiting on it.

**A meaningful example**

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) async -> UIImage? {
        if let hit = cache[url] { return hit }
        let data = try? await fetch(url)
        guard let data else { return nil }

        let decoded = decodeSync(data)   // ⚠️ CPU-heavy, runs ON the actor
        cache[url] = decoded
        return decoded
    }

    private func decodeSync(_ data: Data) -> UIImage? {
        UIImage(data: data)              // expensive
    }
}
```

If 10 callers ask for different images at once, they all queue behind each other because `decodeSync` is sync work holding the actor.

Fix:

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) async -> UIImage? {
        if let hit = cache[url] { return hit }
        let data = try? await fetch(url)
        guard let data else { return nil }

        let decoded = await decode(data)  // hops off the actor
        cache[url] = decoded
        return decoded
    }

    @concurrent
    private func decode(_ data: Data) async -> UIImage? {
        UIImage(data: data)              // runs on global pool
    }
}
```

Now while one image is decoding on the global pool, the actor is free to handle other `image(for:)` calls — check the cache, kick off more fetches, etc. Throughput goes up.

**When `@concurrent` on a custom actor is meaningful**

✅ **Heavy CPU work that doesn't need the actor's state** — decoding, hashing, parsing, transforming bytes, image/audio processing.

✅ **The actor has many concurrent consumers** — caches, request coordinators, database wrappers — anything multiple callers hit at once. Holding the actor on sync work serializes them all.

✅ **The method is essentially a pure function** — it only uses its parameters. The fact that it lives on the actor is organizational, not functional.

**When it's NOT meaningful**

❌ **Method reads/writes actor state a lot** — every access becomes `await MainActor.run { ... }`-equivalent (hop back to the actor). Net result: slower than just running on the actor.

```swift
actor Bank {
    var balances: [String: Int] = [:]

    @concurrent
    func transfer(from: String, to: String, amount: Int) async {
        // ❌ Bad fit: every line needs a hop back to read/write balances.
    }
}
```

❌ **Method is small / fast** — the hop-off-then-hop-back overhead exceeds the work. Just run it on the actor.

❌ **Method is mostly awaits already** — actors don't block during `await`, other methods get to run at suspension points. Offloading buys nothing.

**Mental model**

Think of `@concurrent` as "this method shouldn't hold the actor's serial executor while it runs." The question to ask:

> Does this method spend significant time doing synchronous work that doesn't touch my actor's state?

If yes → `@concurrent` is meaningful.
If no → leave it on the actor.

For MainActor the bar is lower (any blocking is bad because UI). For custom actors the bar is "would queueing behind this hurt my callers' latency?" — usually only true for caches, coordinators, and other multi-consumer actors.

---

## 5. `@concurrent` vs `nonisolated async`

### Q
When should I reach for `@concurrent` instead of `nonisolated async`? Is there a one-sentence rule?

### A
`@concurrent` is guaranteed to run on the shared thread pool in the background. For `nonisolated async`, it depends on your upcoming feature settings. As described earlier, the one-sentence explanation is that you should use `@concurrent` when you want your async function to be offloaded to the concurrent thread pool. It is an explicit spelling for behavior that used to be implicit for async functions, but that behavior is changing because it was not the best trade-off and not the best default for async functions. So you should use `@concurrent` now when you want that behavior. With `nonisolated async`, it depends on whether you have the approachable concurrency upcoming features enabled in your Xcode project, which is the default for new Xcode projects. If you do not have it enabled, I recommend enabling those settings because it gives you a set of defaults that are a bit easier to use with the data safety diagnostics.

### Explain
**The one-sentence rule**

- `@concurrent` = "always run me off-actor."
- `nonisolated async` (with Approachable Concurrency) = "run me wherever the caller is."

That's the whole difference. `@concurrent` is a guarantee; `nonisolated async` is adaptive.

**Side-by-side**

```swift
struct Helper {
    @concurrent
    func parseHeavy(_ data: Data) async -> Model { ... }

    nonisolated
    func parseLight(_ data: Data) async -> Model { ... }
}
```

Calling them from different places:

```swift
@MainActor func fromMain() async {
    await parseHeavy(data)   // ✅ runs on global pool (off main)
    await parseLight(data)   // ✅ runs on MainActor (stays on main)
}

actor Cache {
    func fromActor() async {
        await parseHeavy(data)   // ✅ runs on global pool (off Cache)
        await parseLight(data)   // ✅ runs on Cache's executor (stays on Cache)
    }
}
```

Same two functions, different behaviors based on caller — except `@concurrent` is caller-independent.

**How to choose**

Ask yourself: *"If this function is called from MainActor, do I want it to leave MainActor?"*

| Answer | Use |
|---|---|
| Yes, always — heavy CPU/IO that must not block the caller | `@concurrent` |
| No, stay where you are — light glue code, mostly awaits | `nonisolated async` |
| Depends on the caller — let them decide | `nonisolated async` |

**Why this matters**

Before SE-0461, `nonisolated async` had the `@concurrent` behavior baked in (always hopped to global pool). That was the wrong default — most async helpers just await other things and have no reason to leave the caller. The new design splits the two:

- **Default-safe** (`nonisolated async`): inherit caller, no surprises.
- **Explicit-offload** (`@concurrent`): you meant to leave.

**Practical rule of thumb**

> Reach for `@concurrent` when the function name has a verb like *parse*, *decode*, *encode*, *compress*, *hash*, *resize*, *render*, *compute*. Reach for `nonisolated async` when the function is mostly `await someOtherThing()` plumbing.

And — turn on Approachable Concurrency in your Xcode project if it isn't already. Without it, plain `nonisolated async` still has the old hop-to-pool behavior, and the rule above breaks.

---

## 6. Too Many Actors?

### Q
Is it possible to have too many actors? Is there an intrinsic limit?

### A
There is probably an intrinsic limit depending on how much memory you have, but I would definitely think about limiting how many different actors you have in your program. Generally speaking, you want to have just a few isolation domains in a program to reason about. Concurrent programming is hard, as we said earlier, and most of your code should be synchronous unless you have a reason to introduce concurrency and a reason to introduce different isolation domains. Every actor is a different isolation domain, and if actors need to work together, that creates complexity in your program. So you really want to think carefully when you factor something out into an actor, and decide whether it is really appropriate to add the complexity of needing to access that code asynchronously from other parts of your program. I think a very common question is: when do you actually use an actor? The answer is specifically when you can define that something will always need its own isolated domain, and the entirety of its functionality requires you to maintain that isolation. On the slide, I had a couple of examples like network caching or accessing a database. Those are common things where you often already know the work is going to be in an isolated domain anyway.

### Explain
**Short answer**

Yes — too many actors is a real problem, but the limit is **cognitive**, not technical. Each actor is a separate isolation domain, and every boundary between domains adds an `await`, a Sendable check, and a chunk of complexity.

**The real rule**

> Most of your code should be plain synchronous code. Only reach for an actor when something genuinely needs its own isolation domain for all of its operations.

**Good actor candidates**

These have shared mutable state with multiple concurrent consumers:

```swift
actor ImageCache { ... }         // many screens read/write
actor TokenStore { ... }         // refresh + read race each other
actor Database { ... }           // every query touches shared connection
actor RequestCoordinator { ... } // dedupes in-flight network calls
```

The pattern: **shared + mutable + accessed from multiple places + must stay consistent**.

**Bad actor candidates**

People often introduce actors when they don't need to:

❌ A ViewModel → use `@MainActor class`. It's already isolated, and it usually drives UI which lives on main anyway.

❌ A pure helper / utility → just a struct with `static func` or free functions. No state, no race, no actor needed.

❌ A stateless service (e.g. a thin wrapper around URLSession) → struct or class with `nonisolated` methods. There's nothing to protect.

❌ Something only one place uses → if there's a single owner, you don't have a concurrency problem yet. Wait until you do.

❌ A "manager" that holds config → struct of `let` properties. Immutable = trivially safe, no actor.

**The cost you pay per actor**

Every time non-actor code talks to an actor:

```swift
let value = await cache.image(for: url)  // ← await, suspension, scheduling
```

- Caller must be `async` (or in a Task)
- Inputs/outputs must be Sendable
- A suspension point appears — state can change while you're waiting
- Logic that crosses two actors needs careful ordering ("did A change while I awaited B?")

Multiply that by every actor → boundaries → `await` → reasoning load. Five well-chosen actors are easier to maintain than fifteen accidental ones.

**A simple decision test**

Before making something an actor, ask:

1. Does it have **mutable state**? No → struct or sync class.
2. Is that state accessed from **multiple concurrent contexts**? No → `@MainActor` or single owner.
3. Does **every method** need that protection? No → maybe just one method needs locking; reconsider the design.
4. All three yes? → `actor` is the right tool.

**Mental model**

Think of actors like locks. You wouldn't sprinkle locks across every class "just in case." You add one when there's a real shared resource that needs guarding. Actors are the same — a deliberate concurrency boundary, not a default container.

So: a handful of actors, chosen on purpose. The rest of your app: synchronous code, `@MainActor` for UI, and async functions only where you actually do I/O or heavy work.

---

## 7. `.task` Modifier Cancellation on State Change

### Q
Could a Task started by `.task` modifier in a SwiftUI view be cancelled if the view is refreshed by state change?

### A
The task modifier in SwiftUI cancels your task when the view disappears, and state changes usually do not cause your view to be fully destroyed by SwiftUI. State works in a way where, when the state changes, SwiftUI reevaluates the view's body, but it does not actually destroy the view. One case where you may see this behavior is if you have an `if` statement in your view's body, and one branch becomes true while the other branch previously contained a view with a task. Once the condition flips, SwiftUI would cancel the task because the view in the other branch of the if statement is destroyed. You can also think about it like `onDisappear`: task is a modifier, and the task modifier manages cancellation with the same timeline as the `onDisappear` modifier, if you have used that before.

### Explain
**Short answer**

No — a state change that just re-renders the view does **NOT** cancel `.task`. The task is only cancelled when SwiftUI actually **removes the view from the hierarchy**.

**The mental model**

`.task` ↔ `onAppear` / `onDisappear` lifetime, **not** body re-evaluation lifetime.

- State changes → body re-runs → `.task` keeps running.
- View removed from hierarchy → `.task` is cancelled.

**Examples**

✅ **State change — task is NOT cancelled**

```swift
struct CounterView: View {
    @State var count = 0

    var body: some View {
        VStack {
            Text("\(count)")
            Button("Tap") { count += 1 }   // re-renders body
        }
        .task {
            await loadStuff()              // keeps running across taps
        }
    }
}
```

Tapping the button re-evaluates body 100 times — `loadStuff()` runs once and is never interrupted.

❌ **View removed via `if` branch — task IS cancelled**

```swift
struct Parent: View {
    @State var showChild = true

    var body: some View {
        if showChild {
            Child()           // has .task inside
        } else {
            Text("Gone")
        }
        Toggle("Show", isOn: $showChild)
    }
}

struct Child: View {
    var body: some View {
        Text("loading…")
            .task {
                await longWork()   // cancelled when showChild flips to false
            }
    }
}
```

When `showChild` becomes false, SwiftUI destroys `Child` → its `.task` is cancelled.

❌ **Other lifecycle endings that cancel `.task`**

- View popped from `NavigationStack`
- Sheet / fullScreenCover dismissed
- View removed from a `ForEach` (item deleted from data)
- Tab switched away (depending on tab style — sometimes the view is destroyed)

**Bonus: `.task(id:)` — restarts on id change**

```swift
.task(id: userId) {
    await loadProfile(userId)   // cancelled & restarted whenever userId changes
}
```

Use this when you want the task to react to a value change — e.g. reload when the user navigates to a different profile.

**The rule**

| Event | `.task` cancelled? |
|---|---|
| `@State` changes, body re-evaluates | ❌ No |
| Parent passes new props, body re-evaluates | ❌ No |
| View removed (if flip, pop, dismiss, ForEach delete) | ✅ Yes |
| `.task(id:)` and the id value changes | ✅ Yes (and restarts) |

So you can lean on `.task` for "run this for as long as the view is on screen" — it's tied to the view's lifetime, not its renders.

---

## 8. The `nonisolated` Keyword After Recent Changes

### Q
Intuitively, what does the `nonisolated` keyword mean — especially after the recent changes?

### A
What has not really changed necessarily, but is now defaulted in some cases, is that `nonisolated` is a statement of your intent to not require any particular isolation for a type or function. When you annotate a type as `@MainActor`, you are stating that you only want it to be used from MainActor. `nonisolated` is removing that restriction. You can use it from any isolation domain, and that is really flexible and powerful. It is often the right choice for code in libraries, where you do not know where the app will want to use your library types. It can also be appropriate in your app for low-level models that do not have any inherent need to be isolated to MainActor or some other isolation domain. That is part of what motivated the behavior change for `nonisolated` with async functions. When something is marked as `nonisolated`, you can use it from anywhere. If you have a method that is not isolated and you call it from somewhere, it is going to run in the isolation domain it was called from. That used to not be the behavior for async functions, but now it is. So now there is a consistent meaning: when you see `nonisolated`, it means you can use it from anywhere, and when you call those methods, they are going to run wherever you call them from.

### Explain
**The intuition**

`nonisolated` = "I have no opinion about where I run. Call me from wherever."

It's the opposite of `@MainActor` (which says "you can ONLY call me from MainActor").

| Annotation | Meaning |
|---|---|
| `@MainActor` | "You MUST be on MainActor to call me" |
| `actor` | "You MUST hop into me to call me" |
| `nonisolated` | "Call me from anywhere — I run wherever you are" |

**Why "after the recent changes" matters**

Before SE-0461, `nonisolated` was inconsistent between sync and async:

```swift
// OLD behavior (Swift 6.0)
nonisolated func a() { }              // runs on caller's context ✅ consistent
nonisolated func b() async { }        // hops to global pool ❌ surprising
```

That asymmetry was confusing. Now it's clean:

```swift
// NEW behavior (Swift 6.2 + Approachable Concurrency)
nonisolated func a() { }              // runs on caller's context ✅
nonisolated func b() async { }        // runs on caller's context ✅
```

So now you can read `nonisolated` and have one mental model: *"runs wherever you call it from."*

**When to reach for `nonisolated`**

✅ **Library/SDK code** — you don't know where the app will use it, so don't impose `@MainActor` on consumers.

```swift
public struct ImageDecoder {
    public nonisolated func decode(_ data: Data) -> UIImage? { ... }
}
```

✅ **Low-level model types** — pure data, Codable, value types. They have no business being pinned to an actor.

```swift
struct ParkingSpace: Codable {
    nonisolated func formattedID() -> String { ... }
}
```

✅ **Helper methods on a `@MainActor` class that don't touch state**

```swift
@MainActor
final class ViewModel {
    var items: [Item] = []

    nonisolated func validate(_ id: String) -> Bool {
        // doesn't read self.items, so no need to be on MainActor
        return id.count == 8 && id.allSatisfy(\.isLetter)
    }
}
```

Now `validate` is callable from a background thread, an actor, anywhere — no `await` needed.

**When NOT to use it**

❌ The method touches self's isolated state — compiler will block you.

```swift
@MainActor
final class VM {
    var items: [Item] = []

    nonisolated func clear() {
        items = []   // ❌ can't access MainActor state from nonisolated
    }
}
```

❌ You actually want **guaranteed off-actor execution** — that's `@concurrent`, not `nonisolated`.

**The one-line summary**

`nonisolated` = "no isolation requirement, runs wherever the caller is."

After SE-0461, that meaning is finally consistent for both sync and async — which is the whole point of the change.

---

## 9. MainActor Default Isolation: Apps vs Libraries

### Q
For a new app target, should I adopt MainActor default isolation? Does your answer change for libraries?

### A
In Xcode 26, all new app targets use MainActor isolation by default. As someone from the SwiftUI team, I think that is an absolutely amazing choice for app targets because, in our UI framework, views are already on MainActor. If you have written a SwiftUI view and also have a model type, you may have noticed that it is often convenient to put that model on MainActor as well. With MainActor by default turned on, you do not have to do that manually. It makes UI code much easier to write and reason about, and you do not have to think as much about synchronizing your state because everything is on MainActor. But when it comes to libraries, my answer would change, depending on what the library does. If you are trying to be performant, concurrency can help you optimize behavior in your code by parallelizing operations, and libraries can make use of that in some cases. If you find that an operation is taking too long and you want to split it up, then it can make sense to introduce more concurrency, especially in library code. The same applies to app targets as well: you start on MainActor, and once you see potential for a performance optimization, or you profile the app and see the main thread being blocked by an expensive operation in your model, then you can consider adding concurrency annotations and moving things off MainActor. I also think many general-purpose library APIs are not specific to any particular isolation domain. Most of Foundation is not isolated, and clients of Foundation can choose where they want to use those APIs. For example, with Foundation, most APIs are not specific to any isolation, so most of them are nonisolated. There are a few APIs that are specific, such as `UndoManager`, which is heavily UI-focused and annotated with MainActor. For Foundation, it makes sense to keep most things nonisolated by default and add MainActor isolation for UI-related APIs. But when you are working on an app, it is almost the inverse: most of your things are on MainActor, and you may want to selectively offload work from there.

### Explain
**Short answer**

- **App target** → YES, use MainActor default.
- **Library** → NO, default to `nonisolated`.

The two have opposite needs.

**Why YES for apps**

In Xcode 26, new app targets ship with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` — exactly what your project already has set in CLAUDE.md. That means:

```swift
// No annotation needed — implicitly @MainActor
final class ParkingViewModel {
    var spaces: [Space] = []
    func refresh() { ... }
}
```

Wins:
- Views are already on MainActor, so models flow naturally with them — no `await` to bridge.
- No Sendable boilerplate for things that only ever touch UI.
- Less to think about — one isolation domain by default; introduce more only when you have a reason.
- Performance is rarely the bottleneck on day one — most app code is glue and small operations that are fine on main.

The intended workflow: start on MainActor → profile → selectively offload the hot spots with `@concurrent` or a real actor.

**Why NO for libraries**

A library doesn't know where it'll be used. If you stamp `@MainActor` on everything:

```swift
@MainActor
public struct ImageDecoder {
    public func decode(_ data: Data) -> UIImage? { ... }
}
```

…then every server-side, command-line, or background-task consumer is forced onto MainActor for no reason. That's a leaky concurrency design.

Default to `nonisolated`, let the caller pick:

```swift
public struct ImageDecoder {
    public nonisolated func decode(_ data: Data) -> UIImage? { ... }
}
```

Now an iOS app can call it from MainActor, a server can call it from a request handler, a CLI can call it from a `@main` function — none of them get isolated for free.

**The Foundation pattern**

Apple's own libraries follow this:

| API | Isolation | Why |
|---|---|---|
| `URLSession`, `JSONDecoder`, `Date`, `String` | nonisolated | usable from anywhere |
| `UndoManager` | `@MainActor` | inherently tied to UI |
| `NotificationCenter.publisher` | nonisolated, but main-thread variants exist | flexible |

The rule they applied: **default nonisolated, opt into MainActor only for genuinely UI-bound APIs**. Mirror this in your own libraries.

**Mental model**

- App target: "everything on MainActor; offload the hot bits"
- Library: "nothing isolated; let the caller decide"

For your ParkingLotNavigate app: stick with MainActor default ✅
If you ever extract `AprilTagDetectorImpl` or `PathfindingService` into a reusable SPM package: flip those to `nonisolated` APIs and let the host app decide where to call them from.

---

## 10. Cancelling Tasks in `deinit`, Memory, and Auto-Cancellation

### Q
If a class launches a long-running Task and calls `task.cancel()` in `deinit`, does the task actually stop executing? Is its memory released immediately after cancellation? Are tasks automatically cancelled when their owning scope is deallocated?

### A
Cancellation in Swift concurrency is cooperative. Depending on what the task is doing, that operation needs to check for cancellation. Usually, for a library method or similar API to support cancellation, the course of action is to throw a cancellation error. If you are running long synchronous code and someone cancels the task handle, that synchronous code is not going to be interrupted at any moment. That is actually an important behavior, because the work might need to perform some cleanup operation that must happen atomically without being interrupted by something like task cancellation. So no, the task does not necessarily stop running immediately because of cancellation. But as soon as you hit a piece of code that handles cancellation and maybe throws a cancellation error, that is when it will usually return by throwing such an error.

There was another question about whether tasks get cancelled when their owning scope is deallocated. If a class has a task handle, like we discussed earlier, and you want to track a task to cancel it, you need to store it as a property. But no, it is not automatically cancelled just because the owning scope is deallocated. That is one of the gotchas when you opt into managing your task's lifetime: you always have to remember to cancel it. That is also good in the sense that you are being explicit about the task's lifetime.

There was also a question about tasks swallowing errors. A Task does not require handling of thrown errors, which can lead to silently ignored failures. There is now an accepted proposal that addresses that exact pain point. In Swift 6.4, there will be a new diagnostic for exactly that case, where you have a task and have not handled errors within it. There are different ways to handle errors within a task. You can handle any thrown errors inside the task body, or if the task itself can throw and you want to keep a handle to the task, you can wait until you get the task's value and handle any potential errors that were thrown. Those are two different ways to handle task errors. So yes, with the acceptance of that proposal, you will now get diagnostics for this. If you are interested, you can check the Swift Forums or Swift.org Swift Evolution for the accepted proposed behavior.

### Explain
**Three questions, three answers**

**1. Does `task.cancel()` stop the task immediately?**

No. Cancellation in Swift is **cooperative** — `cancel()` just sets a flag. The task only actually stops if its body checks that flag.

```swift
let task = Task {
    for i in 0..<1_000_000_000 {
        heavyMath(i)               // ⚠️ pure sync — never checks cancellation
    }
}
task.cancel()                       // sets flag; the loop above keeps grinding
```

vs. cooperatively-cancellable code:

```swift
let task = Task {
    for i in 0..<1_000_000_000 {
        try Task.checkCancellation() // ✅ throws if cancelled
        heavyMath(i)
    }
}
```

What automatically responds to cancellation:
- `try await` on most stdlib/Foundation async APIs (URLSession, `Task.sleep`, etc.) — they throw `CancellationError`.
- Explicit `try Task.checkCancellation()` calls.
- `Task.isCancelled` checks you write yourself.

What does **not**:
- Plain synchronous loops, math, file reads, image decoding.

This is intentional — sometimes the work needs to finish a cleanup atomically and shouldn't be yanked.

**2. Is memory released immediately after `cancel()`?**

No. The Task object lives until its body actually returns. Cancellation just *requests* an early return; the body has to honor it. Until then, the task (and everything it captures) stays alive.

```swift
let task = Task { [weak self] in
    while !Task.isCancelled { ... }   // returns soon after cancel() → freed
}
```

If you capture `self` strongly inside the task body, `self` is retained until the task ends — even after `cancel()`.

**3. Are tasks auto-cancelled when their owner deallocates?**

No — and this is one of the biggest Swift concurrency gotchas.

```swift
final class VM {
    var task: Task<Void, Never>?

    func start() {
        task = Task { while !Task.isCancelled { ... } }
    }

    deinit {
        task?.cancel()   // ✅ MUST do this manually
    }
}
```

If you forget the `cancel()` in `deinit`, the task keeps running (and keeps `self` alive if it captured strongly → `deinit` never runs in the first place 😱).

Your `ARNavigationViewModel` already does this correctly — `taskStorage.dispose()` in `onDisappear` is the same pattern.

**The exception: structured tasks**

`async let`, `withTaskGroup`, and SwiftUI's `.task` modifier ARE auto-cancelled when their scope ends. Those are **structured concurrency**. Plain `Task { }` is **unstructured** — you own its lifetime.

| Form | Lifetime tied to | Manual cancel? |
|---|---|---|
| `async let x = ...` | enclosing function body | ❌ no |
| `withTaskGroup { ... }` | the group's closure | ❌ no |
| `.task { }` SwiftUI modifier | view in hierarchy | ❌ no |
| `Task { ... }` (unstructured) | nothing — you own it | ✅ yes |

**Bonus: silently-swallowed errors**

```swift
Task {
    try await riskyThing()   // ⚠️ if this throws, error is silently dropped
}
```

The compiler doesn't warn today. SE-0485 (accepted, landing in Swift 6.4) will add a diagnostic for this. Two valid fixes:

```swift
// Handle inside
Task {
    do { try await riskyThing() }
    catch { logger.error("\(error)") }
}

// Or keep the handle and await the value elsewhere
let task = Task { try await riskyThing() }
do { _ = try await task.value } catch { ... }
```

**Mental model**

- `cancel()` = "please stop when convenient"
- NOT = "kill -9"

Three things to remember:
1. Cancellation is cooperative → write `try await` / `Task.checkCancellation()` at suspension points.
2. Unstructured tasks need manual lifetime management → store, `cancel()` in `deinit`/`onDisappear`.
3. `Task { try await ... }` swallows errors today → handle them, or wait for Swift 6.4's warning.

---

## 11. Best Practice for Async Requests on Button Tap

### Q
What's the best practice for asynchronous requests in response to a button tap? Kicking off a one-off task, or exposing a synchronous method from your model that kicks off a task internally?

### A
The general recommendation, although you can structure your code in both ways, is that I would prefer moving that asynchronous code to your model. Then the model can respond back with synchronous output to your view, so the view stays responsive and the model handles all of that asynchronous work. The benefit of this approach is that you can also test the asynchronous work outside of your view, which brings greater benefits to your entire codebase.

### Explain
**Recommendation: put the async work in the model**

The view should call a **synchronous** method on the model. The model owns the Task internally.

❌ **Inline approach (don't do this)**

```swift
struct ParkingView: View {
    @State var vm = ParkingViewModel()

    var body: some View {
        Button("Fetch spaces") {
            Task {
                await vm.loadSpaces()    // view kicks off the task
            }
        }
    }
}
```

Why it's worse:
- **Untestable** — concurrency lives in the view; you can't unit-test the flow without driving the SwiftUI lifecycle.
- **No lifetime control** — view has no handle to cancel, dedupe, or track the task.
- **Concurrency leaks into UI code** — every button does its own `Task { }` boilerplate.
- **Race-prone** — tap twice, get two parallel requests.

✅ **Model owns the task**

```swift
@Observable @MainActor
final class ParkingViewModel {
    private(set) var spaces: [Space] = []
    private(set) var isLoading = false
    private var loadTask: Task<Void, Never>?

    // Synchronous from the view's perspective
    func loadSpaces() {
        loadTask?.cancel()                    // dedupe / cancel previous
        loadTask = Task {
            isLoading = true
            defer { isLoading = false }
            do {
                spaces = try await api.fetchSpaces()
            } catch {
                logger.error("\(error)")
            }
        }
    }
}

struct ParkingView: View {
    @State var vm = ParkingViewModel()

    var body: some View {
        VStack {
            if vm.isLoading { ProgressView() }
            List(vm.spaces) { Text($0.id) }
            Button("Fetch spaces") {
                vm.loadSpaces()              // ← clean, synchronous call
            }
        }
    }
}
```

What you get:

| Win | How |
|---|---|
| Testable | `await vm.loadTask?.value` in tests, then assert `vm.spaces` |
| Cancelable | `loadTask?.cancel()` on rapid re-taps or `onDisappear` |
| Loading state | `isLoading` is owned by the model, view just reads it |
| Errors handled | Centralized in one place, can surface via `vm.errorMessage` |
| View stays declarative | No `Task { }` clutter in the body |

**When the inline form is OK**

For genuinely fire-and-forget UI-local actions where the model doesn't care:

```swift
Button("Share") {
    Task { await UIActivityFeedback.success() }
}
```

But once there's any state (`isLoading`, error, results to display), move it to the model.

**Mental model**

> The view declares **what** should happen on tap.
> The model decides **how** (and when, and whether to cancel a previous one).

This is the same separation you already use elsewhere — putting the Task in the model is just extending it to async behavior.

---

## 12. Unit Testing Async/Await Code

### Q
What's the best practice to write a unit test for async/await code?

### A
You can use Swift Testing and make your test method async, then test it there. If you need that method to run with a particular isolation, you can mark the test itself or the entire test case type with MainActor, or with whatever isolation you need that test to run on. Swift Testing is definitely worth checking out if you have not already.

### Explain
**Use Swift Testing — async tests are first-class**

In Swift Testing, just mark the test `async` and `await` directly. No `XCTestExpectation`, no `wait(for:)`, no expectation dance.

**Basic async test**

```swift
import Testing

@Test
func fetchesSpaces() async throws {
    let api = MockAPI(spaces: [.fixture(id: "A1"), .fixture(id: "A2")])
    let result = try await api.fetchSpaces()
    #expect(result.count == 2)
    #expect(result.first?.id == "A1")
}
```

**Testing a `@MainActor` model**

When the code under test is MainActor-isolated, mark the test (or whole suite) `@MainActor`:

```swift
@MainActor
@Test
func loadingFlipsCorrectly() async throws {
    let vm = ParkingViewModel(api: MockAPI(...))

    vm.loadSpaces()                       // sync call, kicks off internal Task
    #expect(vm.isLoading == true)

    try await vm.loadTask?.value          // wait for the task to finish
    #expect(vm.isLoading == false)
    #expect(vm.spaces.count == 2)
}
```

Or pin the whole suite once:

```swift
@MainActor
@Suite
struct ParkingViewModelTests {
    @Test func loadsSpaces() async throws { ... }
    @Test func handlesError() async throws { ... }
    @Test func cancelsPreviousLoad() async throws { ... }
}
```

**Testing the "model owns the Task" pattern**

The trick: expose the Task (or a `done()` helper) so tests can `await` it. Otherwise tests become flaky polling loops.

```swift
@Observable @MainActor
final class ParkingViewModel {
    private(set) var spaces: [Space] = []
    var loadTask: Task<Void, Never>?      // accessible to tests

    func loadSpaces() {
        loadTask = Task { spaces = (try? await api.fetchSpaces()) ?? [] }
    }
}

@MainActor @Test
func loadsSpaces() async {
    let vm = ParkingViewModel(api: MockAPI(...))
    vm.loadSpaces()
    await vm.loadTask?.value              // ← deterministic, no sleeps
    #expect(vm.spaces.count == 2)
}
```

**Testing errors**

```swift
@Test
func throwsOnNetworkFailure() async {
    let api = MockAPI(error: URLError(.notConnectedToInternet))
    await #expect(throws: URLError.self) {
        try await api.fetchSpaces()
    }
}
```

**Testing actor methods**

Just `await` them — no special setup:

```swift
@Test
func cacheStoresAndReturns() async {
    let cache = ImageCache()
    await cache.store(image: .test, for: URL(string: "x")!)
    let got = await cache.image(for: URL(string: "x")!)
    #expect(got != nil)
}
```

**Anti-patterns to avoid**

❌ `Task.sleep` to "wait for it" — flaky, slow, hides race conditions.

```swift
vm.loadSpaces()
try await Task.sleep(for: .seconds(1))   // ❌ guess and pray
#expect(vm.spaces.count == 2)
```

✅ Expose the task handle and `await` its `.value`.

❌ `XCTestExpectation` in async tests — you don't need it. Just `await`.

❌ Using `DispatchQueue.main.async` in tests — bridge old code to async with `await MainActor.run { ... }` or restructure.

**Mental model**

> If your production code is async, your test should be async too — let them meet in the same world.

Three rules:
1. `@Test + async throws` for the test signature.
2. Match isolation — `@MainActor` on the test/suite when the code is MainActor.
3. Make tasks awaitable — expose handles or completion signals so tests are deterministic, not timed.

---

## 13. Best Use Case for `Task.detached`

### Q
What is the best use case for `Task.detached`?

### A
I have seen a lot of questions recently about when to use `Task.detached` versus when to use `Task { @concurrent in }`, because now those two things have very similar behaviors. The main difference is that the plain Task initializer, even when you specify some isolation, still inherits certain things from the surrounding context. Obviously, if you have explicitly specified an isolation, it does not inherit isolation, but it does inherit priority from the surrounding context. `Task.detached`, on the other hand, is completely detached. Nothing from the surrounding context is inherited by that task. So if you want something that is completely detached from the surrounding context, use `Task.detached`. If you just want to control the isolation, but you still want those other things to be inherited, use `Task { @concurrent in }`.

### Explain
**Short answer**

Use `Task.detached` when you specifically want to **cut off all inheritance** from the surrounding context — not just isolation, but also priority and task-local values.

For "I just want to run off the actor" → use `Task { @concurrent in ... }` instead. It's the new, more focused tool.

**What gets inherited**

```swift
@MainActor
func tapped() {
    Task { ... }                 // inherits: isolation (Main), priority, task-locals
    Task { @concurrent in ... }  // inherits: priority, task-locals  (NOT isolation)
    Task.detached { ... }        // inherits: NOTHING
}
```

| | Isolation | Priority | Task-locals |
|---|---|---|---|
| `Task { }` | ✅ | ✅ | ✅ |
| `Task { @concurrent in }` | ❌ (global pool) | ✅ | ✅ |
| `Task.detached { }` | ❌ | ❌ | ❌ |

**When `Task.detached` is the right call**

✅ **Background work that shouldn't ride a high-priority context**

```swift
@MainActor
func userTapped() {
    // Tap handler is .userInitiated. We don't want telemetry to share that.
    Task.detached(priority: .background) {
        await analytics.flush()
    }
}
```

If you used plain `Task { @concurrent in }`, the flush would inherit `.userInitiated` and compete with real user work. Detaching breaks the chain.

✅ **Starting a fresh task-local scope**

If you use `@TaskLocal` (request IDs, trace IDs, logging contexts), `Task.detached` gives you a clean slate:

```swift
@TaskLocal static var requestID: UUID?

func handleIncoming() async {
    await Self.$requestID.withValue(UUID()) {
        Task.detached {
            // requestID is nil here — fresh context
            await processInBackground()
        }
    }
}
```

✅ **Long-lived background loops decoupled from any caller**

A monitoring loop, a periodic sync — work whose lifetime shouldn't track whoever happened to start it.

**When NOT to use it**

❌ "I just want off MainActor" → use `@concurrent`:

```swift
// Old habit
Task.detached { let r = decode(data); await MainActor.run { vm.image = r } }

// Better
Task { @concurrent in
    let r = decode(data)
    await MainActor.run { vm.image = r }
}
```

The `@concurrent` version still inherits priority — usually what you want, so a UI tap's decode runs at user-initiated priority instead of dropping to default.

❌ Default choice → don't reach for detached "to be safe." Cutting off priority and task-locals can starve the work or hide bugs in your tracing/logging.

**Mental model**

- `Task.detached` = "this work is its own creature — don't tie it to me"
- `Task { @concurrent in }` = "run this off-actor, but it's still my work"

Reach for `Task.detached` when you can name the inheritance you want to break — priority chain, or task-locals. If you can't name one, you probably want `@concurrent` instead.

---

## 14. Cancelling a URLSession Request When the View Dismisses

### Q
Async Task from a SwiftUI view that calls a shared `@Observable` service. If the user dismisses the view before the task finishes, what's the way to cancel it so the URLSession request is actually dropped (not just abandoned) and no UI state mutation happens post-dismiss?

### A
In SwiftUI's task modifier, SwiftUI does not currently expose a way to give you the task handle so you can cancel the task owned by SwiftUI. But since task is effectively similar to `onAppear` starting a task, you could structure your code so that you own that task yourself. Then, within `onAppear` of your SwiftUI view, you can start it, keep the task handle, and cancel it yourself when needed. If the condition is that the user dismisses the view before the task is finished, then you may also be able to use the task modifier directly, because SwiftUI cancels the task spawned there when the view is dismissed. So it depends on what your expectation is. But there is always a way to use your own Task instance that is not owned by SwiftUI if you need more granular control.

### Explain
**Short answer**

Use the `.task` modifier and call your service through `try await`. SwiftUI cancels the task on dismiss, the cancellation propagates into URLSession, the request is **actually dropped**, and code after the `await` never runs — so no post-dismiss state mutation.

The trick most people miss: **cancellation only works if every `await` in the chain is cancellable** (i.e. uses `try await` and throws `CancellationError`).

**The right pattern**

✅ **Recommended: `.task` modifier + service that propagates cancellation**

```swift
@Observable @MainActor
final class ParkingService {
    var spaces: [Space] = []

    func loadSpaces() async throws {
        // URLSession.data IS cancellable — throws CancellationError
        // when the surrounding task is cancelled.
        let (data, _) = try await URLSession.shared.data(from: endpoint)
        let result = try JSONDecoder().decode([Space].self, from: data)
        spaces = result   // never executes if cancelled above
    }
}

struct ParkingView: View {
    let service: ParkingService

    var body: some View {
        List(service.spaces) { Text($0.id) }
            .task {
                do {
                    try await service.loadSpaces()
                } catch is CancellationError {
                    // expected on dismiss — silently stop
                } catch {
                    logger.error("\(error)")
                }
            }
    }
}
```

What happens on dismiss:
1. SwiftUI removes the view → cancels the `.task`'s task.
2. The `try await URLSession.shared.data(...)` throws `CancellationError`.
3. URLSession actually cancels the network request (sends RST/closes the socket).
4. `spaces = result` is never reached → no UI mutation.

That's exactly the behavior you want, and you didn't write any cancellation code yourself.

**When you need your own handle**

If your work outlives the view (e.g. background sync that should keep going after dismiss), don't use `.task`. Own the task in the model:

```swift
@Observable @MainActor
final class ParkingViewModel {
    private(set) var spaces: [Space] = []
    private var loadTask: Task<Void, Never>?

    func start() {
        loadTask?.cancel()
        loadTask = Task {
            do {
                let (data, _) = try await URLSession.shared.data(from: endpoint)
                spaces = try JSONDecoder().decode([Space].self, from: data)
            } catch is CancellationError {
                return                              // dropped intentionally
            } catch {
                logger.error("\(error)")
            }
        }
    }

    func stop() { loadTask?.cancel() }
}

struct ParkingView: View {
    @State var vm = ParkingViewModel()

    var body: some View {
        List(vm.spaces) { Text($0.id) }
            .onAppear  { vm.start() }
            .onDisappear { vm.stop() }
    }
}
```

This gives you full control: cancel on dismiss, on a button tap, on logout, whenever.

❌ **The trap people fall into**

```swift
.task {
    let result = await heavyThing()    // not `try await`!
    service.spaces = result            // ⚠️ runs even after dismiss
}
```

If `heavyThing()` is `async` (not `async throws`) and doesn't internally check cancellation, the task runs to completion and does mutate state after dismiss. The compiler won't warn.

Fixes:
- Use `try Task.checkCancellation()` inside long sync sections.
- Prefer `try await` chains end-to-end (URLSession, JSONDecoder, etc. cooperate).
- Or guard the write: `guard !Task.isCancelled else { return }` before mutating.

**Why URLSession "actually drops" the request**

`URLSession.shared.data(from:)` is built to listen for cancellation. When its surrounding Task is cancelled:

- It cancels the underlying `URLSessionDataTask`.
- The OS tears down the connection / sends socket close.
- The `await` throws `CancellationError`.

Other APIs (Foundation file I/O, `Task.sleep`, AsyncSequence iterations) follow the same contract. Custom code only inherits this if you write `try Task.checkCancellation()` at meaningful points.

**Decision guide for your scenario**

| Need | Use |
|---|---|
| Work tied to view's lifetime, cancel on dismiss | `.task { try await ... }` |
| Work outlives view, manual control | Task stored on model + `cancel()` in `onDisappear`/`deinit` |
| Restart on parameter change | `.task(id: someValue) { ... }` |
| Multiple parallel cancellable jobs | `withTaskGroup` or store `[Task]` on model |

For the exact case you described — "user dismisses view, drop the network call, don't mutate state" — the `.task` modifier is the right answer. It's not just a convenience; it's wired directly into the cancellation chain.

---

## 15. Does `async throws` Imply Cancellable?

### Q
Does an `async throws` function mean it's cancellable? If a function is only `async` (no throws), is it not cancellable?

### Explain
No — that's a common misconception.

- `async throws` ≠ cancellable.
- `async` ≠ not cancellable.

**Cancellation depends on what's inside the function, not its signature.**

The `throws` part is only about *how* cancellation is reported, not whether it happens.

**The two ways a function can react to cancellation**

**Way 1: Throw `CancellationError` (needs `throws`)**

```swift
func loadData() async throws -> Data {
    try Task.checkCancellation()         // throws if cancelled
    return try await URLSession.shared.data(from: url).0
}
```

**Way 2: Check `Task.isCancelled` and return early (no `throws` needed)**

```swift
func loadData() async -> Data? {
    if Task.isCancelled { return nil }    // returns early, doesn't throw
    return try? await URLSession.shared.data(from: url).0
}
```

Both are cancellable. Only the first one uses `throws`.

**Four combinations to make this concrete**

| Signature | Cancellable? | Why |
|---|---|---|
| `async throws` that calls `try await URLSession...` | ✅ Yes | The inner `try await` throws on cancel |
| `async throws` that runs a tight sync loop | ❌ No | Never checks; throws doesn't help by itself |
| `async` that checks `Task.isCancelled` | ✅ Yes | Returns early without throwing |
| `async` that just does sync work | ❌ No | Doesn't check anything |

**Examples**

`async throws` but NOT cancellable ❌

```swift
func compute() async throws -> Int {
    var sum = 0
    for i in 0..<1_000_000_000 {
        sum += i                          // no cancellation check
    }
    if sum < 0 { throw MyError.bad }      // throws for OTHER reasons, not cancellation
    return sum
}
```

The signature has `throws`, but the body never honors cancellation. `cancel()` does nothing here.

`async` (no `throws`) but IS cancellable ✅

```swift
func processBatch(_ items: [Item]) async -> [Result] {
    var results: [Result] = []
    for item in items {
        if Task.isCancelled { return results }   // partial result, no throw
        results.append(process(item))
    }
    return results
}
```

No `throws`, fully cancellable, returns whatever it had.

**The real rule**

A function is cancellable if its body does at least one of:

1. Calls `try Task.checkCancellation()` — throws on cancel.
2. Checks `Task.isCancelled` and acts on it.
3. `await`s another cancellable async API (URLSession, `Task.sleep`, AsyncSequence iteration, etc.) that does #1 or #2 internally.

`throws` is about the *mechanism* for reporting cancellation (as an error). It's not what makes a function cancellable.

**Why `try await URLSession` works**

`URLSession.shared.data(...)` is `async throws`, and crucially it **checks cancellation internally and throws `CancellationError`**. That's why the cancellation chain works in your earlier example — not because of the `throws` keyword, but because URLSession was implemented to honor it.

If you wrote your own:

```swift
func myDownload() async throws -> Data {
    var collected = Data()
    while !done {
        // No await, no check → not cancellable, despite `throws`
        collected.append(readChunkSync())
    }
    return collected
}
```

…the `throws` is there but cancellation doesn't work. You'd need to add `try Task.checkCancellation()` inside the loop.

**Mental model**

- `async` = "I might suspend"
- `throws` = "I might fail with an error"
- cancellable = "I check `Task.isCancelled` / `Task.checkCancellation` / await something that does"

Three independent properties. Don't conflate them.

So when reviewing code: don't look at the signature to judge cancellability — look at the body for cancellation checks or for awaits on APIs you trust to honor cancellation.

---

## 16. Exposing Actor Data to SwiftUI

### Q
What are Apple's recommended best practices for exposing data from actors in SwiftUI? Creating a "proxy" data broker between the actors and UI adds a lot of boilerplate, and requires my own notification mechanism when actor data changes.

### A
Exposing data from actors in SwiftUI depends on how that proxy object looks. But as we discussed earlier, introducing actors into your code adds extra synchronization points, where you have to await certain pieces. To access actor state from SwiftUI's MainActor isolation, you need to introduce a layer that handles that synchronization. So it is almost inevitable that some kind of layer has to be introduced to synchronize that state. It would be helpful to hear more about the specifics, especially what boilerplate is involved here, what the notification mechanism looks like, and how the architecture is set up. This is probably a good case to continue the conversation on the Swift Forums, or file feedback with a sample project that has a similar architecture. That would make it easier to look more closely at the setup and see whether there is room for improvement to make this easier.

### Explain
**The honest take**

The WWDC answer dodged the real complaint. You're right: bridging an actor to SwiftUI requires a proxy, and there's no built-in observable bridge. So the first question to ask is harder:

> Do you actually need an actor?

A lot of "actor → SwiftUI bridge" pain comes from picking an actor when `@MainActor` would have worked. Walk back through the decision before adding glue.

**Step 1: Question the actor**

| Symptom | Better fit |
|---|---|
| State is mostly read by views, written occasionally | `@MainActor @Observable class` — no bridge needed |
| State is mutated from many concurrent contexts (caches, request coordinators, DB) | Real actor — bridge is unavoidable |
| Heavy CPU work the actor does between mutations | `@MainActor @Observable` + `@concurrent` for the heavy methods |

If you fall in the first or third row, the bridge problem disappears. SwiftUI views can directly observe a `@MainActor @Observable` model. No proxy, no notification mechanism.

**Step 2: If you really need an actor — the patterns**

**Pattern A: MainActor `@Observable` proxy that mirrors actor state**

This is what you're already complaining about — it's also genuinely the most common pattern, because it's what `@Observable` requires (must be MainActor for SwiftUI to observe it).

```swift
actor SpaceCache {
    private var spaces: [Space] = []

    func all() -> [Space] { spaces }
    func add(_ s: Space) { spaces.append(s) }

    func updates() -> AsyncStream<[Space]> {
        AsyncStream { cont in
            self.continuations.append(cont)
            cont.yield(spaces)
        }
    }

    private var continuations: [AsyncStream<[Space]>.Continuation] = []
    private func broadcast() { continuations.forEach { $0.yield(spaces) } }
}

@Observable @MainActor
final class SpaceCacheProxy {
    private(set) var spaces: [Space] = []
    private let cache: SpaceCache

    init(_ cache: SpaceCache) { self.cache = cache }

    func observe() async {
        for await snapshot in await cache.updates() {
            spaces = snapshot          // back on MainActor
        }
    }
}

struct SpacesView: View {
    @State var proxy: SpaceCacheProxy
    var body: some View {
        List(proxy.spaces) { Text($0.id) }
            .task { await proxy.observe() }
    }
}
```

The `AsyncStream` IS the notification mechanism — you don't have to invent one. Cancellation also works for free: when the view leaves, `.task` cancels, the `for await` exits, the actor stops sending.

**Pattern B: Pull on demand, no proxy**

If updates are infrequent or user-triggered, skip the stream entirely:

```swift
@Observable @MainActor
final class SpacesViewModel {
    private(set) var spaces: [Space] = []
    private let cache: SpaceCache

    func refresh() async {
        spaces = await cache.all()    // single hop, simple
    }
}

.task { await vm.refresh() }
.refreshable { await vm.refresh() }
```

Boilerplate: a one-line method. Notification: not needed — you pull when you care.

**Pattern C: Skip the actor, use a serial executor**

If your actor was just "I want serialized access to a resource," modern Swift gives you `globalActor` or simply `@MainActor` with `@concurrent` methods on heavy bits — often cheaper than introducing a real actor + bridge:

```swift
@MainActor @Observable
final class SpaceCache {
    private(set) var spaces: [Space] = []

    @concurrent
    func reload() async {
        let fresh = await fetchFromDisk()      // off-main
        await MainActor.run { spaces = fresh } // back on main
    }
}
```

No actor, no bridge, view observes directly.

**Why there's no built-in `@Observable actor`**

`@Observable` works by tracking property reads inside a SwiftUI view's body, which runs synchronously on MainActor. An actor's properties are async-access — there's no synchronous read for SwiftUI to track. That's a language-level gap, not a library oversight, so a built-in fix would require new compiler/runtime support.

The community is actively discussing this on the Swift Forums (search "Observable actor"), and the pragmatic recommendation today is exactly what you've been doing — proxy + AsyncStream.

**Decision flow**

```
Does the state really have multiple concurrent writers?
├── No  → @MainActor @Observable class. Done.
└── Yes → Do views need live updates?
         ├── No  → @MainActor proxy with pull-on-demand `refresh()` method
         └── Yes → @MainActor proxy + AsyncStream from the actor
```

**Reducing the boilerplate**

If you keep landing in the proxy pattern, factor it once:

```swift
@MainActor
protocol ObservableProxy: Observable {
    associatedtype Snapshot
    func apply(_ snapshot: Snapshot)
    func updates() async -> AsyncStream<Snapshot>
}

extension ObservableProxy {
    func observe() async {
        for await s in await updates() { apply(s) }
    }
}
```

Concrete proxies become ~10 lines each. It's not nothing, but it's not the disaster the question implies — and you usually only need 2–3 proxies in a real app, not one per actor.

**Bottom line**

- Actor-to-SwiftUI always needs a MainActor bridge today; that's a language-level fact.
- Most apps don't need actors in the first place. Audit before you bridge.
- When you do need one, `AsyncStream` is the notification mechanism — you don't have to roll your own.
- Pull-on-demand is dramatically simpler than push and is enough for many cases. Don't reach for streams unless you need live updates.

---

## 17. Does `MainActor` Guarantee the Main Thread?

### Q
Does `MainActor` guarantee main thread? If not, when does it use another thread?

### A
On our systems, MainActor is the main thread for sure. You could imagine an environment that Swift compiles for where MainActor might technically be some other thread. There are programs that are not apps where that might make sense, but for most intents and purposes, MainActor means the main thread. There are also some cases where you can have something annotated as MainActor, but it is still possible to call it from off MainActor in a context where the call site does not have strict concurrency checking enabled. I think the case where people see this most frequently is when interoperating with C, Objective-C, or C++. Those languages do not have static data safety, so it is possible to get a data race if you call something that is MainActor-isolated from one of those contexts. Swift also has tools to insert dynamic checking if you want to catch cases like that at runtime.

### Explain
**Short answer**

Yes — for any iOS/macOS/visionOS/watchOS/tvOS app, **MainActor IS the main thread**. That's a real guarantee enforced by the Swift runtime on Apple platforms.

The "not necessarily" is a theoretical hedge for non-app Swift programs (CLI tools, server-side, embedded, etc.) where there might not even be a "main thread" in the UIKit/AppKit sense.

For your ParkingLotNavigate app: think of `@MainActor` and "main thread" as identical.

**When MainActor code can run off the main thread**

Three real escape hatches — all involve bypassing static checking:

**1. C / Objective-C / C++ interop**

Those languages don't carry isolation info. If a C callback fires on a background thread and you call into a `@MainActor` function from it, the compiler can't catch it.

```objc
// Some Objective-C SDK callback running on a background queue
dispatch_async(backgroundQueue, ^{
    [swiftObject doSomething];   // ← Swift method is @MainActor
                                 //   no compiler error, but data race at runtime
});
```

This is the case the WWDC speaker mentioned. Your project hits this surface area in places like `AprilTagDetectorObjC` and `QRCodeDetectorObjC` — anything that comes back from OpenCV / Vision callbacks and lands in Swift code.

**2. `@unchecked Sendable` / `@preconcurrency`**

You're explicitly telling the compiler "trust me." If you lie, MainActor code can run anywhere.

**3. Older code without strict concurrency**

Modules built without strict concurrency checking (or with `-Xfrontend -strict-concurrency=minimal`) can still cross the boundary.

**Catching violations at runtime**

Set the launch arg:

```
-com.apple.CoreFoundation.dyld_print_warnings 1
```

Or more usefully, enable thread-safety checks via the env var:

```
SWIFT_DEBUG_DEFAULT_ACTOR_VIOLATION_BEHAVIOR=fatal
```

This makes the runtime crash immediately when MainActor code runs off the main thread — far easier to debug than a flaky UI corruption later.

You can also assert manually:

```swift
@MainActor
func criticalUpdate() {
    dispatchPrecondition(condition: .onQueue(.main))   // crashes if not main
    // ... mutate UI state
}
```

Or in newer SDKs:

```swift
MainActor.assertIsolated()
```

**In your project specifically**

Two places worth checking with a runtime assertion:

1. `ARNavigationSceneDelegate.renderer(_:updateAtTime:)` — that's a nonisolated callback (SceneKit calls it on the render thread), and you correctly hop to MainActor with `Task { @MainActor in ... }`. Already safe by construction.
2. AprilTag / QRCode detector callbacks from OpenCV — these are the classic interop case. If any of them eventually mutate `@MainActor` state, ensure the hop is explicit (`await MainActor.run { ... }` or `Task { @MainActor in ... }`); don't rely on the C++ side to be on main.

**Mental model**

`@MainActor` on Apple platforms = main thread.
The exceptions are all places where you've stepped outside Swift's static checks.

If you stay in pure Swift with strict concurrency on, the equivalence holds. If you cross into C/Obj-C/C++ or use `@unchecked`, treat it as a manual responsibility — and consider a `dispatchPrecondition` or `MainActor.assertIsolated()` at the boundary to catch mistakes early.

---

## 18. What Is an Executor?

### Q
What is an executor? Is that exposed to us in Swift?

### A
You can make use of them, but you do not have to. Actors have default executors. The default executor manages scheduling for tasks, or more precisely, the synchronous pieces of a task, which are called jobs. Actors effectively have a priority queue of jobs that need to run on the actor from various tasks, and the executor handles scheduling and running those jobs. By default, that happens on the concurrent thread pool. You can also implement your own custom executors by conforming to a protocol and implementing whatever scheduling you need for your actor. Then you can provide a method on the actor that creates an instance of your custom executor type if you want to control that behavior. But the defaults are there so that you only need to drop down into that behavior if you need very fine-grained control over how those tasks are scheduled or run, or if you want to manage your own custom thread pool. That is something people sometimes do in services. So yes, those tools are available, but there are sensible defaults when you do not need that kind of control.

### Explain
**What an executor is**

An **executor** is the thing that decides *where* and *when* an actor's work runs. It's the scheduler under the hood.

Analogy:

- Actor = an inbox of pending work
- Executor = the worker who picks items from the inbox and runs them

Every actor has one. You usually don't see it.

**The defaults**

| Actor | Default executor | Effect |
|---|---|---|
| Custom actor `MyThing { ... }` | Global concurrent pool, serial within this actor | Runs on some background thread, one job at a time per actor |
| `@MainActor` | Main thread executor | Always runs on main thread |
| `nonisolated async` | Caller's executor (or global pool with `@concurrent`) | No actor → no actor executor |

So when you write:

```swift
actor Cache {
    func store(_ x: Int) { ... }
}
```

You get serial access to `Cache`'s state for free, scheduled by the default executor on the global pool. You never named the executor. That's the point.

**Yes, executors are exposed**

You can implement your own by conforming to `SerialExecutor`:

```swift
final class MyQueueExecutor: SerialExecutor {
    private let queue = DispatchQueue(label: "my.serial")

    func enqueue(_ job: consuming ExecutorJob) {
        let unowned = UnownedJob(job)
        queue.async { unowned.runSynchronously(on: self.asUnownedSerialExecutor()) }
    }

    func asUnownedSerialExecutor() -> UnownedSerialExecutor {
        UnownedSerialExecutor(ordinary: self)
    }
}

actor Database {
    private let _executor = MyQueueExecutor()
    nonisolated var unownedExecutor: UnownedSerialExecutor {
        _executor.asUnownedSerialExecutor()
    }
}
```

Now `Database`'s work runs on your dispatch queue instead of the global pool.

**When custom executors actually make sense**

✅ **Bridging to a legacy DispatchQueue** — your codebase has a critical queue (e.g. a database write queue), and you want a new actor to share the same serialization domain.

✅ **OS-level priority isolation** — you need a dedicated thread that won't be starved by the global pool (audio rendering, real-time work).

✅ **Server-side custom thread pools** — Vapor / SwiftNIO sometimes wire actors to their EventLoop executors so I/O and compute share a tuned pool.

✅ **Testing determinism** — a single-threaded test executor that runs jobs in submission order without scheduler nondeterminism.

**When you definitely shouldn't**

❌ "I want my actor to run on a specific thread for no particular reason." → default is fine.

❌ "I read about executors and want to try them." → resist. The defaults are tuned by the runtime; custom ones are easy to get wrong (priority inversion, deadlocks, broken cancellation).

❌ "I want my actor to run on MainActor." → use `@MainActor`. Don't reinvent it.

**The connection back to earlier topics**

- A **Task** is a unit of asynchronous work. As it runs, it gets chopped into **jobs** at each `await`.
- Each **job** runs on an **executor**.
- Which executor runs your job depends on the isolation of the code at that point: MainActor → main, custom actor → that actor's executor, nonisolated → caller's, `@concurrent` → global pool.

So the chain is:

```
Task → splits into jobs at awaits → each job dispatched to an executor based on isolation
```

**Mental model**

> Executor = the scheduler. Actor = the queue. Task = the work being queued.

99% of app code never touches the executor API. Reach for it only when you can name a concrete reason — sharing a queue with legacy code, dedicated threads for real-time work, or server-side pool tuning. Otherwise, the defaults are designed to be invisible.

---

## 19. `[weak self]` Captures in Tasks

### Q
Does it make sense to use `[weak self]` (or other capture patterns) in Tasks?

### A
It really depends on what you are trying to do. It is definitely popular to use `weak self` in this kind of situation. People often want to optimize for the case where they are worried about capturing a reference to self and keeping it alive for too long. If that is your concern, you might use weak self to make sure the task is not what keeps self alive for too long. But you really need to understand what you are trying to accomplish in that task. It might be appropriate for the task to keep self alive until it finishes, because you want to complete that work regardless of whether everything else in the system has released its references to self. It depends on the kind of work. For example, with a task kicked off in a view, you may not want the view to be held onto by the task for too long after it disappears. That might be a good place to use weak self. As with most things, it depends.

One case to be especially careful with is infinitely running tasks. If there is no end to the task, then the task may be what keeps self around. That is also the case where you can inadvertently get into reference cycles, especially if you are also storing a handle to that task on self. But as long as you do not have that kind of situation, it may be appropriate not to use weak self and let the task keep self alive until the work finishes.

### Explain
**Short answer**

It depends on what should happen if `self` is no longer needed before the task finishes.

| Question | Use |
|---|---|
| Should the task be allowed to die when `self` deallocates? | `[weak self]` |
| Must the work complete regardless? | strong capture (default) |

**When to use `[weak self]`**

✅ **Long-running / infinite loops**

The most common case — and a real retain-cycle trap if you skip it.

```swift
final class VM {
    var task: Task<Void, Never>?

    func start() {
        task = Task { [weak self] in       // ✅ weak required
            while !Task.isCancelled {
                guard let self else { return }
                tick()
                try? await Task.sleep(for: .seconds(1))
            }
        }
    }
}
```

If you wrote `Task { ... }` (strong), the cycle is:
- `self` holds `task` →
- `task` holds the closure →
- closure holds `self`.

Infinite loop ⇒ task never returns ⇒ cycle never breaks ⇒ leak.

This is exactly the pattern in your `ARNavigationViewModel.startTracking()` — it already uses `[weak self]` correctly for the tracking loop.

✅ **View-tied work**

Tasks where it's pointless to keep going if the view is gone:

```swift
Task { [weak self] in
    let data = try await api.fetch()
    self?.items = data        // if view's gone, just drop it
}
```

✅ **Tasks stored back on `self`**

Anywhere `self` holds the task handle, lean toward `[weak self]` to break the potential cycle.

**When strong capture is right (the default, no `[weak self]`)**

✅ **Work that MUST complete**

Saves, uploads, analytics flushes. You want `self` kept alive long enough to finish.

```swift
func saveAndDismiss() {
    Task {
        try await store.save(draft)        // strong self — finish even if view dies
        await analytics.log("saved")
    }
}
```

If the view dismisses mid-save, you don't want the save to vanish with it.

✅ **Short-lived tasks**

A 50-ms async call from a button tap doesn't need weak — the object isn't going to deallocate in that window, and the verbosity isn't worth it.

```swift
Button("Tap") {
    Task {
        await vm.handleTap()    // strong is fine
    }
}
```

✅ **Detached / fire-and-forget where lifetime isn't tied to `self`**

If `self` is a service that lives for the app's lifetime anyway, `[weak self]` is just noise.

**The retain cycle to watch for**

This is the one bug `[weak self]` actually prevents:

```swift
final class VM {
    var task: Task<Void, Never>?

    func start() {
        task = Task {                       // ❌ strong self
            while !Task.isCancelled {       // never ends naturally
                ...
            }
        }
    }
}
```

`self → task → closure → self` and the loop never returns. Even calling `task?.cancel()` only helps if the loop body checks cancellation. If you also forget `cancel()` in `deinit`/`onDisappear`, you've got a permanent leak.

Three ways to break it:
1. `[weak self]` in the closure (most common).
2. Make sure the loop ends (cancellation check + actually call `cancel()`).
3. Don't store the task on `self` (use `withTaskGroup` or a `.task` modifier).

**Decision flow**

```
Is the task long-running / infinite?
├── Yes → [weak self]  (and remember to cancel)
└── No  → Is the work tied to a view that may dismiss?
         ├── Yes → [weak self], drop on `nil`
         └── No  → Must this work finish even if owner dies?
                  ├── Yes → strong (default, no [weak self])
                  └── No  → either works; default to strong for short tasks
```

**Mental model**

- Strong capture = "task carries `self` until done"
- Weak capture = "task lets `self` go if nobody else needs it"

The fact that you use `Task { }` doesn't automatically mean you need `[weak self]` — it depends on whether you want to extend `self`'s lifetime through the task or release it.

In your project, the right pattern is already there: `[weak self]` for the continuous tracking loop in `ARNavigationViewModel`, and strong captures elsewhere for short-lived work that should complete. Keep that split.

---

## 20. Generic Codable Function and "Conformance Must Be on Main Thread"

### Q
I have a generic function that takes a Codable object `T` as a parameter, makes a network call to retrieve an array of those objects, and returns an array of `T`. This generates an error in Swift 6 about *"conformance must be on main thread."* What's the best way to avoid this?

### A
I suspect that isolated conformance might be coming into play here. Depending on the features of the target you are building, if you have MainActor isolation by default enabled, it might require that the Decodable conformance you pass is on the main thread, or that the conformance might be MainActor-isolated, but you are passing it to something that does not require that. I would check the documentation. The `SendableMetatype` protocol might be a good annotation to add to this API, depending on whether the conformance is the problem or whether the values being returned are the problem. That could mean using `SendableMetatype` on the type you are passing in, or perhaps using the `sending` keyword on the values you are returning, if you are just fetching them from the network, handing them off to the caller, and never touching them again. I suspect some of those keywords might help express the fact that you are taking a conformance to Codable and perhaps sending it to a different isolation to perform the network request, or taking the values created from that background isolation and sending them back to the calling isolation. In one of those directions, one of those keywords is likely to help resolve it. The context here is really valuable, so providing a sample through Feedback Assistant, checking the forums, and looking at the documentation would help narrow down the exact recommendation.

### Explain
**What's actually happening**

Swift 6.2 introduced **isolated conformances**. When a project has `@MainActor` default isolation (like yours), any type declared in that module gets its protocol conformances pinned to MainActor too.

```swift
@MainActor
struct ParkingSpace: Codable {   // ⚠️ Codable conformance is MainActor-isolated
    let id: String
}
```

So when you write a generic helper that runs off MainActor:

```swift
func fetch<T: Codable>(_ type: T.Type, from url: URL) async throws -> [T] {
    let (data, _) = try await URLSession.shared.data(from: url)  // off-main
    return try JSONDecoder().decode([T].self, from: data)
    //                              ^^^ T's Codable conformance is MainActor-only
}                                       // → "conformance must be on main thread"
```

The decoder needs `T`'s `Decodable` witness, but the witness is locked to MainActor and you're not on MainActor. That's the error.

**Fix 1 (recommended): constrain `T` so the conformance must be non-isolated**

Add `SendableMetatype` to the generic bound. It tells the compiler "the type metadata for `T` (including its conformances) is safe across actors."

```swift
func fetch<T: Codable & SendableMetatype>(
    _ type: T.Type,
    from url: URL
) async throws -> [T] {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode([T].self, from: data)
}
```

Now the call site enforces the requirement:

```swift
try await fetch(ParkingSpace.self, from: url)
// → error if ParkingSpace's Codable conformance is MainActor-isolated
//   forcing you to fix the type, not the helper
```

This is the right API design — generic networking helpers shouldn't care about isolation, so they should refuse types whose conformance is isolated.

**Fix 2: make the type's conformance non-isolated**

If the model is plain data (no MainActor-only logic), opt the conformance out:

```swift
@MainActor
struct ParkingSpace {
    let id: String
}

extension ParkingSpace: Codable {}   // synthesized — picks up MainActor

// Better: explicit nonisolated conformance
nonisolated extension ParkingSpace: Codable {}
```

Or just declare the whole type nonisolated if it's pure data:

```swift
nonisolated struct ParkingSpace: Codable {   // free to cross actors
    let id: String
}
```

For Codable models specifically, `nonisolated` is almost always the right choice — they're value types holding data, with no reason to be MainActor.

**Fix 3: `sending` on the return (different problem)**

The WWDC speaker mentioned `sending` — that's a different fix, for when the *values* (not the conformance) are the issue:

```swift
func fetch<T: Codable & SendableMetatype>(
    _ type: T.Type,
    from url: URL
) async throws -> sending [T] {
    ...
}
```

`sending` says "the returned array has no other references; safely transfer it across isolation domains." Use this when `T` itself isn't Sendable but you can promise the result is freshly-made and not aliased. For most Codable models this isn't needed — Sendable synthesis usually handles it.

**Which fix when**

| Situation | Fix |
|---|---|
| Generic helper, want it usable from anywhere | Fix 1: add `SendableMetatype` to the constraint |
| Specific model that's just data | Fix 2: make the type or its conformance `nonisolated` |
| Type can't be Sendable but result is fresh | Fix 3: `sending` on the return |
| Both apply | Combine: `T: Codable & SendableMetatype` + `nonisolated` on the type |

**Practical recommendation for your project**

You have `@MainActor` as default isolation on your app target. Most of your Codable types (waypoints, anchor tags, parking spaces) are pure data — they have no business being MainActor-isolated. Two-step fix:

1. **For models** — declare them `nonisolated`:
   ```swift
   nonisolated struct Waypoint: Codable, Identifiable { ... }
   nonisolated struct AnchorTag: Codable { ... }
   ```
2. **For generic networking helpers** — constrain on `SendableMetatype`:
   ```swift
   func fetch<T: Decodable & SendableMetatype>(_: T.Type) async throws -> [T] { ... }
   ```

This way the boundary is enforced at the type level: only properly-non-isolated models can flow through your network layer, and the compiler catches violations at the call site instead of inside the helper.

**Mental model**

> A generic constraint says **what** `T` must be.
> An isolated conformance says **where** that conformance can be used.
> If your function uses `T`'s conformance off-actor, you must require the conformance to be non-isolated.

- `SendableMetatype` is the keyword for "I require a non-isolated conformance."
- `nonisolated` is the keyword for "I'm declaring my conformance non-isolated."

They're two sides of the same contract.

---

## 21. Must a `struct` Be Sendable?

### Q
Does a `struct` always have to be Sendable?

### Explain
No — structs are **not required** to be Sendable.

But most of them automatically are, because structs have value semantics. The compiler synthesizes `Sendable` conformance for free when it's safe to do so.

**The rules**

A struct is **implicitly Sendable** when:
1. It's `internal`, `fileprivate`, or `private` (not `public` / `package`).
2. All its stored properties are Sendable types.

That's it. No `: Sendable` annotation needed.

```swift
// ✅ Implicitly Sendable — String and Int are Sendable
struct ParkingSpace {
    let id: String
    let level: Int
}
```

**When a struct is NOT Sendable**

Holding any non-Sendable property breaks it:

```swift
class Counter { var x = 0 }           // class with var → not Sendable

struct Bad {
    let counter: Counter              // ❌ Bad is NOT Sendable
}

struct Holder {
    let callback: () -> Void          // ❌ closures aren't Sendable by default
}

struct WithUIImage {
    let icon: UIImage                 // ✅ UIImage IS Sendable (Apple marked it)
}
```

The compiler checks each stored property. One non-Sendable field → the whole struct loses its free `Sendable` conformance.

**Public / package structs need explicit declaration**

Across module boundaries the compiler can't auto-synthesize, because the conformance becomes part of your API:

```swift
// In an SPM library
public struct Waypoint {              // ⚠️ NOT Sendable across modules
    public let id: String
}

public struct Waypoint: Sendable {    // ✅ explicit, callers can rely on it
    public let id: String
}
```

This matters for your project — anything declared `public` or `package` in your SPM targets (Model, PathfindingService, etc.) needs an explicit `: Sendable` if you want it usable across actor boundaries.

**Common misconceptions**

❌ "Sendable means immutable." No — `var` stored properties are fine, because struct mutation creates a new value. This is Sendable:

```swift
struct Counter: Sendable {
    var value: Int = 0    // var is OK for structs
}
```

❌ "Value types are always Sendable." No — a struct holding a non-Sendable class is itself non-Sendable.

❌ "You must annotate every struct." No — internal structs with all-Sendable properties get it for free.

**Quick check for your codebase**

In your Model SPM target, anything that crosses isolation domains (network results, AR positions, parking spaces, waypoints) should be:

```swift
public struct Foo: Sendable, Codable {
    public let id: String
}
```

Or, if it's a nonisolated value type with all-Sendable fields, just adding `: Sendable` is the contract that matches the question from earlier (`SendableMetatype` constraints) — without it, generic helpers can't accept your model.

**Mental model**

- Struct + all Sendable fields + internal/private → Sendable for free
- Struct + public/package → must write `: Sendable` explicitly
- Struct + any non-Sendable field → not Sendable, period

Structs aren't required to be Sendable, but in practice you want most of your data structs to be — that's how they cross actor boundaries without `await` ceremony or `sending` annotations.
