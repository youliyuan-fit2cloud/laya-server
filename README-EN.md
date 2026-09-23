# Laya Server

A standalone Laya System One HTTP service with a single-admin console. The backend uses FastAPI + SQLite; the frontend uses React + Vite + TypeScript with shadcn/ui. After building, a single application container serves the web pages and API.

# Upstream source

This repository does not track the Laya source. Before building, check out the fixed v0.3.7 version in the repository root:

```sh
git clone https://github.com/NandhaKishorM/laya.git laya
git -C laya checkout --detach 010bacef009c855ccba814b51f7c8e1d38ab5e3f
sh scripts/check-upstream.sh
```

The 'laya/' directory is added to'.gitignore 'but will be included in the Docker build context. The Dockerfile verifies the full SHA and a clean working tree; the final image contains only the upstream packages and licenses required to run, not the .git history.

# Admin and configuration
Set LAYA_ADMIN_USERNAME and a LAYA_ADMIN_PASSWORD of at least 10 characters in .env; on service startup an Argon2id hash is generated in memory for login verification. Alternatively, you can avoid storing a plaintext password by generating LAYA_ADMIN_PASSWORD_HASH with .venv/bin/python scripts/hash-password.py; you must set exactly one of these two. Wrap the hash value in single quotes to ensure Docker Compose preserves $ literally. LAYA_PUBLIC_ORIGIN must also be set; in production this must be an HTTPS origin, e.g. https://console.example.com. Local HTTP testing requires LAYA_ALLOW_INSECURE_LOCAL=1. .env is gitignored—do not commit real passwords.

```sh
cp .env.example .env
python3.12 -m venv .venv
.venv/bin/python -m pip install -e 'backend[test]'
# edit .env，input LAYA_ADMIN_USERNAME and LAYA_ADMIN_PASSWORD
```

# Model files
Model weights are not distributed with the repository or images. scripts/download-models.py pins the Hugging Face repo to commit 1c5edc17a7acd8701df6fc341c0d179f1c62c982 and places three checkpoints into the persistent model volume at /models/english, /models/multilingual, and /models/typed-decisions. Each directory must contain at least the upstream model package’s rl_agent_config.json, model.safetensors, tokenizer/, and encoder/. File existence can be checked via GET /health/ready; if missing, inference returns 503 MODEL_UNAVAILABLE. Whether the model is actually compatible still requires a smoke test with inference.

Model inference for this project is not automatically downloaded at application startup; prepare the model volume before starting. A single application process only loads the models it needs, by default keeping at most one loaded; this can be adjusted with LAYA_MAX_LOADED_MODELS. That value controls the model cache size and does not limit the number of concurrent requests being handled. The service does not allocate additional inference concurrency slots; actual concurrency depends on the runtime thread pool, the models, and the machine resources. The CPU inference image uses a PyTorch CPU wheel; if deploying on GPU, use the appropriate PyTorch base environment for the device and validate acceptance.

Locally, first install the upstream runtime dependencies and the checked-out Laya (which is ignored by packaging), then download the three pinned-version models and run a real-request smoke test that includes English, explicit selection for Chinese, Chinese automatic routing, and typed-decisions:

```sh
.venv/bin/python -m pip install torch==2.5.1 transformers==4.48.3 safetensors==0.5.3 huggingface-hub==0.29.3 numpy==1.26.4
.venv/bin/python -m pip install --no-deps -e ./laya
LAYA_MODEL_DIR=models .venv/bin/python scripts/download-models.py --model english
LAYA_MODEL_DIR=models .venv/bin/python scripts/download-models.py --model multilingual
LAYA_MODEL_DIR=models .venv/bin/python scripts/download-models.py --model typed-decisions
LAYA_MODEL_DIR=models .venv/bin/python scripts/smoke-real-model.py
```

Model downloads and execution depend on upstream PyTorch, Transformers, Safetensors, Hugging Face Hub, and NumPy.

##  Getting started
Before first run, build the image, download the pinned-version models into the models volume, then start the application:

```sh
docker compose build
docker compose run --rm app python /app/scripts/download-models.py
docker compose up -d
```
Compose only starts the application service and exposes 127.0.0.1:8080 to the host. Public HTTPS access must be provided by an external reverse proxy that forwards requests to that port. SQLite data is in the laya-data volume and models are in the laya-models volume. The build machine needs access to PyPI, the PyTorch CPU package index, and the npm registry; once the image is built, starting it does not require pulling source code or dependencies.

If public HTTPS is provided by an existing 1Panel reverse proxy, forward domain requests to the host's 127.0.0.1:8080 and ensure LAYA_PUBLIC_ORIGIN matches the actual HTTPS domain. The reverse proxy is not one of the project’s application containers.

###   Local development (hot reload)
First run cp .env.example .env, then fill LAYA_ADMIN_USERNAME and LAYA_ADMIN_PASSWORD in .env. If using a hashed password, leave LAYA_ADMIN_PASSWORD empty and set LAYA_ADMIN_PASSWORD_HASH='...' (keep the single quotes). scripts/dev-backend.sh will set local sources, the SQLite path, and the model path to development values. Make sure the Python/frontend dependencies above and the three models are ready.

Open two terminals and run in the repository root:

```sh
# Terminales 1：FastAPI 与 SQLite
sh scripts/dev-backend.sh
```

```sh
# Terminales 2：React/Vite
cd frontend
pnpm dev
```
Open http://127.0.0.1:5173 to log in. Vite will proxy /internal and /v1 to the local port 8000; in production the built frontend is still served by the same FastAPI container. If you prefer a single local process, run pnpm build in frontend/, set LAYA_PUBLIC_ORIGIN to http://127.0.0.1:8000, start Uvicorn and open port 8000.

The console supports Simplified Chinese, English and Traditional Chinese. The login page and the top bar after login can switch languages; on first visit the language is chosen based on the browser language, and a manual selection is saved in the current browser.

##  Calling the API
After logging into the console, create a key on the API Keys page. The full key is shown only once in the creation response.

```sh
curl -X POST https://console.example.com/v1/systemone \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"state":{"message":"I was charged twice"},"questions":{"refund":{"type":"noul","instructions":"Does the customer ask for a refund?"}}}'
```

The state in the request can be a string, a JSON object, or an array; questions is a non-empty mapping of question IDs and supports noul, choice and score; model may be auto (default), english, multilingual or typed-decisions. The response preserves upstream answers, model and usage. Errors use detail.code and detail.message. Invalid key returns 401, validation failure 422, model unavailable 503. The console Playground uses an admin session and is recorded as a separate usage source.

##  Data and maintenance
SQLite runs in WAL mode; for backups stop the application first, then copy the database file or use the SQLite backup API to avoid missing data in the WAL. For restore, stop the application, replace the database in the persistent volume, then start. When upgrading Laya, update the full SHA in scripts/check-upstream.sh and the Dockerfile, re-checkout laya/, run tests and do a real-model smoke test.

```sh
.venv/bin/python -m pytest backend/tests -q
git ls-files laya/
```
The second command should produce no output.
