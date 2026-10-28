# Syl Guardian MCP

<!-- mcp-name: io.sylai/guardian -->

**Is this ROAS / spend-to-revenue lift actually causal?**

Syl Guardian is a Model Context Protocol (MCP) server that works as a causal-validation
agent guardrail. When an AI agent (Claude or any MCP client) is about to act on a claim
about ad spend or budget, such as "this campaign's ROAS is 4x, scale it up" or "ad spend
drove new customers", Guardian checks the claim against your data and returns a `pass`,
`caution` or `hold` verdict, with the naive correlation versus the confounder-adjusted
causal effect, a confidence score and a plain-English recommendation.

Every number is computed deterministically by statistical code and never by an LLM. By
default the summary is a fixed template. For strict reproducibility, set `cleaning` and
`discovery` to `rule` in `config`: the default (`llm`) for those two steps can change the
adjustment set, and so the estimate. See [Engine config](#engine-config) and
[Determinism and reproducibility](#determinism-and-reproducibility).

This repository holds the public listing metadata and connection instructions for the
hosted Guardian MCP server. Guardian itself is a hosted service.

- Endpoint: `https://guardian.sylai.io/mcp` (streamable HTTP)
- Auth: `X-API-Key` header. Get a trial key at <https://sylai.io/guardian>.
- Docs: <https://sylai.io/guardian/docs>

## Connect

Claude Code:

```bash
claude mcp add --transport http syl-guardian https://guardian.sylai.io/mcp \
  --header "X-API-Key: <your key>"
```

Claude Desktop (via `mcp-remote`):

```json
{
  "mcpServers": {
    "syl-guardian": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://guardian.sylai.io/mcp",
               "--header", "X-API-Key:${SYL_API_KEY}"],
      "env": { "SYL_API_KEY": "<your key>" }
    }
  }
}
```

Then ask your agent: *"Use guardian_review on this data: did ad_spend cause
new_customers? Treat seasonality as a confounder."*

Listing the tools does not require a key. A key is checked when `guardian_review` is called,
and each review counts against your plan or trial quota. Requesting an upload link
(`guardian_request_upload`) does not count toward quota.

## Use with agents

Add this rule to your agent's instructions so it checks ad-spend and budget claims
before recommending a change:

> Whenever evaluating marketing campaigns, ad platforms, or budget allocation, verify the
> claimed effect is causal using the Syl Guardian MCP tool (guardian_review) before giving
> a final recommendation.

Ready-to-paste sections for `CLAUDE.md`, `AGENTS.md` and `.cursorrules` are in
[AGENT_RULES.md](AGENT_RULES.md). Also available:

- Rules file: <https://sylai.io/guardian-agent-rules.md>
- Machine-readable summary: <https://sylai.io/llms.txt>
- Docs: <https://sylai.io/guardian/docs>
- Trial key: <https://sylai.io/guardian>

## Python client

For Python agents and scripts, [`syl-guardian-client`](https://pypi.org/project/syl-guardian-client/)
is a thin, MIT-licensed client for the hosted service. It only makes HTTP calls to
`https://guardian.sylai.io`; every number is still computed deterministically by the
service, never by an LLM. Set your key in `SYL_GUARDIAN_API_KEY`.

```bash
pip install syl-guardian-client            # extras: [crewai], [langchain]
```

```python
from syl_guardian_client import GuardianClient

result = GuardianClient().review(
    "ad_spend", "conversion_value",
    data=rows, time_col="date", step=500,
    known_confounders=["brand_query_click_pct"],
)
result.verdict  # "pass" | "caution" | "hold"
```

- PyPI: <https://pypi.org/project/syl-guardian-client/>
- Source: <https://github.com/connor274/syl-guardian-client>

## Tool: `guardian_review`

Sanity-checks a causal claim before an agent acts on it.

| Argument | Description |
|---|---|
| `cause`, `effect` | Required. Column names for the claim. |
| `csv`, `data` or `dataset_id` | Dataset as CSV text, a list of row objects, or the `dataset_id` of a previously uploaded file (see [Large datasets](#large-datasets)). Exactly one. |
| `known_confounders` | Optional. Columns to adjust for; otherwise discovered automatically. |
| `context` | Optional business context, e.g. "Q3 promo evaluation". |
| `units`, `step` | Display-only unit hints; a treatment step to restate the effect in business terms. |
| `time_col`, `adstock`, `se` | Time-series handling: row ordering, carryover effects, standard-error flavor. |
| `test_design` | `true` or options. Also returns how long a spend-cut test needs to run. |
| `cate` | Optional. List of columns to stratify by; also returns subgroup effects. Off by default. |
| `sensitivity` | Optional. `true` also returns sensitivity checks on the estimate. Off by default. |
| `seed` | Optional integer (default 42). Fixes every random draw; the seeds used are returned in `stamp`. |
| `config` | Optional object choosing `rule` or `llm` per step: `cleaning`, `discovery`, `estimator`, `summary`. See [Engine config](#engine-config). |
| `estimator` | Optional. Omit or `auto` (default): defers to `config.estimator`, which defaults to `rule` (deterministic choice). Or force one of `ols_adjusted`, `aipw`, `ipw`, `psm`, `dml_forest`. See [Estimator choice and DAG](#estimator-choice-and-dag). |

The result includes a text summary (verdict, recommended action, effect versus
correlation, interpretation) and the full structured response. Bad input (including an
unknown `config` key or value), auth failures and quota errors come back as MCP tool
errors with a descriptive message. Misspelled argument names are rejected, not ignored.

### Engine config

`config` lets you choose, per step, whether the engine uses fixed rules or an LLM. Each
value is `"rule"` or `"llm"`. Omitted keys use the defaults:

| Step | Default | What it controls |
|---|---|---|
| `cleaning` | `llm` | Dataset cleaning: derived columns and null imputation. The LLM only picks a strategy per column (for example median or drop rows); the values are computed deterministically, and your cause and effect columns are never imputed. |
| `discovery` | `llm` | Pruning and orienting the causal graph. Guardian always receives your cause and effect, so it does not name them. |
| `estimator` | `rule` | Picking the estimator from the supported set, based on each method's documentation. |
| `summary` | `rule` | `rule` is a fixed template for audit trails that explains why: confounder importance, the interval in plain terms (in dollars when units are known), and why this adjustment set and estimator. `llm` is AI-written prose. |

The model key is provided by the service; you never supply one. `rule` always works. If an
`llm` step cannot run, it falls back to `rule`, and the result's `engine_steps` reports
`{requested, used, fallback_reason}` for each step (for example `llm_unavailable`). Setting
all four to `rule` makes no LLM calls.

Example: `{"config": {"cleaning": "rule", "discovery": "rule"}}`.

### Determinism and reproducibility

Under the defaults, the same data and request give the same verdict, estimate and
interval. Randomness uses a fixed seed that is returned. What can change the estimate:

| Setting | Can change the estimate? |
|---|---|
| `cleaning: llm` | Yes: the imputation strategy and which derived columns are dropped. |
| `discovery: llm` | Yes: the adjustment set. |
| `estimator: llm` | Yes: the choice of method. |
| `summary: llm` | No: prose only. |

LLM calls run at temperature 0 with a pinned model, and each decision is recorded in
`llm_decisions` and cached, but providers do not guarantee bit-identical output on a cache
miss or model change, and the cache is in memory only. For strict reproducibility set
`cleaning` and `discovery` to `rule`.

### Estimator choice and DAG

With `estimator` omitted or `"auto"`, the estimator comes from `config.estimator`. The
default is `rule`: a deterministic choice (no confounders: naive OLS; binary treatment:
AIPW; otherwise adjusted OLS). With `config.estimator = "llm"`, an LLM picks one estimator
(plus bounded parameters) from a fixed whitelist, filtered to what your data supports. It
sees only a column schema summary, the DAG edges and the adjustment set, never your rows,
and it never computes a number: the chosen estimator is run deterministically with fixed
seeds. Passing a whitelist id forces that estimator with no LLM involved.

If the LLM is unavailable or returns an invalid or ineligible choice, or a requested
estimator is ineligible for the data or fails, Guardian falls back to the deterministic
rule. This is always reported, never hidden.

Every result includes these fields:

- `estimator`: `id`, `params`, `source`, `rationale` and `fallback_reason`. `source` is
  `rule` (also the value when no LLM is available), `llm`, `override`, or `fallback` (an LLM
  choice or override was attempted but rejected or failed). `rationale` is LLM-written prose and is set only
  for `llm`. `fallback_reason` explains why the requested estimator was not used, otherwise
  `null`.
- `dag`: the inferred causal graph, as `nodes` and `edges` of the form
  `{"source": ..., "target": ...}`. It reflects the cleaned data and any
  `known_confounders`.
- `engine_steps`: requested versus used mode for each step, with any fallback reason.
- `stamp`: a method and version stamp (core version, estimator, methods run, resolved
  config, seeds, models used, a data fingerprint, and the service name and version). If an
  answer changes between two runs, the stamp shows whether the data or a Syl release
  changed.
- `llm_decisions`: the LLM decisions made on this review (empty when no step used an LLM).
- `confounder_importance`: with `summary: rule`, how much the estimate moves when each
  adjustment variable is dropped, as a share of the total movement.
- `cate` and `sensitivity`: present only when requested.

## Tool: `guardian_request_upload`

Returns a one-time upload link for a dataset too large to pass inline.

| Result field | Description |
|---|---|
| `upload_url` | URL to PUT the file to. |
| `dataset_id` | Pass this to `guardian_review` as `dataset_id`. |
| `expires_at` | When the upload link expires. |
| `max_bytes` | Maximum accepted upload size. |

The result also includes a ready-to-run `curl` hint.

## Large datasets

Use inline `csv` or `data` for small data. For more than about 200 rows, use the upload flow:

1. Call `guardian_request_upload`, then upload the file with the returned URL:

   ```bash
   curl -T data.csv "<upload_url>"
   ```

   Gzip is allowed, e.g. `curl -T data.csv.gz -H "Content-Encoding: gzip" "<upload_url>"`.
   The PUT response returns only the schema (columns, dtypes, `n_rows`), never rows.
2. Call `guardian_review` with `dataset_id` instead of `csv` or `data` (they are mutually
   exclusive). Each uploaded dataset is single-use.

Uploaded datasets are visible only to the API key that uploaded them. Each dataset is
deleted as soon as `guardian_review` runs on it, even if the review fails, so a retry needs
a fresh `guardian_request_upload`. Unused uploads are deleted after 30 minutes.

A `dataset_id` can be reviewed once.

Example prompt: *"My data is in sales.csv (50,000 rows). Request an upload link, upload the
file with curl, then use guardian_review once with the returned dataset_id (uploads are
single-use): did ad_spend cause new_customers?"*

## Support

Questions or issues: <https://sylai.io> or connor@sylai.io.

## License

The files in this repository (metadata and documentation) are MIT licensed; see
[LICENSE](LICENSE). The Syl Guardian service is proprietary.
