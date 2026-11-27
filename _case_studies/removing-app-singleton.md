---
title: Removing a singleton that wasn't one
summary: Replacing 122 reads of a global AppContext.shared with explicit dependencies, and finding out the "singleton" had been handing some callers a throwaway instance all along.
hero: /assets/images/di-overhaul/hero.svg
hero_alt: "Launch timeline: a placeholder AppContext exists until reset(loadData:) creates the configured one. Early reads of the placeholder are logged per call site. The configured context reaches screens through constructor injection in destination wrappers and long-lived services through configure(appContext:)."
role: Senior iOS Developer · RipeConcepts, Inc. · Sole developer
timeline: 7 weeks · 2026
platforms: [iOS 17+]
stack: [Swift, SwiftUI, Dependency injection, Swift concurrency, XCTest]
order: 2.5
featured: false
---

<p class="muted"><em>Part 2 of 2. This builds on <a href="/case-studies/navigation-stack-migration/">Moving a retail app to NavigationStack</a>, whose destination wrappers became the place dependencies are injected from.</em></p>

<div class="stats">
  <div><strong>122 → 3</strong><span><code>AppContext.shared</code> reads in production code</span></div>
  <div><strong>57 → 2</strong><span>files that touch the singleton</span></div>
  <div><strong>10</strong><span>PRs over 7 weeks, each shippable alone</span></div>
  <div><strong>5</strong><span>app-lifetime services moved to late binding</span></div>
  <div><strong>1</strong><span>revert, then shipped properly</span></div>
</div>

**TL;DR.** The app had a global `AppContext.shared` that every layer reached into for its cart, lists, rewards and networking. Removing it looked like routine cleanup. It turned out the "singleton" was a placeholder created at launch and replaced once remote config loaded, so anything that read it too early had been holding a second, orphaned copy of app state. I moved the codebase to explicit dependencies in ten small PRs, instrumented the remaining reads in production before deciding what to do with them, and shipped, reverted and re-shipped the one change that got the initialization order wrong.

*Code on this page is rewritten from the production source: names are simplified and business details removed. The structure and the decisions are real.*

## Context

After the [NavigationStack migration](/case-studies/navigation-stack-migration/), every screen was built inside a small destination wrapper that received an `AppContext` from the environment. That gave the app a composition root for the first time, a single place where a screen gets its dependencies. But most of the code below that point ignored it and read `AppContext.shared` directly: 122 references across 57 files, in views, controllers, managers, networking, logging and analytics.

The stated goal was testability (you can't substitute a mock for something a view fetches from a global) and groundwork for Swift 6 strict concurrency, which rejects unprotected global mutable state. The project builds in Swift 5 mode with several Swift 6 upcoming features already on, so the global was one of the things standing between it and the switch.

## The problem: the singleton wasn't single

<p class="code-label before">Before: a global that gets swapped out after launch</p>

```swift
final class AppContext: ObservableObject {
  /// Created the first time anything touches it, long before remote config has loaded.
  static var shared = AppContext.make(loadData: false)

  /// Called once launch has fetched remote config and the storefront is known.
  static func reset(loadData: Bool) -> AppContext {
    let context = AppContext.make(loadData: loadData)
    shared = context
    return context
  }
}
```

`reset` replaces the instance. Code that reads `AppContext.shared` *at the time of use* is fine. Code that reads it *once and keeps the result* is not, and Swift makes that easy to do by accident:

```swift
struct RecommendationsGrid: View {
  init(controller: RecommendationsController,
       // Evaluated wherever and whenever the view is built. Usually after reset. Not always.
       listsController: ListsController = AppContext.shared.listsController) { … }
}
```

A default argument that reads a global hides two things from the caller: that a dependency exists at all, and *which instance* they'll get. Anything that captured the placeholder kept working against a cart, a lists controller and a rewards controller that the rest of the app never looked at. Nothing crashed. State just quietly went nowhere.

## Constraints

1. **Feature work didn't stop.** Two-week releases kept going, so every step had to be mergeable on its own, without a long-lived branch.
2. **One developer, 57 files.** Reviewers needed diffs they could actually read, so each PR covered one layer.
3. **Some things really are app-lifetime.** Deep-link routing, push-notification handling and the login service exist before any screen does and are reached from the app delegate and system callbacks. They can't receive dependencies through a view's initializer.
4. **The launch order was fixed.** Remote config has to load before the real context can be built, and that wasn't up for redesign.

## The approach: outside in, one layer per PR

| PR | Layer | Notes |
|---|---|---|
| 1 | Straightforward view call sites | 18 files: prove the pattern |
| 2 | Destination wrappers, network controller | 40 files: wrappers become the composition root |
| 3 | Daily deals, reviews, cart | Each screen gets its own network controller so loading spinners stop leaking between screens |
| 4 | Remaining feature views | Fixed the favorites regression (decision 2) |
| 5 | Controllers, managers, networking, logging, analytics | 54 files, +935 / −360; `configure` methods for cart and Apple Pay managers |
| 6 | **Revert:** login service dependency capture | See *The revert* below |
| 7 | Cart manager fix | Removed a duplicate cart refresh in the rewards-membership flow |
| 8 | Deep links, push notifications, instrumentation | Production logging of early reads, plus guarded late binding |
| 9 | Login and push notifications, properly | `configure(appContext:)` with a regression test |
| 10 | App extensions | 45 files |

Starting at the views meant each PR pushed the boundary one layer further from the UI. By the time I reached the managers, every caller above them was already passing dependencies explicitly, so a manager's new initializer parameter had somewhere to come from.

## Key decisions and tradeoffs

<div class="decision" markdown="1">

### 1. Measure the remaining reads before removing them

**Problem:** after the first five PRs, the obvious call sites were gone, and the rest were in code that runs at launch, from system callbacks, or on paths QA rarely hits. I couldn't tell by reading which ones ever saw the placeholder in practice.

**Chose:** make the accessor tell me. The backing store keeps the placeholder, the configured instance is marked, and any read that gets the placeholder is logged to production monitoring once per call site:

<p class="code-label after">After: the global reports its own misuse</p>

```swift
private static nonisolated(unsafe) var _shared = AppContext.make(loadData: false)
fileprivate(set) var isConfigured = false

static var shared: AppContext {
  let context = _shared
  if !context.isConfigured {
    logUnconfiguredAccess()
  }
  return context
}

@inline(never)
private static func logUnconfiguredAccess() {
  // Return addresses are stable within a launch and far cheaper than symbolicating.
  let callerAddress = Thread.callStackReturnAddresses.dropFirst(2).first?.uintValue ?? 0
  let isNewCallSite = lock.withLock { loggedCallSites.insert(callerAddress).inserted }
  guard isNewCallSite else { return }

  // Only symbolicate the first time a given call site is seen.
  let callSite = Thread.callStackSymbols.dropFirst(2).first ?? "unknown"
  Logger.shared.warning(.appContextAccessedBeforeConfiguration, source: callSite)
}
```

**Why:** it turned "remove every reference" into a ranked list. Call sites that never fired could wait. The ones that did went to the top. It also caught anything new that reintroduced the pattern.

**Cost:** a lock and a stack walk on the hot path for unconfigured reads only (configured reads pay one `Bool` check), plus `nonisolated(unsafe)` on the backing store, which is an honest label for what it is. I kept the cost low by deduplicating on raw return addresses and only symbolicating new ones.

</div>

<div class="decision" markdown="1">

### 2. Required dependencies, not optional ones

**Regression:** the heart (favorite) button disappeared from recommendation grids and dynamic-content rows. In the middle of the migration, several views had taken lists as an optional:

<p class="code-label before">Before: the compiler can't tell you a caller forgot</p>

```swift
struct RecommendationsGrid: View {
  let listsController: ListsController?

  init(controller: RecommendationsController, listsController: ListsController? = nil) { … }

  var body: some View {
    // …
    if let listsController {
      FavoriteButton(product: product, listsController: listsController)
    }
  }
}
```

Every call site that didn't pass it still compiled, and the button just never rendered. Nothing crashed and nothing was logged.

**Chose:** dependencies are non-optional with no default. The destination wrapper supplies them, and every intermediate view passes them on:

<p class="code-label after">After: forgetting it is a build error</p>

```swift
struct RecommendationsGrid: View {
  let listsController: ListsController

  init(controller: RecommendationsController, listsController: ListsController) { … }
}

// The destination wrapper is the only place that knows where it comes from
RecommendationsGrid(controller: controller, listsController: appContext.listsController)
```

**Cost:** more parameters on intermediate views that only pass things along. I accepted that. A long initializer is visible in review, while a missing button only shows up when someone happens to look.

</div>

<div class="decision" markdown="1">

### 3. Late binding for things that outlive the view tree

**Problem:** the deep-link manager is a static type, and the login service is an `actor` exposed as `static let shared`. Both exist before the configured context does. Constructor injection doesn't work when you don't control the constructor's timing.

**Chose:** an explicit `configure(appContext:)` step, called once from launch right after the context is built, with a guard at every use:

```swift
// Launch, after remote config: build the context once, then hand it out.
let appContext = AppContext.reset(loadData: true)
CartManager.shared.configure(couponWallet: appContext.couponWallet, cart: appContext.cart)
DeepLinkManager.configure(appContext: appContext)
await LoginService.shared.configure(appContext: appContext)
PushNotificationService.configure(appContext: appContext)
```

```swift
private static func requireAppContext(caller: String = #function) -> AppContext? {
  guard let appContext else {
    // A deep link arriving before launch finishes is a real case; log it, don't crash.
    Logger.shared.error(.deepLinkDependencyMissing, source: caller)
    return nil
  }
  #if DEBUG
  // Catches the original bug class: holding a context that reset() has since replaced.
  assert(appContext === AppContext.shared || !AppContext.shared.isConfigured,
         "Stale AppContext. Call configure(appContext:) after reset.")
  #endif
  return appContext
}
```

**Why:** it makes the order explicit in one place, at launch, instead of leaving it to whenever a default argument happened to run. A missing dependency is an expected condition with a log entry rather than a silent placeholder.

**Cost:** it's still temporal coupling: these types are valid only after a call that the compiler doesn't enforce. The guard and the debug assert are compensating controls, not a fix. See *What I'd do differently*.

</div>

<div class="decision" markdown="1">

### 4. Leave the last three reads on purpose

**Chose:** stop at three real call sites, in the scene delegate and a UIKit navigation-bar protocol extension, rather than chasing zero.

**Why:** both are UIKit entry points with no initializer I own and no view above them to inject from. Removing them meant either another `configure` step for a protocol extension or restructuring scene setup. Both were bigger and riskier than the problem. The instrumentation from decision 1 covers them, so if either ever reads the placeholder in production, it shows up in monitoring.

**Cost:** the global still exists, so the Swift 6 blocker isn't fully gone. I chose to document that and keep it visible rather than claim the job was finished.

</div>

## The revert

This is the part I'd want a reviewer to read.

In PR 5, I gave the login service proper initializer injection, the same pattern that had worked for 40 files of views:

<p class="code-label before">Shipped, then reverted</p>

```swift
actor LoginService {
  static let shared = LoginService(networkController: NetworkController())

  private let cart: CartDataController
  private let lists: ListsController
  private let rewards: RewardsController

  init(networkController: NetworkController,
       cart: CartDataController = AppContext.shared.cart,
       lists: ListsController = AppContext.shared.lists,
       rewards: RewardsController = AppContext.shared.rewards) { … }

  func logout() async {
    cart.cart = nil
    lists.reset()
    rewards.memberDetails = nil
  }
}
```

It looks right. But `static let shared` is created lazily on first access, and first access can come before `reset`. When it did, the default arguments captured the placeholder's controllers. Logout then cleared a cart, a list and a rewards profile that no screen was showing, and the user's real ones stayed on screen after sign-out.

**What I did, in order:**

1. **Reverted first** (PR 6), back to reading the global at the time of use. That was ugly but correct, and it put a login and logout path that touches user data back into a known-good state before I tried anything clever.
2. **Added the instrumentation** (decision 1) so I could see which other call sites had the same exposure in production, instead of guessing.
3. **Re-shipped with late binding** (PR 9):

<p class="code-label after">Shipped properly</p>

```swift
actor LoginService {
  static let shared = LoginService(networkController: NetworkController())

  private var appContext: AppContext?

  func configure(appContext: AppContext) {
    self.appContext = appContext
  }

  func logout() async {
    guard let appContext else {
      Logger.shared.warning(.appContextAccessedBeforeConfiguration, source: #function)
      return
    }
    await MainActor.run {
      appContext.cart.cart = nil
      appContext.lists.reset()
      appContext.rewards.memberDetails = nil
    }
  }
}
```

4. **Pinned it with a test** that sets real state, configures the service, logs out, and asserts the *configured* context was cleared:

```swift
func testLogoutClearsCartListsAndRewards() async throws {
  let context = AppContext.shared
  context.cart.cart = try XCTUnwrap(Stub.cart())
  context.lists.hasFetchedInitialData = true

  let sut = makeSUT()
  await sut.configure(appContext: context)
  await sut.logout()

  XCTAssertNil(context.cart.cart)
  XCTAssertFalse(context.lists.hasFetchedInitialData)
  XCTAssertNil(context.rewards.memberDetails)
}
```

The lesson I took from it: **a default argument on a lazily created shared object is an initialization-order bug waiting to happen.** It's the same mechanism as the favorites regression, appearing somewhere far more dangerous. Both times the fix was the same: make the dependency impossible to receive implicitly.

## Results

- **122 → 3** `AppContext.shared` reads in production code, across 57 → 2 files. Most of the 11 remaining text matches are comments and the debug assert.
- **The original bug class is detectable.** Any read of the placeholder is reported from production, deduplicated per call site, and stale references held by long-lived services trip an assert in debug builds.
- **Screens can be tested with mocks.** Views and controllers take their dependencies through `init`, so tests pass mocks (new ones for lists and coupon wallet) instead of mutating global state. About 48 test files were updated to construct their subjects explicitly.
- **A duplicate cart refresh** in the add-rewards-membership flow was found and fixed along the way.
- **Closer to Swift 6.** The global mutable state that strict concurrency would reject is down to one clearly labelled `nonisolated(unsafe)` store and three known readers. That's groundwork for the switch, not the switch itself.

## What I'd do differently

- **Instrument first, not in week five.** The logging in decision 1 would have flagged the login service's exposure before I wrote the version that had to be reverted. On the next migration like this, the measurement goes in the first PR.
- **Replace `configure` with real injection for app-lifetime services.** `static let shared` plus `configure(appContext:)` is safer than a default argument, but it's still a two-step initialization the compiler can't check. The next step is creating these services in the app's launch path and passing them down, so "not configured yet" stops being a possible state.
- **Make the deep-link manager an instance.** It's still a static type with a stored context. As an injected instance, URL routing could be tested end to end without global setup or `resetForTesting()`.
- **Then turn on Swift 6 mode.** With the global down to one store and three readers, strict concurrency checking is the forcing function that would finish the job.

<p class="muted"><em>Thanks to QA and to my teammates for reviewing ten PRs of initializer changes while shipping their own work.</em></p>
