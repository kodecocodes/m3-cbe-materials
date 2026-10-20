# Delete Feed Entry — Feature Readiness Plan

## Purpose and scope

This plan assesses readiness for the provided [delete-feed-entry feature brief](../../AI%20Codebase%20Exploration%20Resources/delete-feed-entry-feature-brief.md): delete one unwanted entry from the active SwiftUI feed, require confirmation, persist the result, preserve unrelated entries and ordering, and remove a deleted Moment’s orphaned image.

The plan does not implement the feature. It preserves unresolved questions and limits scope to the requested single-entry deletion workflow.

## Current state

### Active user path

The scene delegate installs a SwiftUI `FeedView` in a `UIHostingController`; its separate UIKit startup method is not selected by the active launch path. [`SceneDelegate.swift`](../Starter/BabyTrack/BabyTrack/Boilerplate/SceneDelegate.swift#L13-L32)

`FeedView` owns a Swift `Feed` model, displays `feed.items` in a `List`, and renders each item through `FeedCell`. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L9-L39) [`FeedCell.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedCell.swift#L9-L45)

### Data path

The Swift `Feed` is a `@MainActor` `ObservableObject` facade over refined Objective-C `__Feed`. It loads items at initialization and publishes in-memory additions. [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L10-L40) The underlying Objective-C `Feed` API is marked `NS_REFINED_FOR_SWIFT`. [`Feed.h`](../Starter/BabyTrack/BabyKit/Feed.h#L14-L28)

The Objective-C implementation appends event records to `Documents/feed.data` as three comma-separated fields: kind, timestamp, and optional attachment UUID. It loads lines into newest-first display order. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L47-L74) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L76-L94)

Moment images are separate JPEG files in Documents, named from `attachmentId`. The app-target Swift extension reconstructs the attachment URL from that UUID. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L149-L169) [`FeedItem+Ext.swift`](../Starter/BabyTrack/BabyTrack/FeedItem+Ext.swift#L9-L18)

### Existing item identity

`FeedItem` has `kind`, `date`, and optional `attachmentId`; it has no persistent event identifier. [`FeedItem.h`](../Starter/BabyTrack/BabyKit/FeedItem.h#L19-L33) SwiftUI currently uses `date` as its list identity. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L28-L30)

This does not establish that a timestamp is an unambiguous durable deletion key. In particular, the source does not rule out duplicate entries with the same kind, date, and no attachment.

### Retained UIKit source

UIKit `ViewController` and `FeedCell` source files exist and use the Objective-C `Feed` API directly. [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L11-L17) [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L34-L64) Their target membership and product-support status are not established by this assessment.

## Readiness assessment

Deletion is feasible through the existing architecture, but implementation is not ready until the application resolves four data-integrity decisions:

1. A durable, unambiguous way to select one persisted entry.
2. The compatibility behavior for existing three-field records.
3. Attachment ownership and cleanup behavior, including cleanup failure.
4. Read-modify-write concurrency behavior for deletion versus addition.

The appropriate ownership boundary is the existing `Feed`/`__Feed` data layer, with a SwiftUI initiation and confirmation flow above it. A UI row cannot independently guarantee durable feed updates or attachment cleanup.

## Change points

| Area | Files | Role in the feature | Readiness concern |
|---|---|---|---|
| SwiftUI list and confirmation state | [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L9-L39) | Active feed surface; owns the `Feed` instance and list rendering. | Needs selected-entry and confirmation behavior without redesigning the feed. |
| Individual entry UI | [`FeedCell.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedCell.swift#L9-L45) | Renders the entry but has no deletion affordance or action callback. | Must identify the selected entry without treating display identity as durable identity. |
| Swift-facing data boundary | [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L10-L40) | Publishes the in-memory item array and delegates to `__Feed`. | Must expose explicit deletion success/failure semantics and keep `items` coherent with durable state. |
| Objective-C public API | [`Feed.h`](../Starter/BabyTrack/BabyKit/Feed.h#L14-L28), [`BabyKit.h`](../Starter/BabyTrack/BabyKit/BabyKit.h#L6-L15) | Defines the refined legacy API and framework export surface. | Any public deletion contract requires review of both the header and exported framework surface. |
| Persistence parse/write path | [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L47-L94) | Reads existing records and appends new records. | Deletion requires safe targeted update while preserving retained records and ordering. Parser and serializer must be treated as one compatibility unit. |
| Item model and construction | [`FeedItem.h`](../Starter/BabyTrack/BabyKit/FeedItem.h#L19-L33), [`FeedItem.m`](../Starter/BabyTrack/BabyKit/FeedItem.m#L10-L42) | Defines fields and constructors. | Review is required if the accepted identity design changes stored item data. |
| Attachment path and cleanup | [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L149-L169), [`FeedItem+Ext.swift`](../Starter/BabyTrack/BabyTrack/FeedItem+Ext.swift#L9-L18) | Storage writes JPEGs; app target reconstructs their URLs. | The framework cannot rely on an app-target Swift extension. Attachment-path ownership must be explicit and consistent. |
| Derived feed state | [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L33-L45), [`AddFeedItemBar.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddFeedItemBar.swift#L13-L31) | Sleep/Awake action is derived from persisted history. | Deleting Sleep or Awake can change the next available action. |
| Retained UIKit compatibility | [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L34-L82) | Loads persisted feed data into an independent in-memory array. | If UIKit remains supported, it needs persisted-data compatibility; the brief does not require a UIKit delete control. |

## Phased readiness plan

### Phase 1 — Define the selected-record identity contract

**Goal:** Establish how a visible entry maps to exactly one durable record.

**Scope:** Determine whether existing fields can safely identify a record; define behavior for duplicate records; determine whether compatibility-preserving identity evolution is acceptable for existing data.

**Dependencies:** None.

**Interoperability:** Preserve the current ability to load existing three-field records until the compatibility approach is decided. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L61-L71)

**Risks:** Using `date` because it is the current SwiftUI list identifier confuses UI diffing identity with durable deletion identity. Matching by kind/date/attachment remains ambiguous unless uniqueness is guaranteed.

**Validation:** Use representative fixtures containing duplicate event kinds, duplicate or close timestamps, Moments, non-Moments, and legacy records. Demonstrate that the selected UI row maps to one and only one persisted record.

**Decision gate:** Do not define the storage-update algorithm until this contract is accepted.

### Phase 2 — Define durable record removal and attachment ownership

**Goal:** Define a permanent operation that preserves unrelated feed data and does not leave a deleted Moment image orphaned.

**Scope:** Define retained-record ordering, malformed-record policy, attachment ownership/reference rules, cleanup ordering, and recovery behavior for partial failure.

**Dependencies:** Phase 1 identity contract.

**Interoperability:** Keep the existing Swift `Feed` facade over `__Feed`; do not replace the persistence system solely to support deletion. [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L14-L39)

**Risks:**

- Deleting the attachment before updating feed storage can leave a retained Moment pointing to a missing image if the feed update fails.
- Updating feed storage before image cleanup can leave an orphaned JPEG if cleanup fails.
- Attachment IDs are optional model data and the source does not enforce that one UUID can belong to only one feed record. [`FeedItem.h`](../Starter/BabyTrack/BabyKit/FeedItem.h#L26-L32)
- The existing parser silently skips lines that do not have exactly three components. A rewrite based only on parsed records could discard malformed stored lines. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L61-L71)
- The current write path ignores file-operation errors; it cannot by itself define reliable deletion outcomes. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L83-L94) [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L160-L169)

**Validation:** Verify that deletion preserves every unrelated record and their order; delete a Moment and verify its attachment is absent after relaunch; verify other Moment attachments remain; exercise feed-update and cleanup-failure paths; verify malformed-record handling matches the approved policy.

**Decision gate:** Approve product behavior for partial failures before adding the user-facing control.

### Phase 3 — Add the bounded SwiftUI workflow

**Goal:** Meet the brief’s initiation, confirmation, cancellation, and single-item behavior in the active SwiftUI feed.

**Scope:** Add an entry-level deletion initiation path, retain the selected durable identity for confirmation, keep cancellation non-destructive, and invoke the data-boundary operation only after confirmation.

**Dependencies:** Phase 1 and Phase 2 decisions.

**Interoperability:** Preserve the `FeedView` → Swift `Feed` → Objective-C `__Feed` path. Do not change the photo-picker bridge, add-item bar, or unrelated feed design. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L16-L37) [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L10-L72)

**Risks:** The selected record can become stale while confirmation is shown; deletion of Sleep/Awake entries changes derived action state; accessibility and wording require review; a failed durable deletion must not be removed only from the visible list.

**Validation:** Confirm cancel leaves the list, storage, and attachment unchanged; confirm deletion removes only the selected item; relaunch and verify persistence; delete Sleep and Awake entries in representative histories and verify the next action-bar state.

### Phase 4 — Confirm retained-client compatibility and release readiness

**Goal:** Ensure the bounded SwiftUI feature does not break any intentionally supported legacy client.

**Scope:** Confirm UIKit’s product role. If it remains supported, verify that it loads and renders data after deletion in SwiftUI. Do not add UIKit deletion UI unless a separate product decision requires it.

**Dependencies:** Phase 3 feature behavior and a Phase 0-equivalent support decision.

**Interoperability:** UIKit reloads storage in `viewDidLoad` and maintains a separate array. The feature brief requires persistence after relaunch; it does not require live cross-UI synchronization in one process. [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L34-L44)

**Risks:** Expanding deletion into UIKit broadens scope unnecessarily; assuming the visible UIKit source is shipped or supported is unsupported.

**Validation:** If UIKit is supported, create records, delete one in SwiftUI, then load the UIKit feed and verify retained entries, ordering, and attachments. Build and run the active scheme.

## Required validation matrix

| Area | Required evidence |
|---|---|
| Confirmation | A person can initiate deletion for one entry; cancellation mutates neither visible state nor persisted state; confirmation starts only one deletion. |
| Identity | Duplicate and legacy records are not accidentally co-deleted; the approved selected-record mapping is unambiguous. |
| Persistent deletion | The selected record is absent after relaunch; unrelated records and newest-first ordering remain unchanged. |
| Moment lifecycle | Deleting a Moment removes its record and only its owned attachment; no retained record points to a removed attachment. |
| Failure behavior | Feed-update failure, image-cleanup failure, missing image, and malformed data follow approved product behavior and do not misrepresent the outcome in the UI. |
| Derived state | Deleting Sleep, Awake, and unrelated events produces the correct subsequent Sleep/Awake action. |
| Regression | Existing add actions, Moment picking, feed rendering, and relaunch remain functional. |
| Retained UIKit | If UIKit is supported, it loads the post-deletion persisted feed correctly; live synchronization is not assumed unless separately required. |
| Accessibility | The deletion action and confirmation are understandable and operable with assistive technologies. |

No test sources matching `*Tests*` or `*UITests*` were visible in the project structure. This is evidence of no visible test source, not proof that a test target is absent.

## Rejected or deferred scope

- Bulk deletion, undo, recently deleted, cloud synchronization, multi-user behavior, and complete feed redesign are out of scope under the feature brief.
- Rewriting persistence solely for modernization is rejected. The deletion operation should be defined against the existing architecture unless evidence requires broader change.
- Completing an Objective-C-to-Swift or UIKit-to-SwiftUI migration is out of scope.
- Adding a UIKit deletion control is deferred unless the product confirms UIKit is a supported user-facing route.
- A whole-app Observation/Combine migration, navigation migration, or image-loader rewrite is not a dependency of deletion.
- Treating date-based `List` identity as the durable deletion key is rejected until Phase 1 proves an unambiguous mapping.
- Assuming that attachment UUIDs are unique per record is rejected; the source does not enforce that invariant.
- Assuming local attachment URLs fail to load through the current image loader is rejected as unverified; no runtime evidence was collected. [`URLImage.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/URLImage.swift#L27-L43)

## Open questions

1. What durable identity selects exactly one persisted entry, particularly when existing records are duplicates?
2. Must legacy three-field `feed.data` records remain writable as well as readable?
3. If a format evolution is accepted, how are existing records identified and upgraded without violating compatibility?
4. Can multiple feed records reference the same attachment UUID, and how should cleanup behave if they can?
5. Which operation outcome is correct when the feed record and JPEG cannot both be removed: fail, recover later, or another explicitly defined state?
6. What is the required policy for malformed lines during a record-rewrite operation: preserve, recover, or discard?
7. Can add and delete operations overlap, and what serialization policy is required to prevent lost updates?
8. What user-visible error behavior is required when permanent deletion cannot complete?
9. Is UIKit a supported production interface, a fallback, or non-product sample source?
10. If UIKit is supported, does it require only persisted-data compatibility or its own delete control?
11. What deployment targets constrain the confirmation UI choice?
12. Are there retention, recovery, audit, or legal requirements that conflict with permanent deletion?

## Implementation readiness gate

Feature implementation is ready to begin only after all of the following are decided and testable:

1. Selected-record identity is unambiguous for legacy and newly created records.
2. Existing-data compatibility behavior is approved.
3. Attachment ownership and partial-failure behavior are approved.
4. Malformed-record and add/delete concurrency policies are approved.
5. The active SwiftUI confirmation flow’s expected outcomes are specified.
6. Retained UIKit support status is confirmed.

Until then, deletion should remain a planned feature rather than an implementation task.
