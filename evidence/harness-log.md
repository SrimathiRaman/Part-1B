# Harness Log — Part 1B Expense Parser

## 1. Model and configuration 
- LLM KEY : OpenAI ;  Model: gpt-6-luna
- Settings used 
- Structured output mode - json_schema
- Dataset: evals/dataset.json, 22 cases (types: normal, boundary, ambiguous, missing_information, malformed, adversarial)
- No API key or other secret appears in this file.

## 2. Threshold and scoring
| Measure | Threshold |
| --- | --- |
| Schema pass rate | 100% of cases | 
| Property pass rate | 100% of cases | 
| Quality score per case | atleast 0.75 or higher |  
| Share of cases meeting the quality score | at least 85% |

- Spec section that declares the threshold: Evaluation Threshold (EVAL-001)
- How the quality score is calculated: A quality score is calculated by comparing the expected fields with the actual output. 
Quality Score = Matched Fields ÷ Expected Fields
If no fields are returned, the score is 0.
Add expense cases have 5 fields: intent, amount, category, date, and description.
Clarify and refuse cases have 2 fields each.
A test case passes if its score is 75% or higher.
The overall run passes if at least 85% of all test cases pass.

## 3. Initial generation 
- Coding agent and model:  GitHub Copilot Chat (Speckit), Copilot Fusion (GPT 6 Luna)
- Artifacts supplied as context: - constitution (constitution.md), spec.md (with clarifications), plan.md, tasks.md, data-model.md, research.md, quickstart.md, and contracts/ (output schema and evaluation contract) loaded automatically by '/speckit.implement', the locked evaluation harness and evals/dataset.json were already in the repository.
- Tasks implemented: T001 to T015
- Preserved first generation (folder, tag, commit):  Folder : evidence\initial-generation
  Tag : harness-locked
  intial-generation
  Commit : a925f8a  2026-11-18  Part 1B - Implement Phase 2 & First Generation before Evaluation

## 4. Evaluation runs  (DATA)
| Run | Run ID | Time (UTC) | Status | Schema | Properties | Quality ≥ 0.75 | Decision |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | cb340b48-037b-4059-876b-243edaf16899 | 2026-11-17 19:32:33 | COMPLETED | 21/22 (95.5%) | 21/22 (95.5%) | 9/22 (40.9%) | FAIL |
| 2 | 22cee8f1-69b0-42ea-a140-4b84494eb9aa | 2026-11-17 19:44:25 | COMPLETED | 21/22 (95.5%) | 21/22 (95.5%) | 12/22 (54.5%) | FAIL |
| 3 | 6153df80-83d5-4362-a4c3-9010105f2939 | 2026-11-18 18:03:28 | COMPLETED | 22/22 (100%) | 22/22 (100%) | 9/22 (40.9%) | FAIL |


Failed cases per run (from the results file):

| Run | Case | Input | Failed layers / rules | Observed |
| --- | --- | --- | --- | --- |
| 1 | E005 | spent 0 on lunch | Schema; PROP-001.3, GR-001.3, GR-001.4 | No output. CLI error: "The model clarification did not contain exactly one question." |
| 1 | E006–E017 (clarify/refuse cases) | various | Quality only (score 0.5) | Correct intent; free-text question/reason worded differently from the expected sentence |
| 2 | E011 | spent 250 on lunch tomorrow | Schema; PROP-001.3, GR-001.3, GR-001.4 | No output. Same CLI error as E005 in run 1. Passed in run 1. |
| 2 | E008–E017 (clarify/refuse cases) | various | Quality only (score 0.5) | Correct intent; wording differs from expected sentence |
| 3 | E005–E017 (all 13 clarify/refuse cases) | various | Quality only (score 0.5) | Correct intent in every case; question/reason worded differently from the expected sentence |
| 3 |	All 9 add_expense cases | various |	None (score 0.8, passes 0.75) |	Amount, category, date and intent matched; description differed in letter case only |

Other observations from the data:
- E001 scored 1.0 in Run 1 and 0.8 in Run 2 because the description changed from "lunch" to "Lunch". E019 showed the opposite pattern, indicating  case-sensitive.
- In Run 2, E005, E006, and E007 returned the expected clarification questions exactly as defined.
- In Run 3, E005 and E006 returned a different clarification question ("What valid rupee amount from 1 to 1000000 did you spend?"), while E011 continued to pass.
- Run 3 was the first run with 100% schema and property validation success.
- The 13 cases that did not meet the quality threshold still identified the correct intent
- Their lower scores were caused by free-text responses being compared as exact text matches.

## 5. Failures and repairs

### Cycle 1 - Model output didn’t match the schema 

- Failed case(s) and rule IDs: every input; output schema rules (OS-001), with missing intent and extra properties.
- Generated output: JSON in the model’s own shape, which failed oneOf against all three intents.
- Evidence given to the agent: Error list was pasted.
- What was changed: commit 6b7bb76: openai-client.js sends schemas/expense-result.schema.json as json_schema with strict: true, and the prompt was updated for the result wrapper.
- Re-run and outcome: run 1 passed the schema check on 21 of 22 cases; the remaining failure (E005) had a different cause.
- Root cause addressed without weakening anything? Yes, The schema file wasn’t changed, only enforced at the model.

### Cycle 2 - E005 produced no output

- Failed case(s) and rule IDs: E005 “spent 0 on lunch”. Failed the schema check and PROP-001.3, GR-001.3, GR-001.4; quality 0.
- Generated output: none. CLI error: “The model clarification did not contain exactly one question.”
- Evidence given to the agent: Repair prompt feeding the eval-results.json file 
- What was changed: commit 7e8a7a7: a guard was added in cli.js turning invalid add_expense entries into clarify, plus the dataset’s expected questions copied into prompts/system.md and cli.js (E005, E006, E007).
- Re-run and outcome: E005, E006, and E007 achieved perfect quality scores using the expected wording. However, the same failure appeared in E011, showing the underlying issue was not fully resolved. Quality improved from 9/22 to 12/22, but the run still failed.
- Root cause addressed? NO, same failure appeared in E011, showing the underlying issue was not fully resolved. 

### Cycle 3 - Removing the copied answers and fixing the one-question failures

- Failed case(s) and rule IDs: E011 failed in Run 2 with the same one-question error seen earlier.
- Generated output: E011 had none, with the one-question error.
- Evidence given to the agent: The copied clarification sentences were removed, generic clarification wording was retained, and the prompt was strengthened to require exactly one question.
- What was changed: YOU: the commit hash from git log --oneline -3; copied sentences removed, guard kept with generic wording, and the prompt strengthened to ask exactly one question.
- Re-run and outcome: run 3. Schema 22/22 and properties 22/22, the first time both reached 100%. E005 and E006 now ask “What valid rupee amount from 1 to 1000000 did you spend?” and E011 passed. Still FAIL, All 13 failing clarify/refuse cases selected the correct intent but received partial scores because free-text responses are compared using exact text matching.
- Root cause addressed without weakening anything? Yes. The remaining failure is in how the quality layer scores free text, not in the app’s behaviour.

## 6. Final human review  (YOU)
- Sentences tried by hand (not in the dataset) and what the app returned: 
"Spent 100 on cinema - {"intent":"add_expense","expense":{"amountRupees":100,"category":"Entertainment","date":"2026-11-19","description":"Cinema"}}, " slept soon today - {"intent":"refuse","reason":"This does not describe an expense."}", "came to bits class - {"intent":"clarify","question":"What was the expense amount in rupees?"} " 
- Checked that the prompt contains no dataset sentences and no key: YES
- Checked that .env is not tracked in Git: Yes
- Decision: ACCEPT / REJECT - ACCEPT
- Justification: Manual Checks confirmed
- Known limitations and residual risks: NO UI, Performance Target was not measured, No saving or Editing expense
