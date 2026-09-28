# Technical Debt Review Checklist

Use this checklist before accepting an AI-generated technical-debt finding.

## Evidence

- Does the finding cite a real file, symbol, dependency, test, or observed behavior?
- Did you inspect the cited source yourself?
- Does the code support the claim, or is the AI inferring intent?
- Is the finding confirmed, suspected, or unresolved?

## Actual cost or risk

- Does the issue create incorrect behavior, fragility, security risk, maintenance cost, or unnecessary difficulty when changing the app?
- Is the AI calling something debt only because it is old, Objective-C, UIKit, or unfamiliar?
- Is the recommendation based on the needs of this project rather than a generic preference?

## Priority

- How likely is the issue to cause a problem?
- What is the impact if it does?
- How frequently does this area change?
- What is the effort and risk of improving it?
- Should the team address it now, plan it for later, or consciously accept it?

## Human decision

- Accept, reject, or revise the finding.
- Record unresolved questions instead of presenting guesses as facts.
- Confirm that rejected findings do not reappear in the final assessment.
