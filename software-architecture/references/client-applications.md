# Client Application Architecture

Architecture for apps that run on the user's device or in their browser: web front-ends (SPA/SSR), mobile (native or cross-platform), and desktop. The same principles as the backend apply — separate UI from logic, depend on abstractions at boundaries, keep business rules testable — but the constraints differ: unreliable networks, lifecycle events, limited resources, and untrusted runtime.

## Table of Contents

1. [Layering](#layering)
2. [UI Patterns](#ui-patterns)
3. [State Management](#state-management)
4. [Data and Caching](#data-and-caching)
5. [Offline-First and Sync](#offline-first-and-sync)
6. [Modularisation](#modularisation)
7. [Security on the Client](#security-on-the-client)
8. [Testing Client Apps](#testing-client-apps)
9. [Performance and Observability](#performance-and-observability)

---

## Layering

```
┌──────────────────────────────────────────────┐
│ UI (views, components, screens)              │  ← renders state, forwards user intents
├──────────────────────────────────────────────┤
│ Presentation (ViewModel / Presenter / Store) │  ← UI state, maps intents to use cases
├──────────────────────────────────────────────┤
│ Domain (use cases, models)  — optional       │  ← business rules shared across screens
├──────────────────────────────────────────────┤
│ Data (repositories → remote API, local DB)   │  ← single source of truth, caching, sync
└──────────────────────────────────────────────┘
```

- **UI is a function of state**: views render state and emit events; they don't fetch data or hold business rules.
- **Repositories** hide whether data comes from network, cache, or local DB.
- A **domain layer** is optional — add it when logic is reused across screens or is complex enough to deserve isolated tests. Many apps go straight from presentation to repositories.
- The **server stays authoritative** for anything security- or money-related; client-side validation is for UX only.

---

## UI Patterns

| Pattern | Where you'll see it | Notes |
|---|---|---|
| **Component + hooks/composables + store** | React, Vue, Svelte, Solid, Angular (signals) | Keep data fetching and logic in hooks/composables/services, not components |
| **MVVM** | Android (Jetpack Compose + ViewModel), SwiftUI (`@Observable`), .NET MAUI/WPF/WinUI, Flutter with Riverpod/Provider | ViewModel exposes observable UI state; survives configuration changes |
| **MVI / Unidirectional data flow** | Redux/Zustand/NgRx/Pinia, Compose with reducers, The Composable Architecture (Swift), Bloc (Flutter), Elm | `state = reduce(state, intent)`; effects run outside the reducer; very testable |
| **MVP / Coordinator** | Legacy UIKit/Android, some desktop apps | Coordinator/router handles navigation outside screens |

```
// Unidirectional flow, framework-neutral
record ProfileState { loading: bool, user: UserView?, error: string? }
enum ProfileIntent { Load(userId), Retry, EditName(name) }

class ProfileViewModel(users: UserRepository) {
  state: Observable<ProfileState> = ProfileState(loading = false, user = none, error = none)

  on(intent) {
    match intent {
      Load(id) -> {
        state.update(s -> s.copy(loading = true, error = none))
        launch {                                           // coroutine / task / promise, cancelled with the screen
          match users.get(id) {
            ok(user)   -> state.update(s -> s.copy(loading = false, user = UserView.from(user)))
            err(error) -> state.update(s -> s.copy(loading = false, error = messageFor(error)))
          }
        }
      }
      ...
    }
  }
}
```

---

## State Management

Classify state before choosing a tool:

| Kind | Examples | Where it lives |
|---|---|---|
| **Server state** (cached copy of remote data) | user profile, order list | A data-fetching cache: TanStack Query, SWR, Apollo/urql, RTK Query; repositories with local DB on mobile |
| **UI state** (ephemeral) | open modal, input text, scroll position | Component/ViewModel local state |
| **App/session state** (global, client-owned) | auth session, theme, feature flags, cart before checkout | A small global store (Zustand, Pinia, NgRx Signals, Redux Toolkit, Riverpod, a DI singleton) |
| **URL state** | filters, selected tab, pagination | The router/query string — shareable and survives reload |

Most "state management" complexity comes from storing server state in a global store by hand. Use a server-state cache for that and keep the global store small.

---

## Data and Caching

```
class UserRepository(api: UserApi, db: LocalUserStore, clock) {
  // Stale-while-revalidate: emit cached data immediately, refresh in background
  observe(id) -> Stream<User> {
    launch { refreshIfStale(id) }
    return db.observe(id)                      // the local DB is the single source of truth for the UI
  }

  refreshIfStale(id) {
    cached = db.get(id)
    if cached is present and clock.now() - cached.fetchedAt < 5 minutes: return
    match api.getUser(id) {
      ok(dto)  -> db.upsert(User.from(dto), fetchedAt = clock.now())
      err(e)   -> log.warn("refresh failed", e)   // UI keeps showing cached data
    }
  }
}
```

- **Typed API client** generated from the backend contract (OpenAPI/GraphQL/Protobuf codegen) so client and server stay in sync.
- Centralise **auth headers, retries (idempotent requests only), timeouts, and error mapping** in one HTTP client layer (interceptors).
- **Request deduplication and cancellation** when screens unmount/navigate away.
- **Pagination**: infinite lists use cursor-based APIs.
- **Optimistic updates** for snappy UX: update local state immediately, roll back if the server rejects.

---

## Offline-First and Sync

For mobile and field apps where connectivity is unreliable:

- **Local database** as source of truth (SQLite via Room/SQLDelight/GRDB/Core Data/SwiftData/Drift/sqflite, IndexedDB on web, Realm/Couchbase Lite, or sync engines like PowerSync/ElectricSQL/Replicache).
- **Outbox of pending mutations** on the device, replayed when online — each with a client-generated ID used as an **idempotency key** on the server.
- **Conflict strategy** decided per entity: last-write-wins (simple, lossy), server-wins, field-level merge, or CRDTs for collaborative data.
- Show sync status honestly (pending, failed, synced) and let users retry.
- Schedule background sync with platform APIs (WorkManager, BGTaskScheduler, Background Sync / Service Workers) that respect battery and network constraints.

---

## Modularisation

- **Feature modules** (Gradle modules, Swift packages, Dart packages, monorepo packages) with a small public API, depending on shared `core` modules (networking, design system, analytics) — never on each other's internals.
- **Design system** module: tokens (color, spacing, typography), reusable components, accessibility baked in.
- **Navigation** defined at the app level; features expose entry points/routes rather than knowing about each other's screens.
- For large web apps, consider micro-frontends only when independent team deployment truly outweighs the integration cost.

---

## Security on the Client

The client is **untrusted**: anything in the bundle or binary can be read and modified.

- **No secrets in client code** (API keys with privileged access, signing keys). Proxy privileged calls through your backend.
- **Auth**: OAuth 2.x Authorization Code + PKCE; on web prefer a BFF with HttpOnly cookies over tokens in `localStorage`; on mobile/desktop store tokens in the OS keystore. See [security.md](security.md#tokens-and-sessions).
- **XSS**: rely on framework escaping; avoid raw-HTML APIs; set a strict CSP.
- **Deep links / intents / custom URL schemes**: validate parameters; prefer verified App Links / Universal Links.
- **Transport**: HTTPS only; consider certificate pinning only if you can operate key rotation safely.
- **Sensitive data at rest**: encrypt local stores that hold personal data; clear on logout.
- **WebViews / Electron / Tauri**: disable Node integration in renderers, enable context isolation, restrict IPC surface and allowed origins.

---

## Testing Client Apps

| Layer | Tooling examples | What to test |
|---|---|---|
| ViewModel / store / reducer | Plain unit tests (JUnit/Kotest + Turbine, XCTest/Swift Testing, Vitest/Jest, `flutter_test`, bloc_test) | State transitions for each intent, with fake repositories |
| Repository / data | Unit tests with fake API + in-memory DB; MSW / stub servers | Caching, refresh, offline queue, error mapping |
| Components / screens | Testing Library, Compose UI tests, ViewInspector/XCUITest, widget tests | Renders states correctly; accessibility labels; user interactions emit intents |
| Visual regression | Playwright/Chromatic/Percy, Paparazzi/Roborazzi, swift-snapshot-testing, golden tests | Layout regressions across themes and sizes |
| End-to-end | Playwright, Cypress, Maestro, Detox, Espresso, XCUITest, Flutter integration_test | Critical journeys against a test backend |

---

## Performance and Observability

- **Web**: track Core Web Vitals (LCP, INP, CLS); code-split by route; lazy-load heavy components; optimise images; prefer SSR/SSG/streaming where SEO or first paint matters.
- **Mobile/desktop**: track cold start time, frame drops/jank, ANRs/hangs, memory, battery, app size.
- **Crash and error reporting** (Sentry, Firebase Crashlytics, Bugsnag, OpenTelemetry client SDKs) with release/version tags and source maps/dSYMs/mapping files uploaded in CI.
- **Propagate trace context** from client to backend for end-to-end traces of user actions.
- **Accessibility** is architecture too: semantic components, focus management, dynamic type/scaling, contrast — build it into the design system.
