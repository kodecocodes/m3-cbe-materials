# Delete a Feed Entry: Feature Brief

## Goal

Let someone remove one unwanted entry from the current BabyTrack feed without affecting any other recorded events.

## Required behavior

- A person can initiate deletion from an individual entry in the current SwiftUI feed.
- The app asks for confirmation before permanently deleting the entry.
- Canceling leaves the feed unchanged.
- Confirming removes only the selected entry.
- The deletion remains in effect after relaunching the app.
- When the deleted entry is a photo moment, its stored image does not remain orphaned.
- Existing feed ordering and all unrelated entries remain unchanged.

## Constraints

- Preserve compatibility with the existing stored feed data.
- Keep the change bounded to this feature.
- Use the existing app architecture unless verified evidence requires a broader change.
- Do not implement the feature during this module.

## Out of scope

- Bulk deletion
- Undo or recently deleted items
- Cloud synchronization
- Multi-user behavior
- Redesigning the complete feed
- Rewriting the persistence layer solely for modernization
- Completing an Objective-C-to-Swift or UIKit-to-SwiftUI migration

## Questions to investigate

- How does the current code identify a specific feed entry?
- Which layer should own deletion?
- How can the stored feed be updated without damaging other entries?
- Which code owns attachment cleanup?
- Does the retained UIKit interface need a deletion control, or only compatibility with the updated data?
- What automated or manual validation is needed before implementation?
