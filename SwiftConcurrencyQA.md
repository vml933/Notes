 ▐▛███▜▌   Claude Code v2.1.129
▝▜█████▛▘  Opus 4.7 · Claude API
  ▘▘ ▝▝    ~/Documents/XCodeProject/ParkingLotNavigate

❯ there is a qa statement:                                                      
                                                                                
  "What’s the difference between nonisolated and nonisolated(nonsending), and   
  why was the latter introduced?"                                               
                                                                                
  Nonisolated means that a declaration has no preference for isolation. It does 
   not have a fixed isolation, so it can be used in multiple isolation          
  contexts. Nonisolated non-sending is about asynchronous functions. There was  
  a behavior originally implemented for asynchronous functions in Swift         
  concurrency where asynchronous functions would move to the concurrent thread  
  pool, the shared executor, by default. We found that this was not the right   
  trade-off as people adopted Swift concurrency and we learned more. So         
  nonisolated non-sending, which is now the default behavior if you opt in      
  using the approachable concurrency settings, is the new behavior where your   
  asynchronous task or asynchronous function will not leave the isolation it    
  was called from right away, unless there is an explicit reason to.            
                                                                                
  it say a lot, but i still not get it. can u tell me with example ?            

⏺ nonisolated vs nonisolated(nonsending)

  The key thing to grasp: this only matters for async functions. For sync
  functions, nonisolated behaves the same in both worlds.

  The old behavior (plain nonisolated async)

  When you call a nonisolated async function, Swift hops off your actor to the 
  global concurrent pool, runs the function there, then hops back.

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

  Why this was painful:
  1. Sendable headaches — anything you pass into fetch() had to be Sendable,
  because it was crossing isolation boundaries.
  2. Surprising hops — looks like a normal call, but actually causes a thread
  switch.
  3. Worse perf in many cases — you didn't need the work off the main actor, but
   you paid the hop cost anyway.

  The new behavior (nonisolated(nonsending))

  The async function stays on the caller's actor unless it explicitly opts out.

  class Fetcher {
      nonisolated(nonsending) func fetch() async -> [String] {
          // If called from MainActor → runs on MainActor.
          // If called from some other actor → runs on that actor.
          // If called from nowhere (a Task) → runs nonisolated.
          return ["a", "b"]
      }
  }

  Now await fetcher.fetch() from a @MainActor context does not leave MainActor.
  No hop, no Sendable check on the arguments, no surprise.

  When you actually want to hop off

  If fetch() does heavy CPU work and you want it off MainActor, be explicit:

  class Fetcher {
      nonisolated func heavyWork() async -> [String] {
          // Plain `nonisolated` async = old behavior = runs off-actor.
      }
  }

  Or use Task.detached { ... } at the call site.

  TL;DR

  ┌───────────────────────────┬────────────────────┬───────────────────────┐
  │           Form            │  Async function    │       Use when        │
  │                           │      runs on…      │                       │
  ├───────────────────────────┼────────────────────┼───────────────────────┤
  │ nonisolated (sync)        │ caller's context   │ normal sync helpers   │
  │                           │ (no hop)           │                       │
  ├───────────────────────────┼────────────────────┼───────────────────────┤
  │ nonisolated(nonsending)   │ caller's actor (no │ new default — most    │
  │ async                     │  hop)              │ async APIs            │
  ├───────────────────────────┼────────────────────┼───────────────────────┤
  │ nonisolated async         │ global concurrent  │ you genuinely want to │
  │                           │ pool (hop)         │  leave the actor      │
  └───────────────────────────┴────────────────────┴───────────────────────┘

  The reason for the change: the old default was the wrong default. Most async
  functions don't need to leave the caller's actor, and forcing the hop made
  Sendable errors and threading behavior much harder to reason about. Under
  "Approachable Concurrency" (Swift 6.x), nonisolated(nonsending) becomes the
  default for async, so you usually don't even write it — you write nonisolated
  and get the new behavior.

✻ Churned for 24s

❯ does this nonisolated(nonsending) available for all Swift 6.x ?               

⏺ No — it shipped in Swift 6.1 (Xcode 16.3, ~April 2025), via SE-0461
  (https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-f
  unction-isolation.md). Swift 6.0 doesn't have it.

  Quick map:

  ┌─────────────────┬───────────────────────────────────────────────────────┐
  │      Swift      │                        Status                         │
  ├─────────────────┼───────────────────────────────────────────────────────┤
  │ 6.0             │ ❌ Not available                                      │
  ├─────────────────┼───────────────────────────────────────────────────────┤
  │ 6.1             │ ✅ Syntax available; opt-in as default via upcoming   │
  │                 │ feature flag                                          │
  ├─────────────────┼───────────────────────────────────────────────────────┤
  │ 6.2 (your       │ ✅ Available; default-on if you enable Approachable   │
  │ project)        │ Concurrency                                           │
  ├─────────────────┼───────────────────────────────────────────────────────┤
  │ 7.0 (future)    │ ✅ Becomes the language default                       │
  └─────────────────┴───────────────────────────────────────────────────────┘

  To make plain nonisolated async use the new behavior without writing
  (nonsending) everywhere, enable the upcoming feature in your target:

  // Package.swift
  .target(
      name: "MyTarget",
      swiftSettings: [
          .enableUpcomingFeature("NonisolatedNonsendingByDefault")
      ]
  )

  Or in Xcode build settings: Other Swift Flags → -enable-upcoming-feature 
  NonisolatedNonsendingByDefault.

  In Xcode 26+ this is bundled under the "Approachable Concurrency" build
  setting (single toggle that flips on this and a few related upcoming
  features).

  Without the flag, you must write nonisolated(nonsending) explicitly to get the
   new behavior — plain nonisolated async still hops to the global pool.

✻ Cooked for 16s

❯ next qa:                                                                      
  "When marking a ViewModel with @MainActor how should we handle a method       
  inside it that performs heavy I/O processing? How do process outside of       
  MainActor/Main Thread?"                                                       
  "Yes, if you have a particular operation on any MainActor-isolated type and   
  you want only that operation to be offloaded, you can annotate just that      
  method with a different isolation. For example, if you want to offload it to  
  the concurrent thread pool, you can now use the @concurrent attribute. That   
  is the right explicit spelling for it. The compiler will prevent you from     
  directly accessing any MainActor mutable state on that view model, or         
  whatever your MainActor-isolated type is, to make sure you are not accessing  
  that state at the same time as another operation on that type is accessing    
  the same mutable state from the MainActor. So take a look at @concurrent.     
  There are also a lot of references from last year’s WWDC, where many uses of  
  @concurrent were covered as well."                                            
                                                                                
  i don't get it                                                                

⏺ The problem

  You have a @MainActor ViewModel. Every method runs on the main thread. But one
   method does something heavy — image processing, parsing a big JSON, hashing a
   file. You don't want that blocking the UI.

  @MainActor
  final class ImageViewModel {
      var thumbnail: UIImage?

      func makeThumbnail(from data: Data) async {
          // ⚠️ Runs on MainActor → blocks the UI while it crunches.
          let image = expensiveResize(data)
          thumbnail = image
      }
  }

  The fix: @concurrent

  @concurrent is the explicit opt-out. It tells the compiler: "this one method
  should run on the global concurrent pool, not on MainActor."

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

  What the WWDC quote was trying to say

  Three pieces:

  1. Default — methods on a @MainActor type run on MainActor.
  2. @concurrent opts ONE method off — without removing @MainActor from the
  whole class.
  3. The compiler protects you — once that method is off-actor, it physically
  cannot read or write self.thumbnail directly. You must await MainActor.run { 
  ... } (or call back into a MainActor method). This prevents data races on the
  ViewModel's mutable state.

  Why this is better than the old tricks

  Old patterns people used:

  // Old: Task.detached
  func makeThumbnail(from data: Data) async {
      let image = await Task.detached {
          expensiveResize(data)  // data must be Sendable
      }.value
      thumbnail = image
  }

  // Old: a separate nonisolated helper
  nonisolated func resize(_ data: Data) -> UIImage { ... }

  Both work, but they push the off-actor work into a closure or a separate
  function. @concurrent lets the whole method body be the off-actor work, while
  still being a method on your ViewModel — cleaner shape, same safety.

  The mental model

  ┌──────────────────────────────────┬──────────────────────────────────────┐
  │    Annotation on a method of     │         Where the body runs          │
  │         @MainActor class         │                                      │
  ├──────────────────────────────────┼──────────────────────────────────────┤
  │ (none)                           │ MainActor (main thread)              │
  ├──────────────────────────────────┼──────────────────────────────────────┤
  │ nonisolated (sync)               │ Caller's context                     │
  ├──────────────────────────────────┼──────────────────────────────────────┤
  │ nonisolated async (Swift 6.2     │ Caller's actor — usually still       │
  │ default)                         │ MainActor                            │
  ├──────────────────────────────────┼──────────────────────────────────────┤
  │ @concurrent async                │ Global concurrent pool — definitely  │
  │                                  │ off MainActor                        │
  └──────────────────────────────────┴──────────────────────────────────────┘

  So the rule of thumb: for a heavy async method on a MainActor type, mark it 
  @concurrent, do the work, then await MainActor.run { ... } to write results 
  back.

✻ Sautéed for 26s

❯ about the '@concurrent', does it has any limitation, like the data in this    
  fucn must Sendable or something                                               

⏺ Yes, @concurrent comes with real restrictions — they're the price of crossing
  an isolation boundary safely.

  1. Must be async

  @concurrent func foo() { }        // ❌ error: only async functions
  @concurrent func foo() async { }  // ✅

  2. Parameters must be Sendable (or sending)

  You're passing data from the caller's actor → the global pool, so it has to be
   safe to ship.

  final class NotSendable { var x = 0 }     // reference type, mutable, not 
  Sendable

  @MainActor
  final class VM {
      @concurrent
      func bad(_ thing: NotSendable) async { } // ❌ error: NotSendable not 
  Sendable

      @concurrent
      func ok(_ data: Data) async { }           // ✅ Data is Sendable

      @concurrent
      func transfer(_ thing: sending NotSendable) async { }  // ✅ ownership 
  moved
  }

  sending is the escape hatch: "I promise I'm handing this off — nobody else
  holds a reference."

  3. Return values must be Sendable (or sending)

  Same reason, the other direction.

  @concurrent
  func makeImage() async -> UIImage { ... }     // ✅ UIImage is Sendable

  @concurrent
  func makeVM() async -> SomeViewModel { ... }  // ❌ if SomeViewModel isn't 
  Sendable
                                                //    use `-> sending 
  SomeViewModel`

  4. Can't touch self's actor-isolated state directly

  This is the big one people trip on:

  @MainActor
  final class VM {
      var items: [String] = []

      @concurrent
      func reload() async {
          let fresh = await fetch()
          items = fresh                    // ❌ error: can't access MainActor 
  state
          await MainActor.run {
              items = fresh                // ✅ hop back to write
          }
      }
  }

  You can still call other MainActor methods (those are async hops, the compiler
   handles it):

  @concurrent
  func reload() async {
      let fresh = await fetch()
      await apply(fresh)   // ✅ apply() is a MainActor method, await hops back
  }

  func apply(_ items: [String]) {
      self.items = items
  }

  5. Captured values in closures follow the same Sendable rules

  @concurrent
  func work(notSendable: NotSendable) async {
      Task {
          notSendable.x += 1   // ❌ capture must be Sendable
      }
  }

  6. Mutually exclusive with isolation attributes

  @concurrent @MainActor func foo() async { }   // ❌ pick one

  @concurrent means "not isolated to anything" — combining it with @MainActor is
   a contradiction.

  Mental model

  @concurrent punches a hole in your MainActor class. Inside the hole:
  - Inputs/outputs must be Sendable (or sending).
  - self's state is read-only via await-hops, never direct.
  - You're on the global pool, so heavy work is fine — but you've left the
  safety of the actor.

  If a method's signature can't be made Sendable cleanly, that's usually a
  signal it shouldn't be @concurrent — extract the pure-data part into a
  nonisolated helper or a free function instead.

✻ Cooked for 31s

❯ /model 
  ⎿  Kept model as Opus 4.7

❯ next qa:                                                                      
  "When a Task is created from @MainActor, will that block the main thread and  
  cause UI hitches?"                                                            
  "A task being created from MainActor is something you probably encounter      
  often when working on a UI framework. I think that when a task is created on  
  MainActor, there is still a suspension point that the compiler creates, so    
  while that work is being performed, the main thread can still continue        
  working on UI updates and avoid causing hitches. In general, the task         
  machinery tries to schedule your work on the concurrent thread pool so that   
  it does not block excessively. That is one of the nice behaviors that tasks   
  provide. I think it depends on what the task does. If you create a task that  
  is isolated to MainActor and that task starts running on MainActor, and you   
  are doing some very expensive work that takes a long time, then it might      
  block MainActor, and you might notice that. In that case, you may want to     
  profile that code in Instruments and then decide whether to insert some       
  suspension points or offload parts of that expensive task from MainActor. I   
  really recommend that you do not worry about proactively offloading           
  everything you can from MainActor. There are some things that are totally     
  fine to run on MainActor. But if you notice hangs in your app and identify    
  that piece of code through Instruments or your profiling tool of choice, that 
   is when you should make targeted changes to move the expensive work off."    
                                                                                
  this talk so much, i don't get it                                             

⏺ Short answer

  Creating a Task is cheap and never blocks. What's inside it can block — but
  only if it does heavy synchronous work without await points.

  What actually happens

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

  Three cases for what's inside:

  Case 1 — Has await points → fine ✅

  Task {
      let data = await api.fetch()   // ← suspends here
      label = "got \(data.count)"
  }

  At the await, the main thread is released. UI keeps animating, scrolling,
  responding to taps. When fetch() returns, the task resumes on main, sets
  label, done. No hitch.

  Case 2 — Pure heavy sync work, no await → blocks ❌

  Task {
      // No suspension points. Main thread is held the whole time.
      let result = primesUpTo(10_000_000)   // 2 seconds of CPU
      label = "\(result.count)"
  }

  This is identical to just running the code on main directly. The Task { }
  wrapper does not magically move it off the main thread. UI freezes for 2
  seconds.

  Case 3 — Want to offload the heavy bit → use @concurrent or Task.detached

  @concurrent
  func compute() async -> Int {
      primesUpTo(10_000_000).count    // runs on global pool
  }

  Task {
      let count = await compute()     // suspends main, main stays responsive
      label = "\(count)"              // back on main, fast
  }

  What the WWDC speaker meant

  - "Task machinery tries to schedule your work on the concurrent thread pool" →
   true for Task.detached and @concurrent/nonisolated async calls. Not true for
  plain Task { } from MainActor — that one stays on main.
  - "Compiler creates a suspension point" → he means the await points you write.
   The compiler doesn't insert them for you.
  - "Don't proactively offload everything" → most code on main is fine (setting
  properties, small calculations, layout work). Only profile-then-fix when you
  see actual hangs.

  Rule of thumb

  ┌──────────────────────────────────────────────────────┬───────────────────┐
  │                  What's in the Task                  │   Blocks main?    │
  ├──────────────────────────────────────────────────────┼───────────────────┤
  │ await someAsyncCall()                                │ No                │
  ├──────────────────────────────────────────────────────┼───────────────────┤
  │ Tiny sync work (set a property, format a string)     │ No, imperceptible │
  ├──────────────────────────────────────────────────────┼───────────────────┤
  │ Tight loop / parsing big JSON / image work / hashing │ Yes — offload it  │
  └──────────────────────────────────────────────────────┴───────────────────┘

  So: don't be afraid of Task { ... } from MainActor. Be afraid of heavy sync 
  code with no await, wherever it lives.

✻ Sautéed for 25s

❯ i have a question about '@concurrent', we only foucs using '@concurrent' in   
  '@MainActor' before, is any practice using '@concurrent' in Actor ? does this 
   approach meaningful ?                                                        

⏺ Yes — @concurrent works on any actor (custom actor, custom global actor,
  @MainActor). It's the same mechanism. The interesting question is when it's 
  worth doing.

  What makes a custom actor different from MainActor

  ┌───────────────────────────────┬────────────┬────────────────────────────┐
  │                               │ MainActor  │        Custom actor        │
  ├───────────────────────────────┼────────────┼────────────────────────────┤
  │ Backed by                     │ Main       │ A serial executor (some    │
  │                               │ thread     │ background queue)          │
  ├───────────────────────────────┼────────────┼────────────────────────────┤
  │ Blocking it freezes UI?       │ Yes        │ No                         │
  ├───────────────────────────────┼────────────┼────────────────────────────┤
  │ Blocking it hurts throughput  │ Yes        │ Yes — other actor calls    │
  │ of other callers?             │            │ queue up                   │
  └───────────────────────────────┴────────────┴────────────────────────────┘

  So the urgency for offloading is lower for custom actors, but the same kind of
   problem exists: a sync-heavy method monopolizes the actor and stalls everyone
   else waiting on it.

  A meaningful example

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

  If 10 callers ask for different images at once, they all queue behind each
  other because decodeSync is sync work holding the actor.

  Fix:

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

  Now while one image is decoding on the global pool, the actor is free to
  handle other image(for:) calls — check the cache, kick off more fetches, etc.
  Throughput goes up.

  When @concurrent on a custom actor is meaningful

  ✅ Heavy CPU work that doesn't need the actor's state
  Decoding, hashing, parsing, transforming bytes, image/audio processing.

  ✅ The actor has many concurrent consumers
  Caches, request coordinators, database wrappers — anything multiple callers
  hit at once. Holding the actor on sync work serializes them all.

  ✅ The method is essentially a pure function
  It only uses its parameters. The fact that it lives on the actor is
  organizational, not functional.

  When it's NOT meaningful

  ❌ Method reads/writes actor state a lot
  Every access becomes await MainActor.run { ... }-equivalent (hop back to the
  actor). Net result: slower than just running on the actor.

  actor Bank {
      var balances: [String: Int] = [:]

      @concurrent
      func transfer(from: String, to: String, amount: Int) async {
          // ❌ Bad fit: every line needs a hop back to read/write balances.
      }
  }

  ❌ Method is small / fast
  The hop-off-then-hop-back overhead exceeds the work. Just run it on the actor.

  ❌ Method is mostly awaits already
  Actors don't block during await — other methods get to run at suspension
  points. Offloading buys nothing.

  Mental model

  Think of @concurrent as "this method shouldn't hold the actor's serial 
  executor while it runs." The question to ask:

  ▎ Does this method spend significant time doing synchronous work that doesn't 
  ▎ touch my actor's state?

  If yes → @concurrent is meaningful.
  If no → leave it on the actor.

  For MainActor the bar is lower (any blocking is bad because UI). For custom
  actors the bar is "would queueing behind this hurt my callers' latency?" —
  usually only true for caches, coordinators, and other multi-consumer actors.

✻ Churned for 37s

❯ next qa:                                                                      
  "When should I reach for at concurrent instead of nonisolated async? Is there 
   a one sentence rule?"                                                        
  "@concurrent is guaranteed to run on the shared thread pool in the            
  background. For nonisolated async, it depends on your upcoming feature        
  settings. As described earlier, the one-sentence explanation is that you      
  should use @concurrent when you want your async function to be offloaded to   
  the concurrent thread pool. It is an explicit spelling for behavior that used 
   to be implicit for async functions, but that behavior is changing because it 
   was not the best trade-off and not the best default for async functions. So  
  you should use @concurrent now when you want that behavior. With nonisolated  
  async, it depends on whether you have the approachable concurrency upcoming   
  features enabled in your Xcode project, which is the default for new Xcode    
  projects. If you do not have it enabled, I recommend enabling those settings  
  because it gives you a set of defaults that are a bit easier to use with the  
  data safety diagnostics."                                                     

⏺ The one-sentence rule

  @concurrent = "always run me off-actor."
  nonisolated async (with Approachable Concurrency) = "run me wherever the 
  caller is."

  That's the whole difference. @concurrent is a guarantee; nonisolated async is
  adaptive.

  Side-by-side

  struct Helper {
      @concurrent
      func parseHeavy(_ data: Data) async -> Model { ... }

      nonisolated
      func parseLight(_ data: Data) async -> Model { ... }
  }

  Calling them from different places:

  @MainActor func fromMain() async {
      await parseHeavy(data)   // ✅ runs on global pool (off main)
      await parseLight(data)   // ✅ runs on MainActor (stays on main)
  }

  actor Cache {
      func fromActor() async {
          await parseHeavy(data)   // ✅ runs on global pool (off Cache)
          await parseLight(data)   // ✅ runs on Cache's executor (stays on 
  Cache)
      }
  }

  Same two functions, different behaviors based on caller — except @concurrent
  is caller-independent.

  How to choose

  Ask yourself: "If this function is called from MainActor, do I want it to 
  leave MainActor?"

  ┌──────────────────────────────────────────────────────┬──────────────────┐
  │                        Answer                        │       Use        │
  ├──────────────────────────────────────────────────────┼──────────────────┤
  │ Yes, always — heavy CPU/IO that must not block the   │ @concurrent      │
  │ caller                                               │                  │
  ├──────────────────────────────────────────────────────┼──────────────────┤
  │ No, stay where you are — light glue code, mostly     │ nonisolated      │
  │ awaits                                               │ async            │
  ├──────────────────────────────────────────────────────┼──────────────────┤
  │ Depends on the caller — let them decide              │ nonisolated      │
  │                                                      │ async            │
  └──────────────────────────────────────────────────────┴──────────────────┘

  Why this matters

  Before SE-0461, nonisolated async had the @concurrent behavior baked in
  (always hopped to global pool). That was the wrong default — most async
  helpers just await other things and have no reason to leave the caller. The
  new design splits the two:

  - Default-safe (nonisolated async): inherit caller, no surprises.
  - Explicit-offload (@concurrent): you meant to leave.

  Practical rule of thumb

  ▎ Reach for @concurrent when the function name has a verb like parse, decode, 
  ▎ encode, compress, hash, resize, render, compute. Reach for nonisolated async
  ▎  when the function is mostly await someOtherThing() plumbing.

  And yes — turn on Approachable Concurrency in your Xcode project if it isn't
  already. Without it, plain nonisolated async still has the old hop-to-pool
  behavior, and the rule above breaks.

✻ Baked for 21s
❯ next qa:                                                                      
  "Is it possible to have too many actors? Is there an intrinsic limit?"        
  "There is probably an intrinsic limit depending on how much memory you have,  
  but I would definitely think about limiting how many different actors you     
  have in your program. Generally speaking, you want to have just a few         
  isolation domains in a program to reason about. Concurrent programming is     
  hard, as we said earlier, and most of your code should be synchronous unless  
  you have a reason to introduce concurrency and a reason to introduce          
  different isolation domains. Every actor is a different isolation domain, and 
   if actors need to work together, that creates complexity in your program. So 
   you really want to think carefully when you factor something out into an     
  actor, and decide whether it is really appropriate to add the complexity of   
  needing to access that code asynchronously from other parts of your program.  
  I think a very common question is: when do you actually use an actor? The     
  answer is specifically when you can define that something will always need    
  its own isolated domain, and the entirety of its functionality requires you   
  to maintain that isolation. On the slide, I had a couple of examples like     
  network caching or accessing a database. Those are common things where you    
  often already know the work is going to be in an isolated domain anyway."     

⏺ Short answer

  Yes — too many actors is a real problem, but the limit is cognitive, not 
  technical. Each actor is a separate isolation domain, and every boundary
  between domains adds an await, a Sendable check, and a chunk of complexity.

  The real rule

  ▎ Most of your code should be plain synchronous code. Only reach for an actor 
  ▎ when something genuinely needs its own isolation domain for all of its 
  ▎ operations.

  Good actor candidates

  These have shared mutable state with multiple concurrent consumers:

  actor ImageCache { ... }       // many screens read/write
  actor TokenStore { ... }       // refresh + read race each other
  actor Database { ... }         // every query touches shared connection
  actor RequestCoordinator { ... } // dedupes in-flight network calls

  The pattern: shared + mutable + accessed from multiple places + must stay 
  consistent.

  Bad actor candidates

  People often introduce actors when they don't need to:

  ❌ A ViewModel → use @MainActor class. It's already isolated, and it usually
  drives UI which lives on main anyway.

  ❌ A pure helper / utility → just a struct with static func or free functions.
   No state, no race, no actor needed.

  ❌ A stateless service (e.g. a thin wrapper around URLSession) → struct or
  class with nonisolated methods. There's nothing to protect.

  ❌ Something only one place uses → if there's a single owner, you don't have a
   concurrency problem yet. Wait until you do.

  ❌ A "manager" that holds config → struct of let properties. Immutable =
  trivially safe, no actor.

  The cost you pay per actor

  Every time non-actor code talks to an actor:

  let value = await cache.image(for: url)  // ← await, suspension, scheduling

  - Caller must be async (or in a Task)
  - Inputs/outputs must be Sendable
  - A suspension point appears — state can change while you're waiting
  - Logic that crosses two actors needs careful ordering ("did A change while I
  awaited B?")

  Multiply that by every actor → boundaries → await → reasoning load. Five
  well-chosen actors are easier to maintain than fifteen accidental ones.

  A simple decision test

  Before making something an actor, ask:

  1. Does it have mutable state? No → struct or sync class.
  2. Is that state accessed from multiple concurrent contexts? No → @MainActor
  or single owner.
  3. Does every method need that protection? No → maybe just one method needs
  locking; reconsider the design.
  4. All three yes? → actor is the right tool.

  Mental model

  Think of actors like locks. You wouldn't sprinkle locks across every class
  "just in case." You add one when there's a real shared resource that needs
  guarding. Actors are the same — a deliberate concurrency boundary, not a
  default container.

  So: a handful of actors, chosen on purpose. The rest of your app: synchronous
  code, @MainActor for UI, and async functions only where you actually do I/O or
   heavy work.

✻ Churned for 27s

❯ next qa:                                                                      
  "Could a Task started by .task modifier in SwiftUI view be cancelled if the   
  view is refreshed by state change?"                                           
  "The task modifier in SwiftUI cancels your task when the view disappears, and 
   state changes usually do not cause your view to be fully destroyed by        
  SwiftUI. State works in a way where, when the state changes, SwiftUI          
  reevaluates the view’s body, but it does not actually destroy the view. One   
  case where you may see this behavior is if you have an ifstatement in your    
  view’s body, and one branch becomes true while the other branch previously    
  contained a view with a task. Once the condition flips, SwiftUI would cancel  
  the task because the view in the other branch of the if statement is          
  destroyed. You can also think about it like onDisappear: task is a modifier,  
  and the task modifier manages cancellation with the same timeline as the      
  onDisappear modifier, if you have used that before."                          

⏺ Short answer

  No — a state change that just re-renders the view does NOT cancel .task. The
  task is only cancelled when SwiftUI actually removes the view from the 
  hierarchy.

  The mental model

  .task ↔ onAppear / onDisappear lifetime, not body re-evaluation lifetime.

  - State changes → body re-runs → .task keeps running.
  - View removed from hierarchy → .task is cancelled.

  Examples

  ✅ State change — task is NOT cancelled

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

  Tapping the button re-evaluates body 100 times — loadStuff() runs once and is
  never interrupted.

  ❌ View removed via if branch — task IS cancelled

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

  When showChild becomes false, SwiftUI destroys Child → its .task is cancelled.

  ❌ Other lifecycle endings that cancel .task

  - View popped from NavigationStack
  - Sheet / fullScreenCover dismissed
  - View removed from a ForEach (item deleted from data)
  - Tab switched away (depending on tab style — sometimes the view is destroyed)

  Bonus: .task(id:) — restarts on id change

  .task(id: userId) {
      await loadProfile(userId)   // cancelled & restarted whenever userId 
  changes
  }

  Use this when you want the task to react to a value change — e.g. reload when
  the user navigates to a different profile.

  The rule

  ┌──────────────────────────────────────────────────┬──────────────────────┐
  │                      Event                       │   .task cancelled?   │
  ├──────────────────────────────────────────────────┼──────────────────────┤
  │ @State changes, body re-evaluates                │ ❌ No                │
  ├──────────────────────────────────────────────────┼──────────────────────┤
  │ Parent passes new props, body re-evaluates       │ ❌ No                │
  ├──────────────────────────────────────────────────┼──────────────────────┤
  │ View removed (if flip, pop, dismiss, ForEach     │ ✅ Yes               │
  │ delete)                                          │                      │
  ├──────────────────────────────────────────────────┼──────────────────────┤
  │ .task(id:) and the id value changes              │ ✅ Yes (and          │
  │                                                  │ restarts)            │
  └──────────────────────────────────────────────────┴──────────────────────┘

  So you can lean on .task for "run this for as long as the view is on screen" —
   it's tied to the view's lifetime, not its renders.

✻ Cogitated for 21s

❯ next qa:                                                                      
  "Intuitively what does the nonisolated keyword mean? Especially after the     
  recent changes."                                                              
  "What has not really changed necessarily, but is now defaulted in some cases, 
   is that nonisolated is a statement of your intent to not require any         
  particular isolation for a type or function. When you annotate a type as      
  @MainActor, you are stating that you only want it to be used from MainActor.  
  nonisolated is removing that restriction. You can use it from any isolation   
  domain, and that is really flexible and powerful. It is often the right       
  choice for code in libraries, where you do not know where the app will want   
  to use your library types. It can also be appropriate in your app for         
  low-level models that do not have any inherent need to be isolated to         
  MainActor or some other isolation domain. That is part of what motivated the  
  behavior change for nonisolated with async functions. When something is       
  marked as nonisolated, you can use it from anywhere. If you have a method     
  that is not isolated and you call it from somewhere, it is going to run in    
  the isolation domain it was called from. That used to not be the behavior for 
   async functions, but now it is. So now there is a consistent meaning: when   
  you see nonisolated, it means you can use it from anywhere, and when you call 
   those methods, they are going to run wherever you call them from."           

⏺ The intuition

  nonisolated = "I have no opinion about where I run. Call me from wherever."

  It's the opposite of @MainActor (which says "you can ONLY call me from
  MainActor").

  ┌─────────────┬──────────────────────────────────────────────────┐
  │ Annotation  │                     Meaning                      │
  ├─────────────┼──────────────────────────────────────────────────┤
  │ @MainActor  │ "You MUST be on MainActor to call me"            │
  ├─────────────┼──────────────────────────────────────────────────┤
  │ actor       │ "You MUST hop into me to call me"                │
  ├─────────────┼──────────────────────────────────────────────────┤
  │ nonisolated │ "Call me from anywhere — I run wherever you are" │
  └─────────────┴──────────────────────────────────────────────────┘

  Why "after the recent changes" matters

  Before SE-0461, nonisolated was inconsistent between sync and async:

  // OLD behavior (Swift 6.0)
  nonisolated func a() { }              // runs on caller's context ✅ 
  consistent
  nonisolated func b() async { }        // hops to global pool ❌ surprising

  That asymmetry was confusing. Now it's clean:

  // NEW behavior (Swift 6.2 + Approachable Concurrency)
  nonisolated func a() { }              // runs on caller's context ✅
  nonisolated func b() async { }        // runs on caller's context ✅

  So now you can read nonisolated and have one mental model: "runs wherever you 
  call it from."

  When to reach for nonisolated

  ✅ Library/SDK code — you don't know where the app will use it, so don't
  impose @MainActor on consumers.

  public struct ImageDecoder {
      public nonisolated func decode(_ data: Data) -> UIImage? { ... }
  }

  ✅ Low-level model types — pure data, Codable, value types. They have no
  business being pinned to an actor.

  struct ParkingSpace: Codable {
      nonisolated func formattedID() -> String { ... }
  }

  ✅ Helper methods on a @MainActor class that don't touch state

  @MainActor
  final class ViewModel {
      var items: [Item] = []

      nonisolated func validate(_ id: String) -> Bool {
          // doesn't read self.items, so no need to be on MainActor
          return id.count == 8 && id.allSatisfy(\.isLetter)
      }
  }

  Now validate is callable from a background thread, an actor, anywhere — no
  await needed.

  When NOT to use it

  ❌ The method touches self's isolated state — compiler will block you.

  @MainActor
  final class VM {
      var items: [Item] = []

      nonisolated func clear() {
          items = []   // ❌ can't access MainActor state from nonisolated
      }
  }

  ❌ You actually want guaranteed off-actor execution — that's @concurrent, not
  nonisolated.

  The one-line summary

  nonisolated = "no isolation requirement, runs wherever the caller is."

  After SE-0461, that meaning is finally consistent for both sync and async —
  which is the whole point of the change.

✻ Crunched for 20s

❯ next qa:                                                                      
  "For a new app target, should I adopt MainActor default isolation? Does your  
  answer change for libraries?"                                                 
  "n Xcode 26, all new app targets use MainActor isolation by default. As       
  someone from the SwiftUI team, I think that is an absolutely amazing choice   
  for app targets because, in our UI framework, views are already on MainActor. 
   If you have written a SwiftUI view and also have a model type, you may have  
  noticed that it is often convenient to put that model on MainActor as well.   
  With MainActor by default turned on, you do not have to do that manually. It  
  makes UI code much easier to write and reason about, and you do not have to   
  think as much about synchronizing your state because everything is on         
  MainActor. But when it comes to libraries, my answer would change, depending  
  on what the library does. If you are trying to be performant, concurrency can 
   help you optimize behavior in your code by parallelizing operations, and     
  libraries can make use of that in some cases. If you find that an operation   
  is taking too long and you want to split it up, then it can make sense to     
  introduce more concurrency, especially in library code. The same applies to   
  app targets as well: you start on MainActor, and once you see potential for a 
   performance optimization, or you profile the app and see the main thread     
  being blocked by an expensive operation in your model, then you can consider  
  adding concurrency annotations and moving things off MainActor. I also think  
  many general-purpose library APIs are not specific to any particular          
  isolation domain. Most of Foundation is not isolated, and clients of          
  Foundation can choose where they want to use those APIs. For example, with    
  Foundation, most APIs are not specific to any isolation, so most of them are  
  nonisolated. There are a few APIs that are specific, such as UndoManager,     
  which is heavily UI-focused and annotated with MainActor. For Foundation, it  
  makes sense to keep most things nonisolated by default and add MainActor      
  isolation for UI-related APIs. But when you are working on an app, it is      
  almost the inverse: most of your things are on MainActor, and you may want to 
   selectively offload work from there."                                        

⏺ Short answer

  App target → YES, use MainActor default.
  Library → NO, default to nonisolated.

  The two have opposite needs.

  Why YES for apps

  In Xcode 26, new app targets ship with SWIFT_DEFAULT_ACTOR_ISOLATION = 
  MainActor — exactly what your project already has set in CLAUDE.md. That
  means:

  // No annotation needed — implicitly @MainActor
  final class ParkingViewModel {
      var spaces: [Space] = []
      func refresh() { ... }
  }

  Wins:
  - Views are already on MainActor, so models flow naturally with them — no
  await to bridge.
  - No Sendable boilerplate for things that only ever touch UI.
  - Less to think about — one isolation domain by default; introduce more only
  when you have a reason.
  - Performance is rarely the bottleneck on day one — most app code is glue and
  small operations that are fine on main.

  The intended workflow: start on MainActor → profile → selectively offload the
  hot spots with @concurrent or a real actor.

  Why NO for libraries

  A library doesn't know where it'll be used. If you stamp @MainActor on
  everything:

  @MainActor
  public struct ImageDecoder {
      public func decode(_ data: Data) -> UIImage? { ... }
  }

  …then every server-side, command-line, or background-task consumer is forced
  onto MainActor for no reason. That's a leaky concurrency design.

  Default to nonisolated, let the caller pick:

  public struct ImageDecoder {
      public nonisolated func decode(_ data: Data) -> UIImage? { ... }
  }

  Now an iOS app can call it from MainActor, a server can call it from a request
   handler, a CLI can call it from a @main function — none of them get isolated
  for free.

  The Foundation pattern

  Apple's own libraries follow this:

  ┌────────────────────────────────┬───────────────────────┬───────────────┐
  │              API               │       Isolation       │      Why      │
  ├────────────────────────────────┼───────────────────────┼───────────────┤
  │ URLSession, JSONDecoder, Date, │ nonisolated           │ usable from   │
  │  String                        │                       │ anywhere      │
  ├────────────────────────────────┼───────────────────────┼───────────────┤
  │ UndoManager                    │ @MainActor            │ inherently    │
  │                                │                       │ tied to UI    │
  ├────────────────────────────────┼───────────────────────┼───────────────┤
  │                                │ nonisolated, but      │               │
  │ NotificationCenter.publisher   │ main-thread variants  │ flexible      │
  │                                │ exist                 │               │
  └────────────────────────────────┴───────────────────────┴───────────────┘

  The rule they applied: default nonisolated, opt into MainActor only for 
  genuinely UI-bound APIs. Mirror this in your own libraries.

  Mental model

  App target:     "everything on MainActor; offload the hot bits"
  Library:        "nothing isolated; let the caller decide"

  For your ParkingLotNavigate app: stick with MainActor default ✅
  If you ever extract AprilTagDetectorImpl or PathfindingService into a reusable
   SPM package: flip those to nonisolated APIs and let the host app decide where
   to call them from.

✻ Cogitated for 23s

❯ next qa:                                                                      
  "If a class launches a long-running Task and calls task.cancel() in deinit,   
  does the task actually stop executing? And is its memory released immediately 
   after cancellation? Also, are tasks automatically cancelled when their       
  owning scope is deallocated?"                                                 
                                                                                
  "Cancellation in Swift concurrency is cooperative. Depending on what the task 
   is doing, that operation needs to check for cancellation. Usually, for a     
  library method or similar API to support cancellation, the course of action   
  is to throw a cancellation error. If you are running long synchronous code    
  and someone cancels the task handle, that synchronous code is not going to be 
   interrupted at any moment. That is actually an important behavior, because   
  the work might need to perform some cleanup operation that must happen        
  atomically without being interrupted by something like task cancellation. So  
  no, the task does not necessarily stop running immediately because of         
  cancellation. But as soon as you hit a piece of code that handles             
  cancellation and maybe throws a cancellation error, that is when it will      
  usually return by throwing such an error. There was another question about    
  whether tasks get cancelled when their owning scope is deallocated. If a      
  class has a task handle, like we discussed earlier, and you want to track a   
  task to cancel it, you need to store it as a property. But no, it is not      
  automatically cancelled just because the owning scope is deallocated. That is 
   one of the gotchas when you opt into managing your task’s lifetime: you      
  always have to remember to cancel it. That is also good in the sense that you 
   are being explicit about the task’s lifetime. There was also a question      
  about tasks swallowing errors. A Task does not require handling of thrown     
  errors, which can lead to silently ignored failures. There is now an accepted 
   proposal that addresses that exact pain point. In Swift 6.4, there will be a 
   new diagnostic for exactly that case, where you have a task and have not     
  handled errors within it. There are different ways to handle errors within a  
  task. You can handle any thrown errors inside the task body, or if the task   
  itself can throw and you want to keep a handle to the task, you can wait      
  until you get the task’s value and handle any potential errors that were      
  thrown. Those are two different ways to handle task errors. So yes, with the  
  acceptance of that proposal, you will now get diagnostics for this. If you    
  are interested, you can check the Swift Forums or Swift.org Swift Evolution   
  for the accepted proposed behavior."                                          

⏺ Three questions, three answers

  1. Does task.cancel() stop the task immediately?

  No. Cancellation in Swift is cooperative — cancel() just sets a flag. The task
   only actually stops if its body checks that flag.

  let task = Task {
      for i in 0..<1_000_000_000 {
          heavyMath(i)               // ⚠️ pure sync — never checks cancellation
      }
  }
  task.cancel()                       // sets flag; the loop above keeps 
  grinding

  vs. cooperatively-cancellable code:

  let task = Task {
      for i in 0..<1_000_000_000 {
          try Task.checkCancellation() // ✅ throws if cancelled
          heavyMath(i)
      }
  }

  What automatically responds to cancellation:
  - try await on most stdlib/Foundation async APIs (URLSession, Task.sleep,
  etc.) — they throw CancellationError.
  - Explicit try Task.checkCancellation() calls.
  - Task.isCancelled checks you write yourself.

  What does not:
  - Plain synchronous loops, math, file reads, image decoding.

  This is intentional — sometimes the work needs to finish a cleanup atomically
  and shouldn't be yanked.

  2. Is memory released immediately after cancel()?

  No. The Task object lives until its body actually returns. Cancellation just
  requests an early return; the body has to honor it. Until then, the task (and
  everything it captures) stays alive.

  let task = Task { [weak self] in
      while !Task.isCancelled { ... }   // returns soon after cancel() → freed
  }

  If you capture self strongly inside the task body, self is retained until the
  task ends — even after cancel().

  3. Are tasks auto-cancelled when their owner deallocates?

  No — and this is one of the biggest Swift concurrency gotchas.

  final class VM {
      var task: Task<Void, Never>?

      func start() {
          task = Task { while !Task.isCancelled { ... } }
      }

      deinit {
          task?.cancel()   // ✅ MUST do this manually
      }
  }

  If you forget the cancel() in deinit, the task keeps running (and keeps self
  alive if it captured strongly → deinit never runs in the first place 😱).

  Your ARNavigationViewModel already does this correctly — taskStorage.dispose()
   in onDisappear is the same pattern.

  The exception: structured tasks

  async let, withTaskGroup, and SwiftUI's .task modifier are auto-cancelled when
   their scope ends. Those are structured concurrency. Plain Task { } is
  unstructured — you own its lifetime.

  ┌─────────────────────────────┬───────────────────────────────┐
  │            Form             │        Auto-cancelled?        │
  ├─────────────────────────────┼───────────────────────────────┤
  │ async let x = ...           │ ✅ When enclosing scope exits │
  ├─────────────────────────────┼───────────────────────────────┤
  │ withTaskGroup { ... }       │ ✅ When group exits           │
  ├─────────────────────────────┼───────────────────────────────┤
  │ .task { } SwiftUI modifier  │ ✅ When view leaves hierarchy │
  ├─────────────────────────────┼───────────────────────────────┤
  │ Task { ... } (unstructured) │ ❌ You must store + cancel    │
  └─────────────────────────────┴───────────────────────────────┘

  Bonus: silently-swallowed errors

  Task {
      try await riskyThing()   // ⚠️ if this throws, error is silently dropped
  }

  The compiler doesn't warn today. SE-0485 (accepted, landing in Swift 6.4) will
   add a diagnostic for this. Two valid fixes:

  // Handle inside
  Task {
      do { try await riskyThing() }
      catch { logger.error("\(error)") }
  }

  // Or keep the handle and await the value elsewhere
  let task = Task { try await riskyThing() }
  do { _ = try await task.value } catch { ... }

  Mental model

  cancel() = "please stop when convenient"
  NOT     = "kill -9"

  Three things to remember:
  1. Cancellation is cooperative → write try await / Task.checkCancellation() at
   suspension points.
  2. Unstructured tasks need manual lifetime management → store, cancel() in
  deinit/onDisappear.
  3. Task { try await ... } swallows errors today → handle them, or wait for
  Swift 6.4's warning.

✻ Cogitated for 32s

❯ next qa: "What’s the best practice for asynchronous requests in response to a 
   button tap? Kicking off a one-off task? Exposing a synchronous method from   
  your model that kicks off a task internally?"                                 
                                                                                
  "The general recommendation, although you can structure your code in both     
  ways, is that I would prefer moving that asynchronous code to your model.     
  Then the model can respond back with synchronous output to your view, so the  
  view stays responsive and the model handles all of that asynchronous work.    
  The benefit of this approach is that you can also test the asynchronous work  
  outside of your view, which brings greater benefits to your entire codebase." 

⏺ Recommendation: put the async work in the model

  The view should call a synchronous method on the model. The model owns the
  Task internally.

  ❌ Inline approach (don't do this)

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

  Why it's worse:
  - Untestable — concurrency lives in the view; you can't unit-test the flow
  without driving the SwiftUI lifecycle.
  - No lifetime control — view has no handle to cancel, dedupe, or track the
  task.
  - Concurrency leaks into UI code — every button does its own Task { }
  boilerplate.
  - Race-prone — tap twice, get two parallel requests.

  ✅ Model owns the task

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

  What you get:

  ┌─────────────────────┬───────────────────────────────────────────────────┐
  │         Win         │                        How                        │
  ├─────────────────────┼───────────────────────────────────────────────────┤
  │ Testable            │ await vm.loadTask?.value in tests, then assert    │
  │                     │ vm.spaces                                         │
  ├─────────────────────┼───────────────────────────────────────────────────┤
  │ Cancelable          │ loadTask?.cancel() on rapid re-taps or            │
  │                     │ onDisappear                                       │
  ├─────────────────────┼───────────────────────────────────────────────────┤
  │ Loading state       │ isLoading is owned by the model, view just reads  │
  │                     │ it                                                │
  ├─────────────────────┼───────────────────────────────────────────────────┤
  │ Errors handled      │ Centralized in one place, can surface via         │
  │                     │ vm.errorMessage                                   │
  ├─────────────────────┼───────────────────────────────────────────────────┤
  │ View stays          │ No Task { } clutter in the body                   │
  │ declarative         │                                                   │
  └─────────────────────┴───────────────────────────────────────────────────┘

  When the inline form is OK

  For genuinely fire-and-forget UI-local actions where the model doesn't care:

  Button("Share") {
      Task { await UIActivityFeedback.success() }
  }

  But once there's any state (isLoading, error, results to display), move it to
  the model.

  Mental model

  ▎ The view declares what should happen on tap.
  ▎ The model decides how (and when, and whether to cancel a previous one).

  This is the same separation you already use elsewhere — putting the Task in
  the model is just extending it to async behavior.

✻ Crunched for 20s

❯ next qa: "What is the best practice to write a unit test for async await      
  code?"                                                                        
                                                                                
  "You can use Swift Testing and make your test method async, then test it      
  there. If you need that method to run with a particular isolation, you can    
  mark the test itself or the entire test case type with MainActor, or with     
  whatever isolation you need that test to run on. Swift Testing is definitely  
  worth checking out if you have not already."                                  

⏺ Use Swift Testing — async tests are first-class

  In Swift Testing, just mark the test async and await directly. No
  XCTestExpectation, no wait(for:), no expectation dance.

  Basic async test

  import Testing

  @Test
  func fetchesSpaces() async throws {
      let api = MockAPI(spaces: [.fixture(id: "A1"), .fixture(id: "A2")])
      let result = try await api.fetchSpaces()
      #expect(result.count == 2)
      #expect(result.first?.id == "A1")
  }

  Testing a @MainActor model

  When the code under test is MainActor-isolated, mark the test (or whole suite)
   @MainActor:

  @MainActor
  @Test
  func loadingFlipsCorrectly() async throws {
      let vm = ParkingViewModel(api: MockAPI(...))

      vm.loadSpaces()                       // sync call, kicks off internal 
  Task
      #expect(vm.isLoading == true)

      try await vm.loadTask?.value          // wait for the task to finish
      #expect(vm.isLoading == false)
      #expect(vm.spaces.count == 2)
  }

  Or pin the whole suite once:

  @MainActor
  @Suite
  struct ParkingViewModelTests {
      @Test func loadsSpaces() async throws { ... }
      @Test func handlesError() async throws { ... }
      @Test func cancelsPreviousLoad() async throws { ... }
  }

  Testing the "model owns the Task" pattern (from your previous Q)

  The trick: expose the Task (or a done() helper) so tests can await it.
  Otherwise tests become flaky polling loops.

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

  Testing errors

  @Test
  func throwsOnNetworkFailure() async {
      let api = MockAPI(error: URLError(.notConnectedToInternet))
      await #expect(throws: URLError.self) {
          try await api.fetchSpaces()
      }
  }

  Testing actor methods

  Just await them — no special setup:

  @Test
  func cacheStoresAndReturns() async {
      let cache = ImageCache()
      await cache.store(image: .test, for: URL(string: "x")!)
      let got = await cache.image(for: URL(string: "x")!)
      #expect(got != nil)
  }

  Anti-patterns to avoid

  ❌ Task.sleep to "wait for it" — flaky, slow, hides race conditions.

  vm.loadSpaces()
  try await Task.sleep(for: .seconds(1))   // ❌ guess and pray
  #expect(vm.spaces.count == 2)

  ✅ Expose the task handle and await its .value.

  ❌ XCTestExpectation in async tests — you don't need it. Just await.

  ❌ Using DispatchQueue.main.async in tests — bridge old code to async with
  await MainActor.run { ... } or restructure.

  Mental model

  ▎ If your production code is async, your test should be async too — let them 
  ▎ meet in the same world.

  Three rules:
  1. @Test + async throws for the test signature.
  2. Match isolation — @MainActor on the test/suite when the code is MainActor.
  3. Make tasks awaitable — expose handles or completion signals so tests are
  deterministic, not timed.

✻ Crunched for 26s

❯ next qa: "What is best use case of Task.detached?"                            
                                                                                
  " have seen a lot of questions recently about when to use Task.detached       
  versus when to use Task { @concurrent in }, because now those two things have 
   very similar behaviors. The main difference is that the plain Task           
  initializer, even when you specify some isolation, still inherits certain     
  things from the surrounding context. Obviously, if you have explicitly        
  specified an isolation, it does not inherit isolation, but it does inherit    
  priority from the surrounding context. Task.detached, on the other hand, is   
  completely detached. Nothing from the surrounding context is inherited by     
  that task. So if you want something that is completely detached from the      
  surrounding context, use Task.detached. If you just want to control the       
  isolation, but you still want those other things to be inherited, use Task {  
  @concurrent in }."                                                            

⏺ Short answer

  Use Task.detached when you specifically want to cut off all inheritance from
  the surrounding context — not just isolation, but also priority and task-local
   values.

  For "I just want to run off the actor" → use Task { @concurrent in ... }
  instead. It's the new, more focused tool.

  What gets inherited

  @MainActor
  func tapped() {
      Task { ... }                 // inherits: isolation (Main), priority, 
  task-locals
      Task { @concurrent in ... }  // inherits: priority, task-locals  (NOT 
  isolation)
      Task.detached { ... }        // inherits: NOTHING
  }

  ┌─────────────────────────┬──────────────────┬──────────┬─────────────┐
  │                         │    Isolation     │ Priority │ Task-locals │
  ├─────────────────────────┼──────────────────┼──────────┼─────────────┤
  │ Task { }                │ ✅               │ ✅       │ ✅          │
  ├─────────────────────────┼──────────────────┼──────────┼─────────────┤
  │ Task { @concurrent in } │ ❌ (global pool) │ ✅       │ ✅          │
  ├─────────────────────────┼──────────────────┼──────────┼─────────────┤
  │ Task.detached { }       │ ❌               │ ❌       │ ❌          │
  └─────────────────────────┴──────────────────┴──────────┴─────────────┘

  When Task.detached is the right call

  ✅ Background work that shouldn't ride a high-priority context

  @MainActor
  func userTapped() {
      // Tap handler is .userInitiated. We don't want telemetry to share that.
      Task.detached(priority: .background) {
          await analytics.flush()
      }
  }

  If you used plain Task { @concurrent in }, the flush would inherit
  .userInitiated and compete with real user work. Detaching breaks the chain.

  ✅ Starting a fresh task-local scope

  If you use @TaskLocal (request IDs, trace IDs, logging contexts),
  Task.detached gives you a clean slate:

  @TaskLocal static var requestID: UUID?

  func handleIncoming() async {
      await Self.$requestID.withValue(UUID()) {
          Task.detached {
              // requestID is nil here — fresh context
              await processInBackground()
          }
      }
  }

  ✅ Long-lived background loops decoupled from any caller

  A monitoring loop, a periodic sync — work whose lifetime shouldn't track
  whoever happened to start it.

  When NOT to use it

  ❌ "I just want off MainActor" → use @concurrent:

  // Old habit
  Task.detached { let r = decode(data); await MainActor.run { vm.image = r } }

  // Better
  Task { @concurrent in
      let r = decode(data)
      await MainActor.run { vm.image = r }
  }

  The @concurrent version still inherits priority — usually what you want, so a
  UI tap's decode runs at user-initiated priority instead of dropping to
  default.

  ❌ Default choice → don't reach for detached "to be safe." Cutting off
  priority and task-locals can starve the work or hide bugs in your
  tracing/logging.

  Mental model

  Task.detached = "this work is its own creature — don't tie it to me"
  Task { @concurrent in } = "run this off-actor, but it's still my work"

  Reach for Task.detached when you can name the inheritance you want to break —
  priority chain, or task-locals. If you can't name one, you probably want
  @concurrent instead.
