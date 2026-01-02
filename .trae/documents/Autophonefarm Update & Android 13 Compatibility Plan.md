# Project Update and Android 13 Compatibility Plan

## Scope and Goals
- Upgrade Autophonefarm/STF to run reliably on modern runtimes (Node.js 18+ and modern OS images).
- Extend device compatibility from the documented baseline (Android 2.3.3 / SDK 10 → Android 9 / SDK 28) to include Android 13 / SDK 33, while keeping backward compatibility down to the minimum supported version.
- Preserve core product capabilities (device discovery, remote control, install, logs, screenshots, streaming) with clear degradations where platform restrictions require it.

## 1. Code Improvement Strategy
### 1.1 Full Code Review (1–2 weeks, parallelizable)
- Create an inventory of critical paths by feature:
  - Device session lifecycle and ownership
  - Screen streaming pipeline (minicap-based)
  - Touch/input injection pipeline (minitouch-based)
  - Install/uninstall/launch flows
  - Logcat streaming, file operations, port forwarding
  - Authentication and API endpoints
- Identify and rank issues:
  - Reliability: unhandled promise rejections, resource leaks, timeouts, retry loops
  - Security: dependency CVEs, request validation, secrets handling, auth flows
  - Performance: high-CPU loops, unbounded buffers, backpressure issues
  - Maintainability: duplicated logic, unclear module boundaries, legacy patterns
- Produce a prioritized refactor backlog with explicit acceptance criteria per item.

### 1.2 Refactor Inefficient Algorithms and Improve Structure
- Standardize async patterns:
  - Reduce Bluebird-specific control-flow in new/edited code; favor native Promises/async flows.
  - Consolidate retry/backoff patterns into shared utilities to avoid inconsistent behavior.
- Stabilize device I/O boundaries:
  - Centralize ADB command execution patterns (timeouts, retries, error classification).
  - Ensure every long-lived stream/socket has explicit lifecycle management and cleanup.
- Simplify feature toggles by SDK level:
  - Single “capabilities” layer derived from SDK and device properties (screen, input, etc.).
  - Avoid scattering `sdk.level >= 30` checks across many modules.

### 1.3 Modern Coding Standards and Best Practices
- Enforce consistent linting on all changed code via existing `gulp lint`.
- Define and enforce conventions:
  - Error handling: always include actionable error messages and preserve root causes
  - Logging: structured, rate-limited for high-frequency paths (screen/touch)
  - Timeouts: explicit defaults, overridable via config, consistent units (ms)
- Reduce “legacy surface area”:
  - Prefer npm-managed frontend dependencies over Bower where feasible.
  - Remove dead code and unused dependencies identified during audit.

### 1.4 Add Comprehensive Unit Tests (≥80% coverage)
- Establish coverage scope:
  - Unit tests for core library modules (`lib/**`) and critical device plugins.
  - Frontend unit tests under existing Karma/Jasmine setup (`res/test/karma.conf.js`).
  - Integration tests for ADB-level flows via emulator/device lab (smoke-level minimum).
- Coverage gates:
  - Add coverage reporting and fail CI if overall coverage < 80%.
  - Track per-module coverage for highest-risk areas (screen/touch/install/auth).
- Regression test targets (minimum set):
  - Device discovery and status transitions
  - Screen: start/stop stream, rotation changes, screenshot capture
  - Touch: tap, swipe, text input, coordinate mapping across rotations
  - Install: upload APK, install, launch main activity, uninstall

## 2. Dependency Management
### 2.1 Audit All Current Dependencies
- JavaScript dependencies:
  - `package.json` runtime and dev dependencies
  - `bower.json` and any vendored/legacy frontend assets
- System dependencies (documented in `package.json` and README):
  - ADB, RethinkDB, ZeroMQ, Protocol Buffers, GraphicsMagick, build toolchain
- Runtime consistency:
  - Align repository runtime markers (`package.json` engines, `.tool-versions`, Docker images, CI images).

### 2.2 Update to Latest Stable Versions
- Upgrade strategy:
  - Phase 1: “safe” upgrades (patch/minor) to reduce security exposure.
  - Phase 2: major upgrades with targeted migration work (Express, Socket.IO, build chain).
  - Phase 3: replace obsolete tooling where upgrades are not practical (e.g., legacy bower/webpack 1 pipelines).
- Key libraries to prioritize for Android 13 work:
  - `adbkit` and any ADB-related dependencies for compatibility with newer platform behaviors.
  - `minicap-prebuilt-beta` and `minitouch-prebuilt-beta` (or replacements) for Android 11+ constraints.

### 2.3 Verify Backward Compatibility (Android 13 and Lower)
- Define “must work” behavior by SDK bucket:
  - **SDK < 30**: keep minicap/minitouch fast path for streaming and multitouch.
  - **SDK ≥ 30**: provide reliable fallbacks for screen viewing and input, even if performance is reduced.
- Maintain feature parity where possible; otherwise document degradations and acceptable limits.

### 2.4 Document Version Changes and Migration Steps
- Track every dependency upgrade with:
  - Previous version → new version
  - Reason (security, compatibility, bugfix)
  - Required code changes
  - Verification performed (tests + device matrix)
- Maintain version history in `CHANGELOG.md` and include developer-facing “migration notes” in this document (Section 6).

## 3. Compatibility Testing
### 3.1 Android Version Test Matrix (Android 13 → minimum supported)
Minimum supported version is currently documented as Android 2.3.3 (SDK 10).

| Android | SDK | Device Type | Must-Pass Test Focus |
|---|---:|---|---|
| 13 | 33 | Physical | Stream fallback, input fallback, install, logs |
| 12L | 32 | Physical (tablet/foldable) | Screen sizing, rotation, UI layout |
| 12 | 31 | Physical | ADB behaviors, install/launch, streaming stability |
| 11 | 30 | Physical | First “fallback required” baseline (minicap/minitouch constraints) |
| 10 | 29 | Physical | Scoped storage edge cases, install, logs |
| 9 | 28 | Physical | Legacy baseline regression (previously supported) |
| 8.1 | 27 | Emulator + physical if available | Legacy rendering and input behavior |
| 8.0 | 26 | Emulator + physical if available | Legacy rendering and input behavior |
| 7.1 | 25 | Emulator | Session stability and UI compatibility |
| 7.0 | 24 | Emulator | Session stability and UI compatibility |
| 6.0 | 23 | Emulator | Permissions-era compatibility checks |
| 5.1 | 22 | Emulator | ABI detection, device service interaction |
| 5.0 | 21 | Emulator | ABI list handling and older service behavior |
| 4.4 | 19 | Emulator | Pre-ART edge cases, basic device operations |
| 4.1–4.3 | 16–18 | Emulator | Basic device operations and UI rendering |
| 2.3.3 | 10 | Emulator (best-effort) | Smoke: discovery + simple shell/install if possible |

### 3.2 Physical Device Coverage (OEM Matrix)
Test at minimum on the following OEM families (prefer 2 models per family where feasible):
- Google Pixel (AOSP-like baseline)
- Samsung (One UI)
- Xiaomi/Redmi (MIUI/HyperOS)
- OnePlus/Oppo/Realme (ColorOS/OxygenOS variants)
- Motorola or Vivo (additional vendor variance)

### 3.3 Screen Size and Resolution Coverage
- Phone (small): ~720p
- Phone (large): 1080p / high aspect ratio
- Tablet: 10–12"
- Foldable: dynamic resolution/rotation behaviors
- Validate:
  - Rotation and touch coordinate mapping
  - Stream resizing and UI fit in the web client
  - Screenshot correctness (orientation, cropping, secure screens)

### 3.4 Regression Testing for Previously Working Features
- Maintain a “golden” regression suite that runs at least on SDK 28 and SDK 33:
  - Device list and filtering
  - Remote control: screen on/off, rotation handling
  - Touch/tap/swipe and text input
  - APK install/launch/uninstall
  - Logcat streaming and filtering
  - File explorer and port forwarding (where supported)

## 4. Implementation Plan (Weekly Sprints)
### Sprint 1: Baseline Review + Build Stabilization
Deliverables:
- Code review findings and prioritized backlog (Section 1.1 output)
- Reproducible dev environment baseline (Node 18+ aligned across docs/tooling)
- CI baseline: lint + unit test execution on every PR
Checkpoints:
- Code review checkpoint: top 10 risks approved for remediation
- QA checkpoint: “clean build” and “test” run documented

### Sprint 2: Dependency Audit + Low-Risk Upgrades
Deliverables:
- Dependency inventory (npm, bower, system deps) and upgrade plan
- Patch/minor upgrades applied with smoke validation
- Documented deltas and rollback plan for dependency changes
Checkpoints:
- Security checkpoint: known critical CVEs addressed or mitigated

### Sprint 3: Android 11+ Screen Strategy (Android 13 Compatibility Core)
Deliverables:
- Define and implement the screen capability split:
  - SDK < 30: keep minicap streaming
  - SDK ≥ 30: streaming fallback strategy (e.g., `screenrecord`/H.264 pipeline or scrcpy-based approach)
- Ensure screenshots remain reliable (minicap fast path + `adb screencap` fallback)
Checkpoints:
- QA checkpoint: SDK 30/33 smoke on at least 2 OEM devices

### Sprint 4: Android 11+ Input Strategy
Deliverables:
- Complete the touch fallback for SDK ≥ 30 using `adb shell input` (tap/swipe/text) with correct coordinate transforms.
- Document multitouch degradation behavior for fallback mode (if applicable).
Checkpoints:
- QA checkpoint: input reliability tests on SDK 30 and SDK 33

### Sprint 5: Test Coverage + Regression Automation (≥80%)
Deliverables:
- Add/extend unit tests to meet ≥80% coverage target
- Add integration smoke tests that run against an emulator/device in CI (where feasible)
- Create a stable regression checklist and automate the highest-value subset
Checkpoints:
- Quality gate: coverage threshold enforced in CI

### Sprint 6: Full Compatibility Matrix Execution + Bug Fixing
Deliverables:
- Run test matrix (Section 3.1) and record results
- Fix compatibility bugs and document known limitations
- Performance profiling and targeted optimizations in hot paths
Checkpoints:
- Release readiness review: no open critical issues for SDK 33 and SDK 28

### Sprint 7: CI/CD + Rollout Controls
Deliverables:
- Automated build pipeline for artifacts (npm package and/or Docker images)
- Automated test stages (lint, unit, integration smoke, e2e where applicable)
- Canary deployment workflow and monitoring/alerting plan
Rollback procedures:
- Versioned artifacts (tagged Docker images and/or npm versions) for quick rollback
- Feature-flagged fallbacks where possible (disable new pipeline without reverting whole release)

### Sprint 8: Production Release + Post-Release Validation
Deliverables:
- Production deployment with staged rollout
- Post-release monitoring report (errors, performance, device compatibility)
- Final documentation updates and changelog entry

## 5. Success Criteria
- Compatibility:
  - All existing core features work on Android 13 (SDK 33) and remain functional down to the minimum supported version (SDK 10), with documented degradations where unavoidable.
- Performance (benchmarked and tracked):
  - Screen streaming on SDK < 30 maintains baseline FPS/latency (minicap path).
  - SDK ≥ 30 fallback achieves agreed minimum usability thresholds (e.g., stable stream, acceptable latency).
- Reliability:
  - Measurable reduction in crash/fatal error rate in logs (baseline vs. release).
  - No regressions in the “golden” regression suite on SDK 28 and SDK 33.
- Deployment:
  - Successful production rollout with documented rollback executed successfully in staging drills.

## 6. Documentation Deliverables (Included in This Program)
### 6.1 Technical Documentation Updates
- Update README:
  - Supported Android versions (extend to Android 13 and document fallbacks)
  - Node.js requirement (align with current `package.json` engines and actual support stance)
- Update deployment documentation (`doc/DEPLOYMENT.md`) where changes affect ops:
  - Runtime requirements, system deps, container images, CI/CD usage
- Update testing documentation (`TESTING.md`) with:
  - How to run unit tests, e2e tests, and coverage locally
  - How to run the Android matrix tests and where results are recorded

### 6.2 Developer Migration Guide (for This Update)
- Environment migration:
  - Standardize on Node.js 18+ across local dev, CI, and containers.
  - Define how to update toolchain files (`.tool-versions`, container base images) to match.
- Dependency migration:
  - List breaking changes from major dependency upgrades and required code edits.
  - Provide step-by-step rollout order to keep the system deployable at all times.
- Android 13 migration:
  - Describe the capability split (SDK < 30 vs SDK ≥ 30).
  - Document fallback behaviors and any limitations (multitouch, FPS, secure screens).

### 6.3 Changelog and Known-Issues Tracking
- Maintain `CHANGELOG.md` with:
  - Compatibility notes (Android 13 support, fallbacks)
  - Dependency upgrade highlights and breaking changes
- Maintain a “known issues” section for:
  - OEM-specific quirks
  - Secure screen limitations
  - Any features intentionally degraded on SDK ≥ 30 (and planned improvements)
