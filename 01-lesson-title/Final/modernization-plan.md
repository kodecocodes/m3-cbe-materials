# BabyTrack modernization plan

## Purpose

Modernize BabyTrack incrementally while preserving its persisted feed and image data, avoiding unproven rewrites, and keeping each change independently testable. This plan contains no implementation design or converted code.

## Current state

### Active architecture

The app launches into a SwiftUI root view. `SceneDelegate` creates a `UIHostingController` containing `FeedView`; a UIKit navigation-controller startup method exists but is not selected by the active path. [`SceneDelegate.swift`](../Starter/BabyTrack/BabyTrack/Boilerplate/SceneDelegate.swift#L13-L32)

`FeedView` owns a Swift `Feed` model using `@StateObject`, renders the feed list, and presents a sheet for adding a photo moment. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L9-L39)

The Swift `Feed` is a `@MainActor` `ObservableObject` facade. It delegates storage operations to `__Feed`, the Swift-refined name for the Objective-C `Feed` API. [`Feed.h`](../Starter/BabyTrack/BabyKit/Feed.h#L14-L28) [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L10-L40)

The Objective-C implementation persists text event records to `Documents/feed.data` and attachment JPEGs to the Documents directory using UUID filenames. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L47-L74) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L149-L169)

### Coexisting legacy path

A UIKit `ViewController` and table-view `FeedCell` remain in the target. They directly use the Objective-C `Feed` API and maintain their own in-memory item array. [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L11-L17) [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L34-L64)

The UIKit cell reaches Swift display helpers through the generated Swift header, while SwiftUI uses those helpers directly. [`FeedCell.m`](../Starter/BabyTrack/BabyTrack/UIKit/FeedCell.m#L6-L21) [`FeedItem+Ext.swift`](../Starter/BabyTrack/BabyTrack/FeedItem+Ext.swift#L9-L34)

SwiftUI also deliberately bridges UIKit for photo selection using `UIViewControllerRepresentable`, `PHPickerViewController`, and a coordinator. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L10-L30)

### Verified constraints and gaps

- The two UI paths share persisted data but use different presentation-state and picker workflows. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L39-L69) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L97-L146)
- Event and image writes ignore failure results but still return a feed item or attachment UUID. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L83-L94) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L160-L169)
- The active picker delegate does not change its `isPresented` binding for an empty or unsupported result. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L39-L43)
- `FeedView` reads `isBabySleeping` during body construction, and that Objective-C property reloads the feed synchronously from disk. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L13-L25) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L33-L59)
- No test source files matching `*Tests*` or `*UITests*` were visible in the Xcode project structure. This does **not** prove that no test target exists.

## Decisions and planning principles

1. Treat SwiftUI as the active product UI because it is the selected launch path; do not assume that the UIKit path is disposable until its product role is confirmed.
2. Preserve the persisted feed and image contract until compatibility behavior is characterized and validated.
3. Use the existing Swift `Feed` facade as the SwiftUI-facing boundary; do not expose new SwiftUI work directly to the Objective-C implementation without a specific reason.
4. Modernize one boundary at a time. Do not combine storage migration, observation migration, UIKit removal, and UI behavior changes in one effort.
5. Require behavioral validation before deletion or data-format changes.
6. Do not rewrite working Objective-C code solely because it is Objective-C, or working UIKit interoperability solely because the containing screen is SwiftUI.

## Migration approaches

| Approach | What remains stable | Expected benefit | Preconditions | Primary caution |
|---|---|---|---|---|
| Stabilize the current mixed architecture | Objective-C storage, Swift facade, SwiftUI root, UIKit fallback | Lowest risk; preserves compatibility while evidence is gathered | Current behavior characterized | Duplicate UI paths can drift |
| Complete the UI migration vertically | BabyKit persistence and facade | Reduces duplicate presentation code if UIKit is not supported | SwiftUI parity and UIKit support decision | Removing UIKit too early can remove a needed path |
| Harden the facade/data contract | Existing storage implementation initially | Isolates future persistence evolution from UI clients | Tests for successful and failed persistence | Error semantics affect both UI clients |
| Incrementally adopt newer SwiftUI observation | Existing model/storage behavior | Potentially narrower invalidation and less Combine reliance | Confirmed deployment baseline and measured benefit | Observation behavior differs from `ObservableObject` |
| Replace persistence internals | External Swift/UI contracts | Potential validation, durability, and evolvability improvements | Compatibility fixtures and migration policy | Data migration is a high-risk change |

## Phased plan

### Phase 0 — Confirm support and compatibility constraints

**Goal:** Establish the product and platform facts that constrain safe modernization.

**Scope:** Determine deployment targets, target membership, whether UIKit is intentionally shipped/supported, test-target configuration, and the retention/compatibility expectations for `feed.data` and JPEG attachments.

**Dependencies:** None.

**Interoperability:** No source or storage changes. Both clients continue using the current persistence implementation.

**Risks:** Misclassifying inactive UIKit as dead code; adopting APIs unavailable on supported systems; overlooking users’ existing data.

**Validation:** Inspect schemes, targets, build settings, and representative persisted data. Record a decision for UIKit: supported, migration fallback, or non-product sample.

### Phase 1 — Establish a behavior baseline

**Goal:** Describe and protect the behavior at the Objective-C/Swift and UIKit/SwiftUI boundaries before modifying them.

**Scope:** Characterize loading order, every event kind, Sleep/Awake behavior, image storage/display, write failures, malformed records, and picker completion/cancellation paths.

**Dependencies:** Phase 0’s support and data-retention decisions.

**Interoperability:** Confirm that records made by UIKit are readable in SwiftUI and that records made by SwiftUI are readable through the legacy Objective-C route. Both rely on the same storage URLs and item types. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L47-L74) [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L20-L35)

**Risks:** Capturing accidental legacy behavior as a permanent requirement. Classify each observed behavior as intended compatibility, known shortcoming, or undecided product behavior.

**Validation:** Automated coverage where possible, supplemented by simulator/device checks for photo selection, cancellation, relaunch, and attachment rendering.

### Phase 2 — Contain duplicate UI behavior

**Goal:** Prevent further divergence while preserving any still-required legacy UI.

**Scope:** Establish the supported SwiftUI behavior as the reference for the active launch flow. Retain the UIKit UI unchanged until Phase 1 validation and the Phase 0 support decision justify retirement.

**Dependencies:** Phase 1 behavior baseline.

**Interoperability:** Retain `UIViewControllerRepresentable` for `PHPickerViewController`; it is an intentional UIKit-in-SwiftUI boundary, not evidence that the screen requires a rewrite. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L14-L30)

**Risks:** Treating a visual migration as equivalent to behavioral parity; changing picker behavior at the same time as data handling.

**Validation:** Compare event creation, saved records, image attachment behavior, and cancellation/error states across any supported paths.

### Phase 3 — Harden the shared data contract

**Goal:** Make the existing persistence boundary reliable enough for later internal modernization.

**Scope:** Address documented persistence failure behavior, malformed-data handling, attachment consistency, and stable event identity requirements. Keep the public Swift facade as the boundary during this work.

**Dependencies:** Phase 1 coverage and Phase 2 supported-path decision.

**Interoperability:** UIKit continues to use legacy `Feed`; SwiftUI continues to use the Swift facade over `__Feed`. Changes must preserve both clients while both are supported. [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L37-L63) [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L14-L35)

**Risks:** Changing write-failure semantics affects current UI behavior; modifying the text format without compatibility evidence can make existing data unreadable.

**Validation:** Failure-path tests, persisted-data readback checks, legacy-fixture compatibility checks, and verification that an attachment reference never represents a failed image write as a saved image.

### Phase 4 — Selective internal modernization

**Goal:** Modernize only the layers for which Phase 0–3 evidence demonstrates a benefit.

**Scope:** Consider, separately, Observation adoption, persistence replacement, list identity evolution, synchronous-state lookup optimization, and UIKit retirement.

**Dependencies:** Stable data behavior, validated supported UI path, deployment-target decision, and a measured or product-driven reason for each change.

**Interoperability:** Preserve either the Swift `Feed` contract or an explicitly tested successor contract throughout an internal storage change. Observation-based and `ObservableObject` model types may coexist incrementally; a whole-app observation rewrite is not required.

**Risks:** Large coupled rewrites obscure regression causes; format migration without fixtures risks data loss; replacing existing image loading without runtime evidence risks solving an unproven problem.

**Validation:** One modernization concern per change, with persistence compatibility checks, UI regression tests, and performance measurement before accepting performance-focused work.

## Code to retain during the plan

- The `BabyKit` public headers and Objective-C `FeedItem` model, until a compatibility decision is made. [`BabyKit.h`](../Starter/BabyTrack/BabyKit/BabyKit.h#L6-L15) [`FeedItem.h`](../Starter/BabyTrack/BabyKit/FeedItem.h#L19-L33)
- The Swift `Feed` facade, as the current main-actor boundary used by SwiftUI. [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L10-L40)
- The SwiftUI root path in `SceneDelegate`, because it is the active application entry point. [`SceneDelegate.swift`](../Starter/BabyTrack/BabyTrack/Boilerplate/SceneDelegate.swift#L28-L32)
- The `UIViewControllerRepresentable` picker bridge, unless a product or platform requirement proves a different boundary necessary. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L10-L30)
- UIKit `ViewController`, `FeedCell`, storyboard assets, and generated-header interactions until Phase 0 confirms their product role and Phase 1 proves parity where retirement is intended. [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L34-L112) [`FeedCell.m`](../Starter/BabyTrack/BabyTrack/UIKit/FeedCell.m#L18-L39)

## Risks requiring ongoing attention

| Risk | Evidence | Planning response |
|---|---|---|
| In-memory UI can show an item that failed to persist | Write failures are ignored and `addFeedItem:` returns the item. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L83-L94) | Address in Phase 3 after failure behavior is tested. |
| Moment may reference an image that was not written | Image write result is not checked before returning the UUID. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L160-L169) | Address in Phase 3 with attachment-consistency validation. |
| Picker state may remain presented after cancellation/unsupported selection | Early delegate return does not update `isPresented`. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L39-L43) | Validate in Phase 1; contain as a small behavioral correction in Phase 2. |
| UI implementations drift | Each path renders and updates its own item list and picker flow. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L16-L37) [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L47-L83) | Decide support status before deleting or expanding either path. |
| SwiftUI rendering triggers storage reads | `isBabySleeping` reloads the feed while read by `body`. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L16-L25) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L33-L45) | Measure first; consider only in Phase 4 if evidence supports it. |
| List identity is not a dedicated persisted event identity | `List` uses `date`; `FeedItem` has no event ID. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L28-L30) [`FeedItem.h`](../Starter/BabyTrack/BabyKit/FeedItem.h#L30-L32) | Treat as Phase 4 data-model evolution, not an urgent demonstrated defect. |

## Rejected or deferred ideas

- **Immediate Objective-C-to-Swift rewrite:** rejected. The existing refined bridge is functioning as a useful modernization seam, and no source evidence requires a language rewrite.
- **Immediate persistence-format replacement:** deferred. The current format is weakly validated, but no observed data-corruption incident establishes that a full migration is the first safe change. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L61-L71)
- **Immediate UIKit deletion:** rejected pending Phase 0 and Phase 1. Inactivity at launch is not proof that UIKit is unsupported.
- **Whole-app migration from `ObservableObject` to Observation:** deferred. Incremental coexistence is possible, while deployment constraints and measured need remain unknown.
- **Replacing the photo-picker representable solely to remove UIKit:** rejected. The representable is an appropriate interoperability pattern for a UIKit view controller in SwiftUI.
- **Claiming local attachment URLs fail in `URLSession`:** unverified. The loader uses `URLSession`, but this plan contains no runtime evidence that it fails. [`URLImage.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/URLImage.swift#L27-L43)
- **Treating `NavigationView`, Combine, or custom image loading as debt purely due to age:** deferred. They may warrant future review, but source inspection alone does not establish a defect or product cost.

## Open questions

1. What OS versions and deployment targets must BabyTrack support?
2. Is the UIKit route retained for production, migration fallback, learning material, or no current purpose?
3. Are existing user documents/assets required to remain readable indefinitely, and is data migration acceptable?
4. What should users see when event or image persistence fails?
5. Does picker cancellation visibly leave the SwiftUI sheet on screen on the supported OS versions?
6. Does the current attachment loader work reliably for the local Documents URLs used by the app?
7. Are there concurrency scenarios beyond current single-scene, foreground use that can call persistence simultaneously?
8. Is the synchronous disk read during `FeedView` body evaluation measurable as a user-facing performance issue?

## Validation summary

| Area | Required evidence before advancement |
|---|---|
| Product support | Confirmed target/scheme/deployment configuration and UIKit support decision |
| Data compatibility | Existing feed records and image attachments read correctly before and after each data-boundary change |
| Persistence failures | Failed text/image writes are observable, do not masquerade as durable records, and have defined user behavior |
| UI parity | Supported UIKit and SwiftUI paths create equivalent feed records and display equivalent data |
| Picker flow | Success, cancellation, unsupported result, load failure, relaunch, and attachment display are covered |
| State logic | Feed order and Sleep/Awake transitions are verified against persisted histories |
| Performance work | Measurements show a relevant bottleneck before changes to observation or synchronous state lookup |
