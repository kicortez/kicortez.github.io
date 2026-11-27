---
title: Moving a retail app to NavigationStack
summary: Replacing a UIKit navigation layer that fired view controllers at whatever was on screen with a typed, per-tab NavigationStack router, in a live multi-brand app, without freezing feature work.
hero: /assets/images/navigation-refactor/hero.svg
hero_alt: "Architecture diagram: entry points call router.push(route, on: tab); AppRouter keeps a NavigationPath per tab; TabNavigationStack resolves each route through RouteResolver into a destination wrapper that injects dependencies into the screen."
role: Senior iOS Developer · RipeConcepts, Inc. · Architecture owner
timeline: 11 weeks · 2026
platforms: [iOS 17+]
stack: [SwiftUI, NavigationStack, UIKit interop, Dependency injection, XCTest]
order: 2
featured: true
---

<p class="muted"><em>Part 1 of 2. Continued in <a href="/case-studies/removing-app-singleton/">Removing a singleton that wasn't one</a>, which uses the destination wrappers below to remove a global singleton.</em></p>

<div class="stats">
  <div><strong>36</strong><span>typed routes replace ad-hoc pushes</span></div>
  <div><strong>39 → 8</strong><span>direct UIKit push call sites</span></div>
  <div><strong>13 → 0</strong><span>deprecated <code>NavigationView</code>s</span></div>
  <div><strong>44</strong><span>router unit tests</span></div>
</div>

**TL;DR.** I designed and led the move of a large multi-brand shopping app from UIKit's `UINavigationController` to SwiftUI's `NavigationStack`. The core is a typed `Route` enum, one `NavigationPath` per tab owned by a single `AppRouter`, and a resolver layer that also became the seam for removing a global singleton. I wrote the base architecture and 96 of the 131 commits on the migration branch, and teammates moved their feature areas onto it in parallel. It shipped behind a normal release with no change to how users move through the app, which was the point.

*Code on this page is rewritten from the production source: names are simplified and business details removed. The structure and the decisions are real.*

## Context

The app serves several retail brands from one codebase, each a separate App Store app with its own scheme and remote config. It is mostly UIKit, with SwiftUI adopted screen by screen for about four years. By 2026 most screens were SwiftUI views, but each one was still wrapped in a `UIHostingController` and pushed onto a `UINavigationController` (a subclass, in fact, that existed only to keep swipe-back from freezing the UI).

So the screens were SwiftUI and the navigation was still UIKit. Every new SwiftUI screen paid that tax.

## The problem

Navigation had no model. "Go to X" meant *find whatever UIKit thinks is on top right now, build a view controller, and push it.* That logic was copied into dozens of places.

<p class="code-label before">Before: navigating meant guessing at the view hierarchy</p>

```swift
// DeepLinkManager: one of 39 call sites that pushed UIKit controllers directly
static func goToProductPage(id: String?, optionID: String? = nil) {
  guard let productID = id else { return }

  // "Is this product already showing?" Answered by down-casting whatever is on top.
  if let top = tabBarTopViewController as? ProductDetailHostingController,
     top.rootView.controller.productID == productID {
    return
  }
  // "Is it one screen back?" Answered by indexing into UIKit's stack.
  else if let nav = activeTabBarNavigationController,
          let previous = nav.viewControllers[safe: nav.viewControllers.count - 2]
            as? ProductDetailHostingController,
          previous.rootView.controller.productID == productID {
    nav.popToViewController(previous, animated: true)
    return
  }

  let screen = ProductDetailHostingController(appContext: AppContext.shared, productID: productID, optionID: optionID)
  pushToViewController(screen)
}

private static func pushToViewController(_ vc: UIViewController, animated: Bool = true) {
  let nav = (tabs.tabBarController?.selectedViewController ?? rootViewController) as? UINavigationController
  // If that cast fails, nothing happens: no error, no log, no navigation.
  nav?.pushViewController(vc, animated: animated)
}
```

The symptoms, in order of cost:

- **Navigation state couldn't be inspected.** The only answer to "what is on this tab's stack?" was the view controller tree, so deep links had to cast their way through it. Cast against the wrong type and the link did nothing, silently.
- **Construction and navigation were tangled.** About two dozen `UIViewController` extension helpers (`showHelp()`, `showOrderDetails()`, `showRewardsMembership()`, …) each built its own hosting controller and pulled dependencies from `AppContext.shared`. You couldn't unit-test any of them without a window.
- **SwiftUI screens couldn't navigate natively.** A SwiftUI button had to reach back out through `DeepLinkManager.tabBarTopViewController?.navigationController`. Thirteen screens still used the deprecated `NavigationView`.
- **The singleton was load-bearing.** `AppContext.shared` appeared 122 times in production code, much of it there because the navigation layer had no other way to hand a screen its dependencies.

## Constraints

These shaped every decision more than the API did:

1. **A live app shipping releases every two weeks.** Feature teams kept merging to `master` the whole time; the branch took in `master` more than 30 times. A rewrite-and-freeze was never an option.
2. **The tab bar stays UIKit.** It is remote-config driven (tab order and content vary per brand), and rewriting it was out of scope.
3. **Some screens stay UIKit for now.** Product detail, cart and package tracking are large hosting controllers with their own lifecycles. They had to be routable without being rewritten.
4. **Deep links are the contract.** Push notifications, universal links, the search bar and marketing URLs all go through one deep-link manager, and every path in the URL routing table had to keep working.
5. **Regression runs on real devices.** Nightly UI suites per brand on BrowserStack query the accessibility tree, so the change had to keep identifiers and back-button labels stable.

## The architecture

{% include figure.html src="/assets/images/navigation-refactor/hero.svg" alt="Entry points call push on AppRouter, which owns one NavigationPath per tab. TabNavigationStack binds to that path and resolves routes via RouteResolver into destination wrappers that inject dependencies into the screen." caption="One way in (`push(_:on:)`), one source of truth, one place that turns a route into a screen." %}

There are four pieces, each with one job.

### 1. `Route`: a typed vocabulary for "where"

```swift
/// Every place the app can push to. Hashable so it can live in a NavigationPath.
enum Route: Hashable {
  case productDetail(productID: String, optionID: String?)
  case search(SearchRouteData)
  case listDetail(list: WishList)
  case orderDetails(orderID: String, orderToken: String?)
  case cart(showCheckoutOnAppear: Bool = false)
  case inAppBrowser(url: URL, webViewType: WebViewType = .normal)
  // …30 more
}
```

A route is **data, not a view**: it can be compared, logged, asserted in a test, and built from a URL by code that knows nothing about SwiftUI. When a destination needs a lot of parameters, they go into a value type (`SearchRouteData`) with intent-revealing factories like `.keyword("sofa")` and `.url(...)`, so call sites stay one line.

### 2. `AppRouter`: per-tab state, one owner

```swift
@MainActor
final class AppRouter: ObservableObject {
  /// One stack per tab, so navigating in Lists never disturbs Home.
  @Published var paths: [TabType: NavigationPath] = [:]

  /// NavigationPath is opaque. This mirror is what lets deep links ask what's on screen.
  private(set) var routes: [TabType: [Route]] = [:]

  func push(_ route: Route, on tab: TabType) {
    paths[tab, default: NavigationPath()].append(route)
    routes[tab, default: []].append(route)
  }

  func topRoute(for tab: TabType) -> Route? { routes[tab]?.last }

  func path(for tab: TabType) -> Binding<NavigationPath> {
    Binding(
      get: { self.paths[tab, default: NavigationPath()] },
      set: { newPath in
        self.paths[tab] = newPath
        // Back button and swipe-back write here directly, never calling pop().
        self.trimRoutes(for: tab)
      }
    )
  }
  // pop(from:animated:completion:), popToRoot(for:…), replaceLast(with:on:) …
}
```

### 3. `TabNavigationStack`: the only way a tab gets a stack

```swift
struct TabNavigationStack<Content: View>: View {
  let tab: TabType
  @EnvironmentObject private var router: AppRouter
  @ViewBuilder let content: () -> Content

  var body: some View {
    NavigationStack(path: router.path(for: tab)) {
      content()
        .navigationDestination(for: Route.self) { RouteResolver(route: $0) }
    }
    .environment(\.currentTab, tab)
  }
}
```

Each tab root is now `TabNavigationStack(tab: .lists) { ListsView(…) }`. Teammates never write a bare `NavigationStack`, and destinations never need to know which tab they're in, because it's in the environment.

### 4. `RouteResolver` + destination wrappers: route → screen, with dependencies

```swift
struct RouteResolver: View {
  let route: Route

  var body: some View {
    // Exhaustive: add a Route case and the compiler shows you where to handle it.
    switch route {
    case .listDetail(let list):           ListDetailDestination(list: list)
    case .search(let data):               SearchDestination(data: data)
    case .productDetail(let id, let opt): ProductDetailDestination(productID: id, optionID: opt)
    // …
    }
  }
}
```

The resolver is deliberately thin. The real work is in the **destination wrappers**, one per route, and they earned their place twice (see decisions 3 and 4 below).

## Key decisions and tradeoffs

<div class="decision" markdown="1">

### 1. One typed `Route` enum, not `NavigationLink(destination:)`

**Chose:** every push goes through `router.push(.someRoute, on: tab)`. Inline `NavigationLink { SomeView() }` is not used for app-level navigation.

**Why:** deep links, push notifications and the search bar all need to navigate *from outside the view tree*. View-built destinations can't be triggered from there, can't be inspected, and can't be tested. A value-type route can be all three.

**Cost:** a central enum that every feature touches, so merge conflicts land in one file. Associated values must be `Hashable`, which pushed a few models into conformances they didn't have before. I accepted that: a conflict in an enum is a five-second fix, and a navigation bug you can't reproduce is not.

</div>

<div class="decision" markdown="1">

### 2. A readable mirror alongside an opaque `NavigationPath`

**Problem:** `NavigationPath` is type-erased, so you can append to it and count it but can't read what's in it. The old deep-link code's most important question ("is this product already on top? one screen back?") had no answer.

**Chose:** keep a parallel `[Route]` per tab, updated on every router mutation. The binding setter trims it when SwiftUI pops on its own (back button, swipe-back, long-press back menu).

**Cost:** two sources of truth, which is normally a smell. I contained it by making the mirror `private(set)`, letting it *trail* the path but never *lead* it, and treating it as best-effort when a few legacy `navigationDestination(isPresented:)` sites add entries it doesn't know about. Those rules are pinned by tests: desync must never trap, and must never overwrite a route the mirror does track.

<p class="code-label after">After: the same deep-link logic, as pattern matching on data</p>

```swift
static func goToProductPage(id: String?, optionID: String? = nil) {
  guard let productID = id, let tab = tabs.selectedTabType else { return }

  // Already on top? No casts, no view hierarchy.
  if case .productDetail(let topID, let topOptionID) = router.topRoute(for: tab),
     topID == productID, optionID == nil || optionID == topOptionID {
    return
  }
  // One screen back? Pop to it instead of stacking a duplicate.
  if let stack = router.routes[tab], stack.count >= 2,
     case .productDetail(let previousID, _) = stack[stack.count - 2], previousID == productID {
    router.pop(from: tab)
    return
  }

  router.push(.productDetail(productID: productID, optionID: optionID), on: tab)
}
```

</div>

<div class="decision" markdown="1">

### 3. Destination wrappers own object lifetimes

**Problem found in QA:** search results and category pages randomly reset their scroll position and filters. The cause: one `AppRouter` publishes on *every* push or pop in *any* tab, which re-evaluates every `navigationDestination` closure. A controller created inline there and passed to an `@ObservedObject` gets rebuilt each time, because `@ObservedObject` doesn't retain.

**Chose:** every route that needs a controller gets a small wrapper that holds it in `@StateObject`, so SwiftUI owns the lifetime and it survives re-renders:

```swift
/// Holds the controller in @StateObject so router-driven re-renders don't rebuild it.
struct SearchDestination: View {
  @EnvironmentObject private var appContext: AppContext
  let data: SearchRouteData

  var body: some View { Body(data: data, appContext: appContext) }

  private struct Body: View {
    @StateObject private var controller: SearchResultsController

    init(data: SearchRouteData, appContext: AppContext) {
      _controller = StateObject(wrappedValue: SearchResultsController(
        type: data.type,
        networkController: appContext.networkController
      ))
    }

    var body: some View { SearchResultsView(controller: controller) }
  }
}
```

**Cost:** about 25 small boilerplate types. The upside I didn't fully plan for: this became the **composition root**. Each wrapper is the single place where a screen receives its dependencies, which let me run [a follow-up](/case-studies/removing-app-singleton/) that removed `AppContext.shared` from feature code (122 → 3 reads) by threading explicit dependencies from the wrapper down, without touching navigation again.

</div>

<div class="decision" markdown="1">

### 4. Bridge UIKit screens, don't rewrite them

**Chose:** legacy-heavy destinations (product detail, cart, package tracking, daily deals) are routed through `UIViewControllerRepresentable` wrappers. The tab bar stays a `UITabBarController` whose tabs host `TabNavigationStack` roots instead of navigation controllers.

**Why:** the goal was a single navigation model, not a SwiftUI rewrite. Bundling "rewrite the PDP" into "change navigation" would have produced an unreviewable PR and an unbisectable regression.

**Cost:** a hybrid that newcomers have to understand: a UIKit tab bar, SwiftUI stacks, and some UIKit leaves. Each of those leaves can now be rewritten on its own later, behind the same `Route` case, without any caller changing.

</div>

<div class="decision" markdown="1">

### 5. I added a feature flag, then removed it three weeks in

**Started with:** a remote-config flag (`navigationUsesSwiftUI`) so the new stack could be rolled out per brand and rolled back without a release.

**Learned:** navigation isn't a feature you can flag call by call. A tab is either a `UINavigationController` or a `NavigationStack`; it can't be half of each. So the flag forked tab construction and *every* push site into two code paths, and both had to keep working under constant merges from `master`. Removing it deleted 572 lines in one commit.

**Chose instead:** a long-lived branch kept current with `master` (merged in 30+ times), teammates migrating their own feature areas onto it in parallel, and the full per-brand UI regression run against branch builds before merge. Rollback meant reverting a release, not flipping a flag. That's a real loss of safety, and I took it deliberately: a migration that never finishes because of dual paths is the bigger risk.

</div>

<div class="decision" markdown="1">

### 6. Navigation completions are deterministic, not tied to view lifecycle

**Problem:** the first version fired `pop` completions from the popped view's `onDisappear`. That broke two ways: `popToRoot` on a tab that *isn't on screen* (common in deep links: switch tab, reset it, push) has no view to disappear, so the completion was stranded; and chained deep links raced an animation still in flight.

**Chose:** completions are delivered on the next main-queue turn after the path mutation, and deep-link entry points pass `animated: false` (via a `Transaction` with `disablesAnimations`) when more navigation follows immediately. Behavior is the same whether or not the tab is visible, and it's unit-testable.

</div>

## Applying the pattern: one feature

Each feature area went through the same three steps: wrap the tab root in `TabNavigationStack`, add `Route` cases plus destination wrappers, and replace the pushes. Here is Lists:

<p class="code-label before">Before</p>

```swift
Button("Find a List") {
  // A SwiftUI view reaching out to the deep-link manager to find UIKit's navigation controller
  DeepLinkManager.tabBarTopViewController?.navigationController?
    .navigate(to: FindListViewHostingController(appContext: appContext))
}
Button("Create List") {
  DeepLinkManager.tabBarTopViewController?.navigationController?
    .navigate(to: ListCreateAndEditViewHostingController(list: $createdList, navigationTitle: "Create List"))
}
// The result came back through a binding threaded into a UIKit controller
.onReceive(Just(createdList)) { … }
```

<p class="code-label after">After</p>

```swift
var body: some View {
  TabNavigationStack(tab: .lists) { content }
}

Button("Find a List")   { router.push(.findList, on: .lists) }
Button("Create List")   { router.push(.createEditList(nil, title: "Create List"), on: .lists) }

// The result comes back through shared state; the destination pops itself via the router
.onChange(of: listsController.lastCreatedList) { _, newList in handleListCreated(newList) }
```

Three hosting controllers for this one tab were deleted. Repeated across tabs, the app went from 34 hosting controllers to 24, and the custom swipe-back `UINavigationController` subclass, with its hand-forwarded delegate methods, went away entirely.

## Verification

Navigation bugs are rarely crashes. Usually it's the wrong screen, a duplicate screen, or no screen, which tests catch more reliably than code review. The router got 44 focused unit tests: tab isolation, completion timing, mirror/path sync, and the system-initiated pop that bypasses the API:

```swift
// SwiftUI writes to the path binding on swipe-back and never calls pop().
// The mirror must follow, or the next deep link makes decisions on stale data.
func testSwipeBackViaBindingTrimsRoutes() {
  router.push(.notifications, on: .home)
  router.push(.dailyDeals, on: .home)

  var binding = router.path(for: .home)
  var swiped = binding.wrappedValue
  swiped.removeLast()
  binding.wrappedValue = swiped

  XCTAssertEqual(router.routes[.home]?.count, 1)
  XCTAssertEqual(router.topRoute(for: .home), .notifications)
}
```

Above that, every PR on the branch ran the unit suite and per-brand smoke UI tests on BrowserStack, and the full regression ran against branch builds before merge. Several fixes late in the branch were about keeping UI-test queries stable (back-button labels, search-field focus) rather than app behavior, so the regression suite stayed trustworthy through the change instead of being mass-updated to pass.

## Results

- **One way to navigate.** `router.push(route, on: tab)` is the API for buttons, deep links, push notifications and the search bar alike. Direct UIKit push call sites went from 39 to 8, and the remaining ones are presentation edge cases.
- **Deep links became readable and testable.** Duplicate-screen logic is pattern matching on values, not down-casting controllers.
- **No `NavigationView` left** (13 → 0), and the custom swipe-back navigation controller was deleted.
- **It unlocked dependency injection.** The destination layer became the composition root that the [`AppContext.shared` removal](/case-studies/removing-app-singleton/) (122 → 3) built on.
- **Built for a team.** Teammates migrated Home, Search, PDP sub-flows and Lists onto the architecture in parallel. After the initial walkthrough, the pattern (route case, wrapper, push) needed no further explanation.

## What I'd do differently

- **`@Observable` instead of `ObservableObject`.** With iOS 17 as the minimum, an `@Observable` router would invalidate only views that read the path that changed. That fixes the re-render problem at the source instead of working around it in wrappers. I kept `ObservableObject` to match the rest of the codebase mid-migration; it's the first follow-up I'd do.
- **Split `Route` by domain.** 36 cases in one enum is fine; 80 won't be. Nested enums (`.account(.orders)`, `.lists(.detail(list))`) would keep ownership local without giving up exhaustiveness.
- **Retire the remaining `NotificationCenter` hops.** A few flows, such as re-triggering "write a review" on a PDP that's already open, still post notifications instead of carrying intent in the route. They work, but they're the last untyped navigation in the app.
- **Make the deep-link manager an instance.** It's still a static type configured with an `AppContext` at launch. That's better than reading a singleton, but an injected instance would let URL routing be tested end to end without global setup.

<p class="muted"><em>Thanks to my teammates, who migrated their feature areas onto the router and caught a good share of the edge cases above in review.</em></p>
