# TDS Desk: classify-then-compute for vendor invoices (India)

Prototype for the Tresata take-home. An accounts-payable team at an Indian company must decide, for every vendor bill, **which TDS (tax deducted at source) entry applies, at what rate, on what base**, before paying. This prototype splits that decision in two:

* an **LLM classifies the nature of each invoice line** into a closed set of categories and quotes the exact words that justify it;
* a **deterministic engine** applies versioned rule tables (thresholds, rates, aggregates, no-PAN rate, GST exclusion) and cites the section;
* a **router** sends anything uncertain, contradictory or high-value to a qualified person.

The model never sees a rate and never outputs a rupee amount.

​```
invoice text ──► preprocess (regex: PAN, payee type from PAN, GSTIN, date, GST shown?, line items)
                    │
                    ▼
               classifier (LLM or keyword baseline) ──► verifier (amounts match? quotes verbatim? category allowed?)
                    │                                                │
                    ▼                                                ▼
               engine (rules/tds_rules.json, chosen by invoice date; vendor ledger; buyer profile)
                    │
                    ▼
               router ──► auto-post with trace   |   review queue with reasons
​```

## Run it

Python 3.10+, no third-party packages.

​```bash
python data/make_invoices.py          # (re)generate the 15 synthetic invoices + labels
python run_eval.py --mode gold        # engine/verifier/router with hand-labelled categories
python run_eval.py --mode baseline    # keyword classifier (the non-AI approach) + same engine
export OPENROUTER_API_KEY=sk-or-...   # free key from openrouter.ai
export OPENROUTER_MODEL=meta-llama/llama-3.3-70b-instruct:free   # any ':free' model on openrouter.ai/models
python run_eval.py --mode llm --sleep 4         # LLM classifier + same engine
python run_eval.py --mode llm_direct --sleep 4  # rejected design: LLM asked for the TDS amount itself
​```

Reports are written to `results/<mode>.md` and `results/<mode>.json` (the JSON keeps each invoice's full trace and the raw model output).

## What is in here

| Path | What it is |
|---|---|
| `tds/preprocess.py` | Exact, regex-only facts. Payee type comes from the PAN's 4th character (P individual, C company, F firm/LLP, H HUF…), not from the model. |
| `tds/classifiers.py` | Keyword baseline, LLM classifier (prompt included), and the end-to-end "llm_direct" comparison. |
| `tds/verify_route.py` | Verifier and routing rules (confidence < 0.75, alternative category, glossary trap, reimbursement, declarations, TDS > ₹50,000). |
| `tds/engine.py` | Threshold logic: single-or-aggregate (contracts), aggregate with catch-up, per-month (rent), excess-over (goods). |
| `rules/tds_rules.json` | Two rule sets: Income-tax Act 1961 as amended by Finance Act 2025 (to 31-Mar-2026) and Income-tax Act 2025 (from 1-Apr-2026, s.393 tables). Each rule carries its citation; `last_verified` is empty on purpose. |
| `rules/glossary.json` | Known classification traps kept in code with their source, e.g. "technical consultancy" is a profession (10%, not 2%); an advertising agency's "commission" is a contract payment. |
| `data/` | 15 **synthetic** invoices, expected answers with hand working (`gold.json`), a vendor ledger and buyer profile. All names, PANs and GSTINs are invented (PANs start `ZZZ`). |

## Test set (15 made-up invoices)

Each one targets a specific way the decision goes wrong:

| # | Case | Expected |
|---|---|---|
| 01 | CA firm audit fee, GST shown | professional 10% on pre-GST value, ₹35,000 |
| 02 | Housekeeping / manpower supply | contract 2%, ₹3,600 |
| 03 | IT helpdesk + server monitoring | technical 2%, ₹1,800, **review** (arguably contract) |
| 04 | Office rent ₹1.2 lakh/month | rent building 10%, ₹12,000 |
| 05 | DG set "hire charges" (no word "rent") | rent of machinery 2%, ₹1,500 |
| 06 | Sales commission | 2%, ₹920 |
| 07 | Architect, ₹40,000 bill, ₹30,000 paid earlier | aggregate crosses ₹50,000 → 10% on ₹70,000 = ₹7,000 |
| 08 | Transporter with ≤10-truck declaration | nil, **review** to confirm declaration on file |
| 09 | Event company: management + banquet hall | ₹4,800 + ₹6,000, **review** (hall may be part of contract) |
| 10 | Lawyer, no PAN | 20%, ₹12,000 |
| 11 | ₹26 lakh ingots, ₹40 lakh bought earlier | 0.1% on ₹16 lakh above ₹50 lakh = ₹1,600 |
| 12 | Consulting fee + reimbursed travel | 10% on gross ₹1,88,000, **review** |
| 13 | "Technical consultancy" | professional 10%, ₹8,500 (trap: not 2%) |
| 14 | Ad agency bill with "agency commission" | contract 2% on ₹5 lakh = ₹10,000 (trap: not commission) |
| 15 | March 2026 invoice | old-Act rule set, cited as 194J(b), ₹8,000 |

## Results so far (honest)

| Mode | Categories right | TDS right | Route right | Wrong **and** auto-posted |
|---|---|---|---|---|
| gold (hand labels) | 15/15 | 15/15 | 15/15 | none |
| baseline (keywords) | 10/15 | 9/15 | 10/15 | INV08, INV15 |
| llm | **not run here** | | | |
| llm_direct | **not run here** | | | |

* **Gold mode only proves the deterministic half**: given correct categories, the engine reproduces my hand calculations, including catch-up on aggregates, the 50-lakh excess rule, the no-PAN rate and the 1961→2025 Act switch by invoice date. It says nothing about the AI.
* **The keyword baseline fails exactly where the brief predicts.** "Release of print advertisements" matched `lease` and became rent. "Return filing support" matched `support` and became a 2% technical fee (wrong by ₹6,400 and auto-posted). "Hire charges – DG set" matched nothing. The transporter declaration was invisible to it, so it deducted ₹1,400 that should have been nil, and auto-posted it. The glossary traps caught INV13 and INV14 and sent them to review, which is the safety net working, not the classifier.
* **The LLM modes were not executed in this build environment**: outbound calls to openrouter.ai are blocked there. The code path is complete; run `--mode llm` with a free key and the results table fills in. Until then I make no claim about LLM accuracy.

## Known weaknesses

* Line items are parsed with a regex that expects `N. description | amount`. Real invoices are PDFs/scans with varied layouts; an OCR/layout step would be needed and is the most likely real-world failure point.
* The rule tables were assembled from secondary sources (tax portals) and the Income Tax Department section page, not verified by a chartered accountant. `last_verified` is null until one does.
* Glossary positions (Circular 715, Circular 23/2017, notified professions) are 1961-Act era; their carry-over to the 2025 Act is flagged, not assumed.
* Out of scope: payments to non-residents (s.393(2) / DTAA), salary TDS, lower-deduction certificates, TDS return filing.

## Model used

Configured default: `meta-llama/llama-3.3-70b-instruct:free` via OpenRouter, temperature 0. Free-model names change; set `OPENROUTER_MODEL` to any current `:free` model and record which one you ran in this section.
