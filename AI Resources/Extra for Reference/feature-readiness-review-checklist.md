# Feature Readiness Review Checklist

Use this checklist when independently reviewing a proposed feature-readiness plan.

## Evidence

- Does every project-specific claim cite a real file, symbol, dependency, or behavior?
- Do the cited sources support the claim?
- Are likely change points separated from confirmed requirements?
- Are assumptions and unresolved questions clearly labeled?

## Scope

- Does the plan cover the complete feature brief?
- Has it introduced unrelated cleanup, modernization, or implementation work?
- Does it preserve explicit exclusions?
- Can any proposed change be removed without affecting the feature goal?

## Change impact

- Does the plan trace the UI, model, interoperability, persistence, and attachment-storage boundaries?
- Does it identify data-identity and compatibility concerns?
- Does it consider behavior shared by the SwiftUI and retained UIKit paths?
- Are dependencies and ordering constraints visible?

## Risk and validation

- Are data loss, accidental deletion, orphaned attachments, ordering, and regression risks considered where supported?
- Does the plan identify missing automated tests without inventing coverage?
- Are manual and automated validation needs practical and specific?
- Are important failure paths or edge cases missing?

## Human decision

- Verify each review finding against the source.
- Accept, revise, reject, or leave each major finding unresolved.
- Record why the final scope and readiness recommendation are appropriate.
- Confirm that rejected recommendations do not return in the final plan.
