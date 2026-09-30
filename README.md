
# HealthGrid AI

Disease burden forecasting, medicine demand estimation and procurement planning
for India's public health network. Built on the HMIS sub-district indicator
extract, deployable on Google Cloud.

This is a regeneration of an earlier archive. The review that produced it is in
[ANALYSIS.md](ANALYSIS.md) — read that first if you worked on the previous
version, because five correctness defects were fixed and two of them changed
every number the dashboard displayed.

## Run it

```bash
pip install -r requirements-dev.txt
streamlit run frontend/app.py          # dashboard on :8501
uvicorn backend.main:app --reload      # API on :8000, docs at /docs
```

The dataset is committed at 1.4 MB, so there is nothing to download or build
first. Prophet is optional; without it the engine computes a trend-and-seasonal
baseline and says so in the interface.

## What it does

**Forecasting.** 286 indicators, 36 states and union territories, 263,978
observed state-months from January 2017 to May 2021, projected twelve months
ahead per state and indicator.

**Medicine demand.** Fourteen disease programmes have a medicine and a per-case
dosage reviewed against Indian standard treatment guidelines. Projected cases
times dosage plus a 30% safety buffer gives a required quantity; comparing that
against stock gives a shortfall and a priority tier.

**Procurement planning.** Shortfalls are sequenced into order-now, order-this-
quarter and monitor tiers. Three Vertex AI agents write the plan when CrewAI and
credentials are available; a deterministic heuristic planner writes it otherwise.
The response always says which one did.

**Clinical surfaces.** Referral ranking, the multilingual patient assistant and
preliminary imaging reads are demonstration fixtures. Every response carries
`demonstration: true` and the interface labels each screen, because a committee
shown a live-looking panel will assume it is live.

## Honest status of every number on screen

| Shown | Real | Estimated | Fixture |
|---|---|---|---|
| Observed monthly cases | ✓ (HMIS extract) | | |
| Twelve-month projection | ✓ (method is stated) | | |
| Per-case medicine dosage | ✓ (14 programmes reviewed) | | |
| Required quantity | ✓ (derived) | | |
| Stock on hand | | ✓ (deterministic estimate) | |
| Shortfall and priority | | ✓ (follows from stock) | |
| Procurement plan | ✓ (agents or heuristic) | | |
| Referral ranking weights | ✓ | | |
| Referral facilities and beds | | | ✓ |
| Assistant refusal rules | ✓ | | |
| Assistant answer text | | | ✓ |
| Imaging findings | | | ✓ |

Stock is the important row. Until a DVDMS or e-Aushadhi feed is connected, every
deficit on the medicine demand tab is derived from an estimated stock position.
It is deterministic and stable between refreshes, but it is not a warehouse
reading, and no purchase order should be raised against it unverified.

## Layout

```
backend/forecasting.py     Engine: history, projection, medicine demand, shortfalls
backend/procurement.py     Agent and heuristic planners
backend/clinical.py        Referral, assistant, imaging fixtures; real refusal rules
backend/main.py            FastAPI application
backend/common/            GCP integration: FHIR, ABDM, Vertex AI, config, logging
frontend/app.py            Streamlit dashboard
scripts/build_dataset.py   411 MB raw CSV -> 1.4 MB committed dataset
scripts/train_prophet.py   Optional Prophet training
data/                      The committed dataset, documented in data/README.md
tests/                     Regression tests, one per defect fixed
```

`backend/common/` is the production GCP integration path — FHIR reads with a
consent gate, ABDM consent artefacts, Vertex AI with schema-validated output.
The unified server does not use it yet. It is kept because it is where the
clinical endpoints go when they stop being fixtures, not because it is wired in.

## Rebuilding the dataset

The committed file is reproducible, not an opaque artefact:

```bash
python scripts/build_dataset.py --source /path/to/major-health-indicators-subdistrict-level.csv
```

Keep the 411 MB source outside the repository. Nothing at runtime reads below
state level.

## Optional Prophet forecasts

```bash
pip install prophet joblib
python scripts/train_prophet.py              # 14 curated indicators
python scripts/train_prophet.py --all        # all 286, much slower
```

Writes `data/prophet_forecasts.csv.gz`. The engine picks it up on next start and
the sidebar switches from a baseline warning to a trained-model confirmation.

Prophet is deliberately excluded from the container image: it pulls a compiler
toolchain and cmdstan for roughly a gigabyte, and training is an offline job
rather than a request path. Train outside the container and mount the result.

## Tests

```bash
pytest tests -q
```

Each test names the defect it guards against, so a change that reintroduces one
fails with an explanation. They cover the date reconstruction, state-code
mapping, forecast determinism, the single-variable filter, and that every mapped
indicator exists in the data.

## Deploying

```bash
make image
make deploy PROJECT=my-project
```

One container runs both processes, with Streamlit in the foreground. Set
`ALLOWED_ORIGINS` to the dashboard host; the default allows only localhost.

For the full Google Cloud footprint — Healthcare API FHIR and DICOM stores,
customer-managed encryption, ABDM consent enforcement, VPC Service Controls —
see the reference architecture deck and the GCP repository this prototype was
extracted from.

## Before this informs a real procurement decision

- Connect DVDMS or e-Aushadhi for actual stock positions.
- Have a public health physician review the fourteen per-case dosages.
- Train Prophet and compare against the baseline on held-out months.
- Extend the extract beyond May 2021; a projection from four-year-old data
  describes 2021, not today.
- Get the projections validated against a district the team knows well before
  trusting them for one they do not.
