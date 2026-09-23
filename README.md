# PaidFair v2

**Is this salary offer fair?** PaidFair v2 answers with a verdict that a jury of GenLayer validators, each running a different AI model, has to agree on before it is stored on-chain.

> Informational only. PaidFair is not legal, tax or career advice.

## What it does

| | |
|---|---|
| **Input** | role, location, experience, annual gross salary, currency. Optional: one public source URL |
| **Output** | `UNDERPAID`, `FAIR`, `WELLPAID` or `INSUFFICIENT_DATA`, stored as a numbered check `PF-YYYY-NNNN` |
| **Transparency** | every check records its basis (`MODEL_KNOWLEDGE` or `SOURCE`) and, if a source was given, whether source and models agree |

## Why GenLayer

Whether a salary is fair is a judgment call, not a number a classic oracle can deliver. A single AI model would simply pass on its own bias. On GenLayer, validators running different models must independently arrive at the same answer, otherwise nothing is stored.

## How validators reach agreement

Every non-deterministic step returns exactly **one token**, and `gl.eq_principle.strict_eq` compares that token across validators. Everything else is deterministic Python.

### 1. Model jury (always runs)

1. Each validator's model estimates the typical **median** annual gross salary for the role, experience level and location, in the given currency. **The offered salary is not shown to the model**, so the estimate is not anchored to the offer.
2. Python inside the non-deterministic block parses that number and classifies the offer with integer arithmetic:
   - `BELOW` if the offer is under 75% of the median
   - `ABOVE` if the offer is over 125% of the median
   - `WITHIN` otherwise
   - `NO_DATA` if the model gives no usable estimate
3. Validators compare only this token. The models do not need to agree on an exact number, only on the band the offer falls into.

### 2. Optional source (runs only if `source_url` is set)

- Every validator fetches the page itself with `gl.nondet.web.render(url, mode="text")`. Nothing the submitter claims about the page is trusted.
- The model reads the figures on the page and answers `BELOW`, `WITHIN`, `ABOVE` or `NO_DATA` under fixed rules (same currency only, monthly figures x12, range bounds or +/-10% around a single typical figure).
- A failed or empty fetch returns `UNREACHABLE`. Page content is treated as untrusted data, and the prompt instructs the model to ignore instructions inside it.

### 3. Deterministic decision

| Situation | Verdict source | `basis` |
|---|---|---|
| Source finding is `BELOW` / `WITHIN` / `ABOVE` | source | `SOURCE` |
| No usable source, model finding usable | model jury | `MODEL_KNOWLEDGE` |
| Neither usable | none | `NONE`, verdict `INSUFFICIENT_DATA` |

Mapping: `BELOW` -> `UNDERPAID`, `WITHIN` -> `FAIR`, `ABOVE` -> `WELLPAID`.

## Contract

| | |
|---|---|
| **Source** | [`Contract.py`](Contract.py) (repository root) |
| **SDK header** | `# v0.3.0` |
| **Network** | [NETWORK] |
| **Address** | `[CONTRACT_ADDRESS]` |
| **Explorer** | https://explorer-studio-dev.genlayer.com/address/[CONTRACT_ADDRESS] |

### Write method

`submit_check(role, location, experience, annual_gross, currency, source_url, year, request_id) -> str`

| Parameter | Rule | Example |
|---|---|---|
| `role` | 2-100 characters | `Junior Data Analyst` |
| `location` | 2-80 characters | `Berlin, Germany` |
| `experience` | 1-40 characters | `0-2 years` |
| `annual_gross` | whole number, no separators | `48000` |
| `currency` | 3-letter code | `EUR` |
| `source_url` | empty, or one `https://` URL | *(empty)* |
| `year` | 4 digits | `2026` |
| `request_id` | 8-64 chars `[A-Za-z0-9-]`, unique | `pf-test-0001` |

Returns the check ID, for example `PF-2026-0001`.

### View methods

| Method | Returns |
|---|---|
| `get_check(check_id)` | the stored record as JSON, or `""` |
| `get_by_request(request_id)` | the record created by this request, or `""` |
| `list_checks(role_filter)` | up to 50 newest records, filtered by role (empty string = all) |
| `check_count()` | number of checks |

Clients and agents should poll `get_by_request` with their own `request_id`. That way a client can never receive an older check by mistake.

### Stored record (shape)

```json
{
  "annual_gross": 48000,
  "basis": "MODEL_KNOWLEDGE",
  "currency": "EUR",
  "experience": "0-2 years",
  "id": "PF-2026-0001",
  "location": "Berlin, Germany",
  "model_finding": "WITHIN",
  "request_id": "pf-test-0001",
  "role": "Junior Data Analyst",
  "seq": 1,
  "source_and_model_agree": "N/A",
  "source_finding": "",
  "source_url": "",
  "verdict": "FAIR",
  "year": "2026"
}
```

## Test it in GenLayer Studio

1. Open GenLayer Studio, load `Contract.py` and deploy it (no constructor arguments).
2. Call `submit_check` with the base values from the parameter table above, then change only what each row says:

| # | Change | Expected |
|---|---|---|
| 1 | none (`48000`) | `FAIR` |
| 2 | `annual_gross` = `20000`, `request_id` = `pf-test-0002` | `UNDERPAID` |
| 3 | `annual_gross` = `150000`, `request_id` = `pf-test-0003` | `WELLPAID` |
| 4 | `role` = `Xyzqwerty`, `request_id` = `pf-test-0004` | `INSUFFICIENT_DATA` (likely; a model may still guess a number) |
| 5 | `source_url` = raw URL of [`demo-salary-a.md`](demo-salary-a.md), `request_id` = `pf-test-0005` | `FAIR`, basis `SOURCE` |
| 6 | `source_url` = `https://diese-domain-existiert-nicht-xyz123.com`, `request_id` = `pf-test-0006` | basis `MODEL_KNOWLEDGE` |
| 7 | repeat `request_id` = `pf-test-0001` | rejected: `request_id already used` |

3. Read results with `get_by_request`.

A test only counts as passed if the consensus history does **not** end in `UNDETERMINED`.

[`demo-salary-a.md`](demo-salary-a.md) contains **synthetic** figures (Junior Data Analyst, Berlin: 42,000 to 52,000 EUR). It exists only to demonstrate the source path with a known expected result.

## Test evidence

| # | Transaction hash | Consensus | Verdict |
|---|---|---|---|
| 1 | `[TX_HASH]` | [RESULT] | [VERDICT] |
| 2 | `[TX_HASH]` | [RESULT] | [VERDICT] |
| 3 | `[TX_HASH]` | [RESULT] | [VERDICT] |
| 5 | `[TX_HASH]` | [RESULT] | [VERDICT] |

## Limitations

- **Model knowledge is dated.** `MODEL_KNOWLEDGE` verdicts rely on training data with a cutoff and may not reflect current pay.
- **The +/-25% band is a deliberate trade-off.** It makes agreement between different models likely, but only clear deviations are reported as `UNDERPAID` or `WELLPAID`.
- **Consensus can still fail.** If the models' median estimates differ by more than the band and the offer falls in between, the transaction can end `UNDETERMINED`.
- **The submitter chooses the source.** A single hand-picked page can bias a `SOURCE` verdict. The record always shows the URL and whether the models agree.
- **No currency conversion.** Sources in another currency return `NO_DATA`.
- **Monthly figures are multiplied by 12.** Extra monthly salaries or bonuses are not considered.
- **Some pages cannot be read.** Bot-protected or login-only pages return `UNREACHABLE`.
- **No timestamp field.** The time of a check is visible from its transaction in the explorer.

## Privacy

Everything stored by this contract is public and permanent. **Do not enter names, employers or any other identifying details.** The record contains no wallet address. The transaction sender stays visible on-chain as with any transaction.

## What changed from v1

| | v1 | v2 |
|---|---|---|
| Judgment | one model prompt returning a large JSON with salary figures | one number per validator, classified by deterministic Python |
| Consensus | `prompt_comparative` on the tier | `strict_eq` on a single token |
| Evidence | none | optional source fetched by every validator, recorded in the check |
| Storage | one result per wallet, overwritten by each new analysis | one record per `request_id`, numbered `PF-YYYY-NNNN` |
| Anchoring | offered salary shown to the model | offered salary hidden from the model |

## License

Released under the MIT License.
