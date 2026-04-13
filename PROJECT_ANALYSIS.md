# Project Analysis: Blixt Wallet

## 1) What this project is

Blixt Wallet is a React Native TypeScript Lightning wallet with Android, iOS, and web targets. The README explicitly positions web as a prototyping target and native apps as the real wallet deployments.

## 2) Technology stack

- **UI/runtime:** React 17 + React Native 0.64 + TypeScript.
- **State management:** easy-peasy with typed hooks.
- **Navigation:** React Navigation stack/bottom/material tabs.
- **Persistence:** AsyncStorage (app settings/state), Keychain (secrets), SQLite (transactional data).
- **Lightning backend:** Embedded `lnd` via native module bindings (`lndmobile`), with optional fake backend flavor for local/demo use.
- **Web build:** webpack + react-native-web with a number of platform replacement hacks.
- **Tests:** Jest with a mix of state-model and React snapshot/UI tests.

## 3) High-level architecture

The app architecture is centered around a large global store model that composes many feature sub-models (`lightning`, `send`, `receive`, `onChain`, backups, notifications, etc.).

Startup flow:

1. `App` mounts providers (easy-peasy store, native-base, navigation) and, on web, clears app storage for clean demo behavior.
2. `Main` decides which flow to show (`init`, `authentication`, `started`) based on app/store state.
3. `initializeApp` (store thunk) performs app bootstrap:
   - open database,
   - first-run setup/migrations/config,
   - initialize native LND module,
   - start lnd process if needed,
   - initialize feature stores,
   - subscribe to wallet state events.
4. Lightning store initialization then asynchronously does chain/graph sync, autopilot setup, and dependent store initialization.

This design provides a clear boot pipeline, but it concentrates complexity in startup thunks.

## 4) Strengths

- **Cross-platform approach is practical:** one codebase across Android/iOS/web with platform-selective fallbacks.
- **Feature-rich wallet surface:** LNURL, WebLN browser, backups, Tor, autopilot, on-chain and LN flows.
- **Reasonable data separation:**
  - AsyncStorage for app config,
  - Keychain for secrets,
  - SQLite for structured transactional history.
- **Store modularization:** distinct state models for major wallet domains.
- **Test scaffolding exists:** dedicated tests for easy-peasy models and UI windows.

## 5) Risks / maintainability concerns

1. **Aging dependency baseline**
   - React Native 0.64-era dependencies and legacy packages can increase maintenance/security risk and make platform upgrades painful.

2. **Large startup orchestration in store thunks**
   - `initializeApp` and `lightning.initialize` carry many responsibilities (platform migration, tor startup, lnd lifecycle, store bootstrap, event wiring), creating a high blast radius for regressions.

3. **Potential security hardening opportunities**
   - Some keychain writes use `ACCESSIBLE.ALWAYS`; this may be acceptable for usability, but should be reviewed against threat model and platform best practices.

4. **Web target complexity**
   - Many webpack normal-module replacements and polyfills indicate significant platform divergence. This can slow dependency upgrades and increase hidden breakage.

5. **Store shape centralization**
   - The root model aggregates many domains directly; this is workable, but scaling team contributions and enforcing boundaries can become difficult over time.

## 6) Suggested technical roadmap (incremental)

1. **Stabilization pass (short term)**
   - Add explicit startup-state telemetry/logging milestones.
   - Increase test coverage on bootstrap/error paths (lnd already running, tor startup failure, migration failure).
   - Document critical startup sequence and recovery behavior.

2. **Architecture hardening (short-to-mid term)**
   - Extract app bootstrap orchestration into dedicated service modules (e.g., `BootstrapService`, `LndLifecycleService`) and keep thunks thinner.
   - Move side-effect-heavy async flows into testable helpers with clear input/output contracts.

3. **Security review (mid term)**
   - Re-evaluate keychain accessibility class for wallet password and sensitive values per platform.
   - Audit persistence defaults and backup inclusion/exclusion behavior.

4. **Dependency modernization plan (mid-to-long term)**
   - Plan staged upgrades: TypeScript toolchain, React/React Native major versions, navigation stack.
   - Reduce/customize web hacks where possible and isolate them behind a compatibility layer.

## 7) Practical contributor guidance

For a new contributor, highest-leverage starting points are:

- Read startup path in `src/App.tsx`, `src/Main.tsx`, `src/state/index.ts`, and `src/state/Lightning.ts`.
- Understand persistence interfaces in `src/storage/app.ts`, `src/storage/keystore.ts`, and `src/storage/database/*`.
- Run and inspect existing Jest tests before modifying state models.

---

If you want, I can also produce a **module dependency map** (state slices -> storage -> lndmobile methods -> windows) and a **prioritized technical debt backlog** with estimated effort/risk.
