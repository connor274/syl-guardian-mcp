# Syl Guardian: agent rules template

Drop-in instructions that make an AI coding or analytics agent verify that a claimed marketing effect is causal, using the Syl Guardian MCP tool `guardian_review`, before it gives a final recommendation.

Prerequisites: connect the MCP server and get a trial key at https://sylai.io/guardian.

```bash
claude mcp add --transport http syl-guardian https://guardian.sylai.io/mcp \
  --header "X-API-Key: <your key>"
```

Docs: https://sylai.io/guardian/docs. Machine-readable summary: https://sylai.io/llms.txt. Hosted copy of this file: https://sylai.io/guardian-agent-rules.md.

Python agents can call Guardian with `pip install syl-guardian-client` (CrewAI and LangChain tools included) instead of writing REST calls.

The rule text below is the same for every agent format. Copy the section for the file your agent reads.

---

## CLAUDE.md

```markdown
## Ad spend and budget decisions: Syl Guardian check

Whenever evaluating marketing campaigns, ad platforms, or budget allocation, verify the claimed effect is causal using the Syl Guardian MCP tool (guardian_review) before giving a final recommendation.

How to call it:
- cause = the spend column; effect = the outcome column (e.g. conversion value).
- Send only the rows for the campaign in question.
- time_col = the date column; step = the proposed budget change.
- Pass known confounders as known_confounders=[...] (the parameter is NOT named "confounders"), e.g. seasonality or brand-query share.
- Send only cause, effect, time_col and confounder columns. Drop downstream columns such as roas, clicks and conversions (they are consequences of spend and bias the estimate).
- For more than ~200 rows do not paste data: call guardian_request_upload, run the returned curl command to upload the CSV, then pass the returned dataset_id to guardian_review. Uploads are single-use; request a new one to retry.

Act on the verdict:
- pass: proceed.
- caution: proceed only with a reduced or narrowed change, and say why.
- hold: do not change the budget. Report what is needed to support the claim.
Always report the verdict, the naive vs. adjusted effect, and the confidence interval alongside the recommendation. A passing threshold check (e.g. ROAS above a target) is never sufficient on its own.
```

## AGENTS.md

```markdown
## Syl Guardian: causal check for ad spend and budget changes

Whenever evaluating marketing campaigns, ad platforms, or budget allocation, verify the claimed effect is causal using the Syl Guardian MCP tool (guardian_review) before giving a final recommendation.

- Arguments: cause = spend column, effect = outcome column, rows for the one campaign only, time_col = date column, step = proposed budget change, known_confounders=[...] (NOT "confounders").
- Drop downstream columns (roas, clicks, conversions) before sending; they are bad controls.
- Over ~200 rows: guardian_request_upload, curl the CSV to the returned URL, then call guardian_review with dataset_id.
- pass: proceed. caution: proceed with a reduced change and explain why. hold: do not change the budget.
- Report the verdict, naive vs. adjusted effect and confidence interval with every recommendation.
```

## .cursorrules

```text
# Syl Guardian: causal check for ad spend and budget changes

Whenever evaluating marketing campaigns, ad platforms, or budget allocation, verify the claimed effect is causal using the Syl Guardian MCP tool (guardian_review) before giving a final recommendation.

Call guardian_review with:
- cause = spend column
- effect = outcome column
- only the rows for the campaign being evaluated
- time_col = date column
- step = the proposed budget change
- known_confounders=[...]  (the parameter is NOT named "confounders")
Send only cause, effect, time_col and confounder columns. Drop downstream columns (roas, clicks, conversions).
If the dataset has more than ~200 rows, use guardian_request_upload, upload the CSV with the returned curl command, and pass the dataset_id instead of pasting rows.

Verdicts:
- pass: proceed
- caution: proceed only with a reduced change; explain why
- hold: do not change the budget; explain what is needed
Always report the verdict, naive vs. adjusted effect, and the confidence interval.
```

---

## Notes

- Guardian computes every number with deterministic statistical code; an LLM is never used to compute a number. For strict run-to-run reproducibility, pass `config={"cleaning": "rule", "discovery": "rule"}` (the default `llm` for those steps can change the adjustment set).
- Guardian reports whether the data support a causal reading of the claim. It does not replace a randomized test; use the `test_design` option to size a spend-cut test when the verdict is `caution` or `hold`.
