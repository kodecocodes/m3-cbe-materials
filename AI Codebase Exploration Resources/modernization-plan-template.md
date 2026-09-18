# Modernization Plan

## Goal and constraints

Describe the verified reason for considering modernization, the intended outcome, and the boundaries of this plan.

## Current state

Summarize the relevant Objective-C, Swift, UIKit, and SwiftUI areas. Cite important files or symbols.

## Modernization principles

List the principles guiding decisions, such as incremental delivery, interoperability, bounded scope, behavior preservation, and validation.

## Objective-C-to-Swift approach

Explain whether and where an incremental language migration could provide value. Include important dependencies and interoperability considerations.

## UIKit-to-SwiftUI approach

Explain whether and where incremental SwiftUI adoption could provide value. Include hosting, navigation, state, and interoperability considerations where relevant.

## Proposed phases

For each phase, include:

- **Goal:** The verified outcome the phase supports.
- **Scope:** The bounded area being considered.
- **Dependencies:** What must already be understood or completed.
- **Interoperability:** How old and new code would coexist.
- **Risks:** What could go wrong.
- **Validation:** How behavior would be checked.
- **Human decision:** Accepted, revised, deferred, rejected, or unresolved.

## Retain as-is

Identify relevant code that does not currently justify migration and explain why it should remain unchanged.

## Risks and mitigations

Summarize risks that affect more than one phase and the proposed safeguards.

## Deferred or rejected recommendations

Record important AI recommendations that were deferred or rejected and explain the decision.

## Open questions

List questions requiring more investigation, product context, historical knowledge, or stakeholder input.

## Validation strategy

Summarize how the team would validate behavior throughout modernization without providing implementation instructions.
