# BabyTrack codebase assessment

## Scope and evidence

This assessment is based on the source visible in the Xcode project and a successful build-for-testing request. It distinguishes direct source observations from conclusions that require runtime or project-configuration verification.

## Codebase map

### Entry points

- [`AppDelegate`](../Starter/BabyTrack/BabyTrack/Boilerplate/AppDelegate.swift#L8-L10) is the application delegate.
- [`SceneDelegate`](../Starter/BabyTrack/BabyTrack/Boilerplate/SceneDelegate.swift#L13-L18) creates the window. Its active startup path installs a `UIHostingController` whose root view is [`FeedView`](../Starter/BabyTrack/BabyTrack/Boilerplate/SceneDelegate.swift#L28-L32).
- The app plist also names `Main` as the main storyboard ([`Info.plist`](../Starter/BabyTrack/BabyTrack/Boilerplate/Info.plist#L42-L45)), but the scene delegate assigns the root view controller directly. The storyboard is therefore not the active root-view path.

### Screens and user flow

| Screen/component | Responsibility | Evidence |
|---|---|---|
| `FeedView` | Active root screen. Displays the action bar and a list of feed items; presents the moment picker. | [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L9-L39) |
| `AddFeedItemBar` | Presents Bottle, Food, Sleep/Awake, Diaper, and Moment actions. Sleep is represented as Awake when the feed says the baby is sleeping. | [`AddFeedItemBar.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddFeedItemBar.swift#L13-L31) |
| `AddMomentView` | Wraps `PHPickerViewController`, accepts one image, stores it, and adds a Moment record. | [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L10-L72) |
| `FeedCell` | Renders an item’s emoji, title, relative timestamp, and optional attachment. | [`FeedCell.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedCell.swift#L9-L45) |
| UIKit `ViewController` / `FeedCell` | A parallel UITableView/storyboard implementation of the same feed workflow. It is not selected by the active startup path. | [`ViewController.m`](../Starter/BabyTrack/BabyTrack/UIKit/ViewController.m#L34-L112), [`SceneDelegate.swift`](../Starter/BabyTrack/BabyTrack/Boilerplate/SceneDelegate.swift#L20-L32) |

### Data flow

```text
FeedView (@StateObject Feed)
  -> BabyKit Swift Feed facade
  -> refined Objective-C __Feed persistence API
  -> Documents/feed.data and Documents/<attachment UUID>.jpg
  -> @Published items update FeedView's List
```

- The Swift-facing `Feed` class is `@MainActor`, owns `@Published items`, loads on initialization, and delegates storage calls to a refined Objective-C `__Feed`. [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L10-L40)
- Text events are appended to `feed.data` in the Documents directory. The loader parses lines and inserts each at index zero, yielding newest-first display order for append-order storage. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L47-L74), [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L83-L94)
- Moment images are resized, encoded as JPEG, stored in Documents using a generated UUID, and linked to the event through `attachmentId`. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L149-L169)
- Attachment URLs are reconstructed in the app target from `attachmentId`. [`FeedItem+Ext.swift`](../Starter/BabyTrack/BabyTrack/FeedItem+Ext.swift#L9-L18)
- The current state used for the Sleep/Awake action is derived by reloading records and comparing the first Sleep and Awake records in newest-first order. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L33-L45)

### Dependencies

- In-project framework: `BabyKit`, exposing Objective-C `Feed` and `FeedItem` APIs through [`BabyKit.h`](../Starter/BabyTrack/BabyKit/BabyKit.h#L6-L15).
- Apple frameworks used directly: SwiftUI, UIKit, Foundation, PhotosUI, Combine, and URLSession. [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L6-L8), [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L6-L8), [`URLImage.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/URLImage.swift#L6-L43)
- No package dependencies were observed in the visible Xcode project structure.

### Tests and build evidence

- No files matching `*Tests*` or `*UITests*` were found in the visible Xcode project structure, and no XCTest or Swift Testing imports were found in the source.
- The project built successfully with a build-for-testing request.
- **Limit:** this does not establish whether an empty or externally managed test target exists; it establishes only that no test source was visible and that the build request succeeded.

## Prioritized technical debt

### Address now

#### 1. Persistence failures are silently represented as successful in-memory changes

`addFeedItem:` creates a record, opens a file handle with `error:nil`, writes with `error:nil`, and returns the record regardless of whether opening or writing succeeded. The Swift facade inserts that returned record into the published in-memory list. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L83-L94), [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L20-L22)

**Verified risk:** a failed event write can be shown during the current session but be absent after a reload.

`storeImage:` also ignores the result of the JPEG write and returns an attachment UUID after attempting the write. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L160-L169)

**Verified risk:** a Moment record can refer to an attachment UUID for which no image was saved.

**Tradeoff:** surfacing storage failure adds error-path handling across the Objective-C/Swift boundary, but this code stores the app’s primary user data.

#### 2. The active photo-picker flow does not reset its presentation binding for cancellation or an unsupported result

The SwiftUI picker delegate returns when no picker result exists or the provider cannot load a `UIImage`; neither route writes `isPresented = false`. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L39-L43) The binding is reset only inside the image-load completion task. [`AddMomentView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/AddMomentView.swift#L53-L69)

**Verified state-management gap:** cancellation and unsupported-result paths leave the SwiftUI presentation binding unchanged. The legacy UIKit picker explicitly dismisses on an empty result. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L110-L115)

**Tradeoff:** handling every terminal picker outcome is a small amount of additional state/error behavior, but avoids relying on unspecified interaction between the UIKit picker and the enclosing SwiftUI sheet.

#### 3. No visible automated test coverage protects primary behavior

No test source files were found. The persistence parser, write-failure paths, feed ordering, Sleep/Awake derivation, and picker cancellation path therefore have no observed automated coverage.

**Tradeoff:** test maintenance costs time, but this is the safety net needed before changing the data layer.

### Plan later

#### 4. The persistence format is permissive and weakly validated

Records are serialized as comma-separated fields. The loader silently skips lines that do not have exactly three components, while numeric values are accepted via `intValue` and `doubleValue` without semantic validation. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L61-L71)

**Verified condition:** malformed records can be silently omitted or interpreted as default numeric values.

**Not established:** the assessment found no evidence that a user’s current data is corrupted.

**Tradeoff:** a versioned structured store would improve validation and evolvability but introduces migration/versioning work; retaining the format while improving validation is a narrower path.

#### 5. Feed state performs synchronous file reads while the SwiftUI body is evaluated

`FeedView` reads `feed.isBabySleeping` during `body` evaluation. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L13-L25) That property calls `loadFeedItems`, which reads the storage file synchronously. [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L33-L45), [`Feed.m`](../Starter/BabyTrack/BabyKit/Feed.m#L47-L59)

**Verified condition:** view construction can trigger disk reads.

**Tradeoff:** caching a derived state shifts responsibility to keeping that state synchronized. The present implementation is simple and may be adequate for the current small data set, so this is capacity/performance planning rather than a confirmed defect.

#### 6. Feed list identity is timestamp-based rather than event-based

The feed list identifies rows by `FeedItem.date`. [`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L28-L30) Feed records do not carry a persistent event identifier; the only UUID is optional and belongs to image attachments. [`FeedItem.h`](../Starter/BabyTrack/BabyKit/FeedItem.h#L19-L33)

**Verified condition:** row identity depends on timestamps being unique.

**Not established:** a duplicate timestamp or incorrect list update has not been observed.

**Tradeoff:** event identifiers add stored data and migration considerations, but make UI identity independent of timing and support future item-level operations.

## Rejected or unverified claims

| Claim | Status | Reason |
|---|---|---|
| “Local attachment images fail because `URLSession` is used for `file:` URLs.” | Unverified; not treated as debt. | `URLImage` does use `URLSession.dataTaskPublisher` with attachment URLs ([`URLImage.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/URLImage.swift#L32-L42)), but no runtime test was performed to establish failure in this app. |
| “There is no test target.” | Unsupported. | No visible test source files were found, but source inspection cannot prove target configuration is absent. |
| “The UIKit implementation must be removed.” | Rejected as a defect claim. | UIKit is demonstrably inactive in the current launch path, but retaining it may be intentional for migration, comparison, or teaching. |
| “`NavigationView`, Combine, or custom image loading are technical debt because they are old.” | Rejected as a source-supported defect claim. | Their use is visible ([`FeedView.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/FeedView.swift#L13-L39), [`Feed.swift`](../Starter/BabyTrack/BabyKit/Feed.swift#L6-L18), [`URLImage.swift`](../Starter/BabyTrack/BabyTrack/SwiftUI/URLImage.swift#L27-L43)), but age or style preference alone does not demonstrate a product or maintainability problem. |
| “The persistence format is already corrupting user data.” | Unsupported. | The format and error handling are fragile, but no corrupted data sample or runtime failure was observed. |

## Open questions

1. Does cancelling the Photos picker leave the SwiftUI sheet visibly presented on supported OS versions, or does UIKit independently dismiss it despite the binding remaining true?
2. Do local `file:` attachment URLs successfully load through the current `URLSession` pipeline in the app’s supported deployment environments?
3. Are writes ever performed concurrently—for example through rapid user interactions, scene transitions, or future background work—and if so, what serialization guarantee is required?
4. What data-retention expectation applies to orphaned image files when an image is written but its corresponding feed record cannot be saved?
5. Is the UIKit implementation intentionally retained, and is it compiled/shipped in production configurations?
6. What deployment targets and supported OS versions constrain future API choices? These were not established by the visible source files.

## Next investigation

1. Exercise the Moment flow on a simulator or device: cancel the picker, choose an unsupported/unloadable provider if reproducible, choose a valid photo, then relaunch and inspect the feed and image.
2. Inject or simulate failures for feed-file opening, feed-file writing, image writing, and malformed `feed.data`; record the user-visible and persistence outcomes.
3. Inspect the project’s target membership, build settings, and schemes to confirm test-target status, deployment targets, and whether the UIKit sources ship in the product target.
4. Measure feed body evaluation and disk-read frequency with a representative history size before classifying the synchronous state lookup as a performance problem.
5. Create a minimal reproducible runtime check for attachment loading from the Documents directory before making any claim about the image-loader transport.
