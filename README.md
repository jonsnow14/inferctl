# inferctl

CLI-first sentiment scorer that grows into a full MLOps stack: package → API → train → track → version → deploy → schedule → monitor.

```text
inferctl predict   --text "..." [--url|--model-id|--model-dir]
inferctl serve     --model-dir|--model-id [--host --port]
inferctl train     --data ... --out models/ [--epochs]
inferctl evaluate  --model-dir ... --split test
inferctl publish   --repo YOU/name
inferctl repro                  # dvc repro
inferctl register  --run-id ... # MLflow registry
inferctl rollback  --stage Production --version N
inferctl status
```

This README is the **8-week, Monday–Friday map**. Each weekday is about **2.5 hours** (~12.5 h/week). **Friday is the exam**: if the gate fails, do not start the next week.

The spine is always `inferctl`. Learn a slice, then put it in the tool the same day.

---

## Timeline

```text
Week 1  CLI + package + predict
Week 2  Flask API + SDK + OpenAPI
Week 3  Real HF train / eval
Week 4  MLflow tracking + registry
Week 5  DVC data + pipelines
Week 6  Docker + Compose deploy
Week 7  Tests, CI, orchestration
Week 8  Monitor + rollback + demo

Optional 9–12  Staging/prod, batch, PEFT/Hub, canary
```

| Horizon | Length | What you have |
|---|---|---|
| Spike | 3.5 days | Toy CLI that predicts |
| **This map** | **8 weeks** | Train → track → version → deploy → schedule → monitor |
| Production-shaped | 12 weeks | Staging/prod, drift, rollback, A/B |

---

## Target stack

```text
git + inferctl CLI
        │
        ▼
   DVC pipeline          prepare → train → evaluate
        │
        ▼
   MLflow                runs, metrics, model registry
        │
        ▼
   Hugging Face Hub      checkpoints + model card
        │
        ▼
   Docker Compose        Flask scorer + MLflow UI
        │
        ▼
   Prefect / GitHub Actions    schedule + CI
        │
        ▼
   Prometheus / Grafana + smoke/rollback
```

---

## How to use Friday every week

1. Run the exam commands **cold** (new terminal, activate venv, no leftover servers).
2. Fill this 5-line log in `notes/week-N.md`:

```text
Gate: PASS / FAIL
What I can now do:
What I still fake:
Link/run_id/screenshot:
Start next week? yes / no (if no: Saturday redo, do not skip ahead)
```

3. If Friday fails, the next week’s Monday is a **redo of the failed gate**, not new material.

---

# Week 1 — CLI on PATH + `predict`

**Week outcome:** `pip install -e . && inferctl predict --text "this is great"` works.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | Shell you will live in: paths, venv, `PATH`, pipes, exit codes | https://missing.csail.mit.edu/ · https://linuxcommand.org/tlcl.php (chs. 1–10) | Repo skeleton | `git init` if needed. Create `src/inferctl/`, `.gitignore`, venv. Confirm `python -c "import sys; print(sys.executable)"` is the venv. |
| **Tue** | CLI parsing + packaging + `console_scripts` | https://docs.python.org/3/howto/argparse.html · https://typer.tiangolo.com/ · https://packaging.python.org/tutorials/packaging-projects/ · https://packaging.python.org/specifications/entry-points/ | Installable CLI with `--help` | `pyproject.toml` with `[project.scripts] inferctl = "inferctl.cli:app"`. Typer (or argparse) group: `predict`, `serve`, `train`, `publish` as stubs. `pip install -e .` then `inferctl --help`. |
| **Wed** | What a Transformer pipeline is: tokenizer → model → postprocess | https://huggingface.co/docs/transformers/en/quicktour · https://huggingface.co/learn/llm-course/en/chapter1/1 · https://huggingface.co/learn/llm-course/en/chapter2/1 | `src/inferctl/predict.py` | Load `distilbert-base-uncased-finetuned-sst-2-english` via `pipeline("sentiment-analysis")`. Function `predict_text(text, model_id) -> dict`. No Flask yet. |
| **Thu** | Logging, exit codes, not `print` | https://docs.python.org/3/library/logging.html · https://docs.python.org/3/library/pathlib.html | Wire `inferctl predict` | `--text`, `--model-id`. JSON to stdout. `logging` to stderr. Exit `2` on bad args, `1` on model/runtime error, `0` on success. |
| **Fri** | **Exam — see below** | — | Freeze Week 1 | Tag `week-1` if the gate passes. |

### Friday exam (Week 1)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e .
inferctl --help
inferctl predict --text "this is great"
inferctl predict          # must fail with exit 2
```

| Pass | Fail |
|---|---|
| `inferctl` is on PATH after editable install | Only works as `python src/...` |
| Happy path prints JSON with `label` + `score` | Raw Python traceback to the user |
| Missing `--text` exits `2` | Exits `0` or `1` on usage error |
| No `print()` in library code | Debug prints mixed into JSON |

---

# Week 2 — Flask API + SDK + OpenAPI

**Week outcome:** CLI in-process and `curl` return the **same JSON**.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | HTTP/REST: methods, status codes, JSON resources, what an API is vs an SDK | https://restfulapi.net/ · https://developer.mozilla.org/en-US/docs/Web/HTTP · https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design · https://academy.postman.com/path/api-beginner | Write the contract first | `openapi.yaml`: `GET /health`, `POST /predict` with request/response schemas. Open in https://editor.swagger.io/ — it must validate. |
| **Tue** | Flask app factory, blueprints, project layout | https://flask.palletsprojects.com/en/stable/tutorial/ · https://flask.palletsprojects.com/en/stable/tutorial/factory/ · https://flask.palletsprojects.com/en/stable/tutorial/views/ · https://flask.palletsprojects.com/en/stable/quickstart/ | `src/inferctl/app.py` | `create_app()`. Blueprint with `/health` only. `flask --app inferctl.app:create_app run` returns `{"status":"ok"}`. |
| **Wed** | Mapping HTTP ↔ your predict function | Same Flask tutorial + MDN status codes | `POST /predict` + `inferctl serve` | Body `{"text":"..."}`. 400 on missing text, 200 + same dict as CLI. `inferctl serve --host 127.0.0.1 --port 8000` starts the factory. |
| **Thu** | SDK design: retries, timeouts, typed errors | https://newsletter.pragmaticengineer.com/p/building-great-sdks · https://auth0.com/blog/guiding-principles-for-building-sdks/ | `src/inferctl/client.py` | `Client(base_url)` with `health()` and `predict(text)`. Timeouts. Retry once on 503. `inferctl predict --url http://127.0.0.1:8000 --text "..."`. |
| **Fri** | **Exam** | — | Contract freeze | Commit `openapi.yaml` + client. |

### Friday exam (Week 2)

```bash
inferctl serve --port 8000 &
curl -s http://127.0.0.1:8000/health
curl -s -X POST http://127.0.0.1:8000/predict -H 'Content-Type: application/json' -d '{"text":"this is great"}'
inferctl predict --text "this is great" > /tmp/cli.json
inferctl predict --url http://127.0.0.1:8000 --text "this is great" > /tmp/http.json
diff /tmp/cli.json /tmp/http.json
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://127.0.0.1:8000/predict -d '{}'
```

| Pass | Fail |
|---|---|
| `diff` is empty (same `label`/`score`/`model_id`) | CLI and HTTP disagree |
| Bad body → **400**, not 500 | Unhandled exception page |
| OpenAPI file opens in Swagger Editor | Spec is comments in README only |
| Client timeout/retry exist as real code | `requests.get` with no timeout |

---

# Week 3 — Real HF train / eval

**Week outcome:** a fine-tune beats the off-the-shelf baseline on a held-out slice.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | Models, tokenizers, datasets, Hub | https://huggingface.co/learn/llm-course/en/chapter1/1 through https://huggingface.co/learn/llm-course/en/chapter2/6 · https://huggingface.co/docs/datasets | `src/inferctl/data.py` | Load a **small** SST-2 slice (e.g. 1k/200 train/val) via `datasets`. Tokenize. Save processed arrows or a local cache path. |
| **Tue** | Fine-tuning with `Trainer` | https://huggingface.co/learn/llm-course/en/chapter3/1 · https://huggingface.co/docs/transformers/en/training | Notebook or script **first**, then port | Fine-tune `distilbert-base-uncased` for 1 epoch on the slice. Confirm `eval_accuracy` is logged. Do not wrap CLI until this runs. |
| **Wed** | Turning a script into a CLI command + reproducibility | https://huggingface.co/docs/accelerate · https://docs.python.org/3/library/logging.html | `inferctl train` | Flags: `--data`, `--out`, `--epochs`, `--model-id`, `--seed`. Write `models/.../config` + weights + `metrics.json`. |
| **Thu** | Evaluation vs prediction; config files | HF course ch. 3 (evaluation) · https://huggingface.co/docs/datasets · TOML via stdlib | `inferctl evaluate` + `train.toml` | `evaluate --model-dir --split test` writes metrics. Train can take `--config train.toml`. Seed is set (`random`, `numpy`, `torch`). |
| **Fri** | **Exam** | — | Champion vs baseline | Compare pipeline baseline vs your `--model-dir`. |

### Friday exam (Week 3)

```bash
inferctl train --data sst2 --out models/week3 --epochs 1 --seed 42
inferctl evaluate --model-dir models/week3 --split validation
inferctl predict --model-dir models/week3 --text "terrible movie"
# baseline (pretrained SST-2 head) vs your run — record both accuracies in notes/week-3.md
```

| Pass | Fail |
|---|---|
| `models/week3` exists and `predict --model-dir` uses **your** weights | Still silently hitting the Hub pretrained model |
| `metrics.json` has accuracy (or F1) | Only loss printed in the terminal |
| Seed flag is wired | Cannot rerun the same experiment |
| Your eval ≥ baseline **or** you write *why* it lost (data too small is acceptable if honest) | “It trained” with no number |

---

# Week 4 — MLflow tracking + registry

**Week outcome:** every train is a run; a champion is registered and servable.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | Why experiment tracking exists; runs vs models vs registry | https://mlflow.org/docs/latest/index.html (Get Started + Tracking) · Zoomcamp Module 2: https://github.com/DataTalksClub/mlops-zoomcamp/tree/main/02-experiment-tracking · https://madewithml.com/courses/mlops/ | Local MLflow | `pip install mlflow` then `mlflow ui`. Log a **dummy** run by hand (`mlflow.start_run`, log param/metric). |
| **Tue** | Logging from training code | MLflow Python API (same docs) · Zoomcamp M2 homework as reference | Instrument `inferctl train` | On each train: log params (epochs, lr, seed, model_id), metrics from `Trainer`, artifacts (`metrics.json`, model dir). `MLFLOW_TRACKING_URI` env. |
| **Wed** | Model registry + stages | https://www.mlflow.org/docs/latest/ml/model-registry/ | `inferctl register` | `--run-id` `--name inferctl-sst2` `--stage Staging`. Use `mlflow.register_model` + transition stage. |
| **Thu** | Serving a registry URI | MLflow Models / pyfunc docs · Week 2 Flask app | `serve` loads `models:/inferctl-sst2/Staging` | `--model-uri models:/inferctl-sst2/Staging`. `/health` reports the URI + version. |
| **Fri** | **Exam** | — | Two runs, one champion | `notes/week-4.md` names the winning `run_id`. |

### Friday exam (Week 4)

```bash
inferctl train --data sst2 --out models/w4a --epochs 1 --seed 1
inferctl train --data sst2 --out models/w4b --epochs 2 --seed 2
mlflow ui   # visually confirm 2 runs
inferctl register --run-id <winner> --name inferctl-sst2 --stage Staging
inferctl serve --model-uri models:/inferctl-sst2/Staging --port 8000
curl -s http://127.0.0.1:8000/health
```

| Pass | Fail |
|---|---|
| UI shows two runs with params **and** metrics | Only local folders, nothing in MLflow |
| Registry has `inferctl-sst2` in Staging | Model copied by hand into Flask |
| `/health` reports registry version | You cannot tell which run is live |
| Train without MLflow still works if URI unset (or fails loudly) | Hard crash with an opaque stack trace |

---

# Week 5 — DVC data + pipelines

**Week outcome:** `dvc repro` is how you train, not a notebook.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | Why data cannot live in git; `.dvc` files, remotes | https://dvc.org/doc/start · https://dvc.org/doc/start/data-management | Init + track raw data | `dvc init`. Put a small raw SST-2 export under `data/raw/`. `dvc add data/raw`. Commit the `.dvc` file, **not** the bytes. |
| **Tue** | Pipeline stages, deps, outs | https://dvc.org/doc/start/data-pipelines | `prepare` stage | `dvc.yaml` stage `prepare` runs `inferctl` (or a small module) and writes `data/processed`. `dvc repro` builds it. |
| **Wed** | Chaining train/eval as stages | Same DVC pipelines page · Zoomcamp if the current cohort covers DVC | `train` + `evaluate` stages | `train` deps on processed data; outs `models/dvc` + metrics. `evaluate` deps on the model. Commands must call **`inferctl`**, not hidden scripts. |
| **Thu** | Metrics, plots, remotes | https://dvc.org/doc/user-guide/experiment-management/comparing-experiments · DVC remotes in the Start guide | `dvc metrics` + local remote | Declare metrics in `dvc.yaml`. `dvc remote add -d local .dvc-remote` (or `/tmp/dvc-remote`). `dvc push` / `dvc pull`. |
| **Fri** | **Exam** | — | `inferctl repro` | Thin wrapper around `dvc repro`. |

### Friday exam (Week 5)

```bash
dvc status
dvc repro
# change one processed-data input or a train hyperparam in params.yaml
dvc repro          # must skip unchanged stages, rerun downstream
dvc metrics show
dvc push
inferctl repro
```

| Pass | Fail |
|---|---|
| `data/raw` is **not** in git history | Dataset committed to git |
| Second `dvc repro` after a param change retrains and skips `prepare` if data unchanged | Everything always reruns, or nothing does |
| `dvc metrics show` prints accuracy | Metrics only in MLflow or a screenshot |
| Fresh `dvc pull` after deleting `data/` restores files | Pipeline only works on your laptop’s cache |

---

# Week 6 — Docker + Compose

**Week outcome:** a stranger can `docker compose up` and hit `/predict`.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | Images vs containers, layers, `WORKDIR`, ports | https://docs.docker.com/get-started/ | Run someone else’s image | `docker run --rm -p 8000:8000` a tiny hello image so Docker itself is not the blocker. |
| **Tue** | Writing a service Dockerfile | https://docs.docker.com/get-started/workshop/02_our_app/ · https://flask.palletsprojects.com/en/stable/tutorial/deploy/ | `Dockerfile` for **serve** | Install from `pyproject.toml`. `CMD` = `inferctl serve --host 0.0.0.0 --port 8000`. `.dockerignore`. Non-root user. |
| **Wed** | Compose services + volumes | https://docs.docker.com/compose/ | `compose.yaml` | Services: `api` (your image), `mlflow` (`mlflow ui` or server), named volume for artifacts. `docker compose up --build`. |
| **Thu** | Training in a container + version labels | Docker docs on `docker compose run` | `compose run train` | One-shot train service writes into the shared volume. Label image with git sha / model version. `/health` returns that version. |
| **Fri** | **Exam** | — | Clean-room boot | Remove local venv from the test path. |

### Friday exam (Week 6)

```bash
docker compose down -v
docker compose up --build -d
curl -sf http://127.0.0.1:8000/health
inferctl predict --url http://127.0.0.1:8000 --text "this is great"
# from a second terminal: compose logs show no traceback
```

| Pass | Fail |
|---|---|
| Predict works **without** `pip install -e .` on the host | Container imports host site-packages |
| `/health` includes model/image version | “ok” with no identity |
| Image rebuild is deterministic enough to document | “It works on my Docker Desktop” only |
| `api` can read the model volume | Model baked in with no way to swap |

---

# Week 7 — Tests, CI, orchestration

**Week outcome:** a breaking PR cannot merge; training can be scheduled.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | Testing CLIs and Flask apps | https://docs.pytest.org/en/stable/how-to/ · https://flask.palletsprojects.com/en/stable/tutorial/tests/ · https://learn.scientific-python.org/development/ | `tests/` | `pytest`: parse CLI help; `POST /predict` via Flask test client; one golden JSON fixture (tiny fake/stubbed pipeline if needed so CI is CPU-light). |
| **Tue** | CI as a gate | https://docs.github.com/en/actions/get-started/quickstart · Zoomcamp CI bits | `.github/workflows/ci.yml` | On PR: lint (`ruff`), `pytest`, optional OpenAPI parse. No GPU. Fixture data only. |
| **Wed** | Orchestration vs cron vs CI | https://docs.prefect.io/ · Zoomcamp orchestration module in https://github.com/DataTalksClub/mlops-zoomcamp | Prefect hello | Install Prefect. A flow that prints and calls `inferctl evaluate` on an existing model. Run once locally. |
| **Thu** | Promotion policy in a flow | Prefect docs: tasks, retries · Week 4 registry | Flow: `repro → read metrics → register if acc > threshold` | Threshold in config. If below, flow **succeeds** but does not promote (do not fail the world on a bad model). |
| **Fri** | **Exam** | — | Red-green CI + one flow run | Break a test on a branch and watch CI fail. |

### Friday exam (Week 7)

```bash
pytest -q
# push a branch that deletes an assertion or breaks /health
# CI must go red
# run the promote-or-skip Prefect flow once
```

| Pass | Fail |
|---|---|
| `pytest` green on a clean tree | Only manual clicking |
| Intentional break → CI red | CI is `echo hello` |
| Flow registers **only** when metric beats threshold | Always registers, or always crashes |
| CI does not download a 400MB model every PR | Jobs timeout / burn the free minutes |

---

# Week 8 — Monitor, rollback, demo

**Week outcome:** you can promote, serve, watch, and undo. This is the finish line.

| Day | Learn | From | Implement | How |
|---|---|---|---|---|
| **Mon** | SLIs for a model service: latency, errors, throughput | https://huyenchip.com/mlops/ · Zoomcamp monitoring module · https://prometheus.io/docs/instrumenting/exposition_formats/ | `GET /metrics` | Counters: requests, errors. Histogram: latency. Use `prometheus_flask_exporter` or a 30-line manual exporter. |
| **Tue** | Dashboards | Grafana first-dashboard docs **or** a static HTML page hitting `/metrics` | One dashboard | Three panels: QPS, p95, error rate. Docker Compose service optional. |
| **Wed** | Prediction logs + cheap drift | Chip Huyen monitoring notes · https://madewithml.com/courses/mlops/ | Log each request | Append JSONL: timestamp, text length, predicted label, score. On-demand job: if mean length or label mix shifts past a threshold, print `DRIFT`. |
| **Thu** | Rollback as a first-class op | MLflow registry stage transitions | `inferctl rollback` + `inferctl status` | `rollback --name inferctl-sst2 --stage Production --version N`. `status` prints live URI, registry version, last DVC metric, `/health`. |
| **Fri** | **Exam — full stack demo** | — | One script, ~4 minutes | Record commands + screenshot of MLflow + dashboard in `notes/week-8.md`. |

### Friday exam (Week 8) — the only exam that matters

```bash
git pull && dvc pull
inferctl repro
inferctl register --run-id <latest> --name inferctl-sst2 --stage Production
docker compose up -d --build
inferctl predict --url http://127.0.0.1:8000 --text "this is great"
curl -s http://127.0.0.1:8000/metrics | head
inferctl rollback --name inferctl-sst2 --stage Production --version 1
inferctl status
```

| Pass | Fail |
|---|---|
| Script runs without hand-editing files mid-way | “Then I opened a notebook…” |
| `/metrics` moves when you send 20 requests | Empty endpoint |
| Rollback changes what `/health` reports | Rollback is `git checkout` of weights |
| README / notes can be followed in a **new** venv/compose | Works only in the original shell session |

---

## What not to do when

| Temptation | Wait until |
|---|---|
| Kubeflow / K8s | Week 12, if ever |
| Feature store (Feast) | After week 8, only if you have real features |
| W&B instead of MLflow | Fine, but pick **one** tracker |
| Full Flask Mega-Tutorial auth/DB | Never required for this scorer |
| Giant LLM training | Never on this roadmap; use a classification / small LM |
| Perfect SDK generation | Week 2 handwritten client is enough |

---

## Weekly checkpoints

| End of | Must be true |
|---|---|
| W1 | `pip install -e . && inferctl predict --text "ok"` |
| W2 | CLI and `curl` return the same JSON |
| W3 | Fine-tuned weights beat the baseline on a held-out slice (or you document why not) |
| W4 | Champion is in the MLflow registry |
| W5 | `dvc repro` is how you train, not a notebook |
| W6 | Stranger can `docker compose up` and hit `/predict` |
| W7 | A PR that breaks tests cannot merge |
| W8 | You can promote, serve, watch, and roll back |

---

## Optional weeks 9–12

Only after Week 8 is boringly reliable.

| Week | Mon learn | Tue–Thu implement | Friday exam |
|---|---|---|---|
| **9** Staging vs Production | https://huyenchip.com/mlops/ + MLflow aliases | Two Compose files / two registry stages; human approval before Production | Cannot promote to prod from laptop without an explicit `--yes` + stage check |
| **10** Batch vs online | HF Datasets + your client | `inferctl batch --input data.csv --output preds.csv` | 1k-row CSV scored offline; numbers match online `/predict` on 5 sampled rows |
| **11** PEFT/LoRA + Hub registry | https://huggingface.co/docs/peft · https://huggingface.co/docs/huggingface_hub/guides/cli · https://huggingface.co/docs/trl | LoRA train on a slightly larger model; `inferctl publish --repo YOU/inferctl-lora` | `predict --model-id YOU/inferctl-lora` works on a clean machine after `huggingface-cli download` |
| **12** Canary / optional K8s | Huyen serving + Docker networking | 90/10 traffic split between two `--model-uri`s **or** a single k8s Deployment | Kill canary; error rate stays flat; rollback still works |

---

## How this maps to the five learning phases

| Phase | Topics | Weeks | Outcome |
|---|---|---|---|
| 1. Foundation | CLI + shell automation | 1 | You live in the terminal; `inferctl` is on PATH |
| 2. Interfaces | APIs → SDKs | 2 | You consume and wrap your own service |
| 3. Python CLIs | argparse → automation → packaging | 1–2 | You ship `inferctl` |
| 4. Web | Flask | 2 + 6 | You serve `/predict` locally and in Docker |
| 5. ML | HF Transformers → full MLOps | 3–8 (9–12 optional) | You train, track, version, deploy, schedule, monitor |

---

## Resource index

| Topic | URL |
|---|---|
| Missing Semester | https://missing.csail.mit.edu/ |
| Linux Command Line (free PDF) | https://linuxcommand.org/tlcl.php |
| argparse tutorial | https://docs.python.org/3/howto/argparse.html |
| Typer | https://typer.tiangolo.com/ |
| PyPA packaging | https://packaging.python.org/tutorials/packaging-projects/ |
| Entry points | https://packaging.python.org/specifications/entry-points/ |
| logging | https://docs.python.org/3/library/logging.html |
| pathlib | https://docs.python.org/3/library/pathlib.html |
| REST | https://restfulapi.net/ |
| MDN HTTP | https://developer.mozilla.org/en-US/docs/Web/HTTP |
| Microsoft API design | https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design |
| Postman Academy | https://academy.postman.com/path/api-beginner |
| OpenAPI | https://spec.openapis.org/oas/latest.html |
| Swagger Editor | https://editor.swagger.io/ |
| Flask tutorial | https://flask.palletsprojects.com/en/stable/tutorial/ |
| Flask factory | https://flask.palletsprojects.com/en/stable/tutorial/factory/ |
| Flask views | https://flask.palletsprojects.com/en/stable/tutorial/views/ |
| Flask tests | https://flask.palletsprojects.com/en/stable/tutorial/tests/ |
| Flask deploy | https://flask.palletsprojects.com/en/stable/tutorial/deploy/ |
| Flask quickstart | https://flask.palletsprojects.com/en/stable/quickstart/ |
| Building great SDKs | https://newsletter.pragmaticengineer.com/p/building-great-sdks |
| Auth0 SDK principles | https://auth0.com/blog/guiding-principles-for-building-sdks/ |
| HF LLM Course ch.1 | https://huggingface.co/learn/llm-course/en/chapter1/1 |
| HF LLM Course ch.2 | https://huggingface.co/learn/llm-course/en/chapter2/1 |
| HF LLM Course ch.3 | https://huggingface.co/learn/llm-course/en/chapter3/1 |
| Transformers quicktour | https://huggingface.co/docs/transformers/en/quicktour |
| Transformers training | https://huggingface.co/docs/transformers/en/training |
| Datasets | https://huggingface.co/docs/datasets |
| Accelerate | https://huggingface.co/docs/accelerate |
| PEFT | https://huggingface.co/docs/peft |
| TRL | https://huggingface.co/docs/trl |
| huggingface-cli | https://huggingface.co/docs/huggingface_hub/guides/cli |
| MLflow | https://mlflow.org/docs/latest/index.html |
| MLflow Model Registry | https://www.mlflow.org/docs/latest/ml/model-registry/ |
| MLOps Zoomcamp | https://github.com/DataTalksClub/mlops-zoomcamp |
| Zoomcamp M2 | https://github.com/DataTalksClub/mlops-zoomcamp/tree/main/02-experiment-tracking |
| Made With ML | https://madewithml.com/courses/mlops/ |
| Chip Huyen MLOps | https://huyenchip.com/mlops/ |
| DVC start | https://dvc.org/doc/start |
| DVC data | https://dvc.org/doc/start/data-management |
| DVC pipelines | https://dvc.org/doc/start/data-pipelines |
| Docker get started | https://docs.docker.com/get-started/ |
| Docker Compose | https://docs.docker.com/compose/ |
| pytest | https://docs.pytest.org/en/stable/how-to/ |
| GitHub Actions quickstart | https://docs.github.com/en/actions/get-started/quickstart |
| Scientific Python dev | https://learn.scientific-python.org/development/ |
| Prefect | https://docs.prefect.io/ |
| Prometheus exposition format | https://prometheus.io/docs/instrumenting/exposition_formats/ |

---

## Weekly notes

Copy this into `notes/week-N.md` every Friday:

```text
Gate: PASS / FAIL
What I can now do:
What I still fake:
Link/run_id/screenshot:
Start next week? yes / no
```
