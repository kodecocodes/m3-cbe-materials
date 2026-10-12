# Modernization Plan Review Checklist

Use this checklist before accepting an AI-generated modernization plan.

## Evidence and purpose

- Does every proposed change support a verified project need or goal?
- Are project-specific claims tied to real files, symbols, dependencies, or behavior?
- Is Objective-C, UIKit, or code age being treated as a problem without evidence?
- Does the plan distinguish confirmed facts from assumptions and open questions?

## Scope

- Is each phase bounded enough to review and validate?
- Does the plan preserve code that does not need to change?
- Has the AI introduced a rewrite, unrelated cleanup, or speculative architecture work?
- Should any recommendation be revised, deferred, or rejected?

## Sequencing and interoperability

- Does the sequence respect dependencies between components?
- Does it explain how Objective-C and Swift can coexist where relevant?
- Does it explain how UIKit and SwiftUI can coexist where relevant?
- Can the team stop after a phase without leaving the app in an unusable state?

## Risk and validation

- Does each phase identify realistic regression risks?
- Is there a practical way to validate behavior after each phase?
- Are testing gaps or manual checks clearly stated?
- Are unresolved decisions marked instead of guessed?

## Human decision

- Accept, reject, revise, or defer each major recommendation.
- Record why the scope and order fit this project.
- Confirm that rejected work does not return in the final plan.
