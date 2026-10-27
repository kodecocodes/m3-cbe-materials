# Lesson 1 Token Usage

## Scope

This report reconstructs the three visible Lesson 1 interactions from the recorded prompts and the completed **codebase-assessment.md**:

1. BabyTrack codebase exploration and potential technical-debt analysis
2. Source-backed recheck and prioritization of the findings
3. Creation of the final **codebase-assessment.md**

The original Xcode token telemetry is no longer available. The estimated cost prices are for GPT-5.6 Terra model, in USD as of October 2026. These values are calibrated estimates, not exact billed usage. 

## Estimated total

| Direction | Estimated tokens |
|---|---:|
| Your prompts (input) | 148 |
| Responses visible in the conversation (output) | 5,531 |
| **Total tokens**   |  **5,679** |
| **Estimated cost** | **~$0.07** |

## Estimated interaction totals

| Lesson interaction | Your prompt | Response | Total |
|---|---:|---:|---:|
| Codebase map and potential technical-debt analysis | 40 | 2,230 | 2,270 |
| Source recheck and prioritization | 45 | 1,020 | 1,065 |
| Final `codebase-assessment.md` generation | 63 | 2,281 | 2,344 |
| **Estimated total tokens** | **148** | **5,531** |  **5,679** |
| **Estimated total cost** |           |           | **~$0.07** |

## Estimation method

- The input estimate uses the actual three Lesson 1 prompts and the observed prompt-token ratios from the verified Lesson 2 and Lesson 3 reports.
- The first response is calibrated against Lesson 2's comparable code-boundary analysis response.
- The second response is calibrated against Lesson 3's comparable plan-review response.
- The final response is calculated from the actual size of **codebase-assessment.md**, using the known response-to-document ratio from Lesson 3's final Markdown generation.

## Precision and exclusions

A reasonable range for the three visible interactions is approximately **4,900–6,500 total tokens**. The central estimate above is the best-supported single value from the surviving materials.

The estimate excludes hidden system and developer instructions, internal reasoning, tool calls and tool payloads, attached-project transfer overhead, source files read by tools, and the later request to create this report. Exact billed counts would require the original Xcode or model-provider telemetry.
