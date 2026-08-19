# Fasiri - Team Onboarding & Architecture Guide

This document is for anyone joining the Fasiri project. It explains what Fasiri
is, how the code is laid out, how to run it, how each piece fits together, and
how to do the common jobs you'll be asked to do (add a model, add a provider,
add a language, rotate keys, update the docs, ship a deploy).

Read it top to bottom once. After that, treat the "Common tasks" and "Repo map"
sections as your reference.

---

## i. What Fasiri actually is

Fasiri is one API that translates, transcribes, and synthesises speech across
30+ African languages. Instead of every app talking to Sunbird, GhanaNLP, and
HuggingFace separately (three different auth schemes, three sets of language
codes, three failure modes), they talk to Fasiri, and Fasiri routes each request
to the best provider for that language pair and falls back automatically if a
provider is down.

There are three moving layers:

1. **The API service** (`app/`) - a FastAPI app. This is the product. It owns
   authentication, rate limiting, routing, and the provider adapters.
2. **The Python SDK** (`fasiri/`) - a client library so Python developers can
   call Fasiri (or the underlying providers directly) without writing HTTP by
   hand.
3. **The docs site** (`docs/`) - a MkDocs (Material theme) site that documents
   the API and the SDK for outside users.

The providers behind it:

- **Sunbird AI** - Ugandan languages (Luganda, Runyankole, Lugbara, Acholi,
  Ateso, and English). Best quality for those. Also the only provider that does
  speech (STT and TTS).
- **Khaya / GhanaNLP** - West and East African languages (Twi, Ewe, Ga, Fante,
  Yoruba, Dagbani, Kikuyu, Gurune, Luo, Kimeru, Kusaal).
- **HuggingFace** (Helsinki-NLP OPUS models) - the catch-all fallback for
  everything else, and the safety net when Sunbird or Khaya fail.

Note for anyone who worked on the old version: the Groq / llama-3.3 chat demo
and the Next.js frontend are no longer part of this repo. The one thing that
carried over is the key-persistence design (Neon Postgres), which is still here
and still important.

---

## ii. The request lifecycle (the mental model)

A single translation call travels through the system like this:

1. Request hits `POST /api/v1/translate` in `app/api/translate.py`.
2. `require_api_key` (in `app/middleware/auth.py`) checks the `Authorization:
   Bearer fsri_...` header against the key store and rejects if invalid.
3. `check_rate_limit` (in `app/middleware/ratelimit.py`) checks the caller
   hasn't blown their per-minute budget.
4. If no `source_lang` was given, `detect_language` (in
   `app/utils/lang_detect.py`) guesses it.
5. `route_translation` (in `app/services/routing.py`) decides which provider
   should handle this pair, asks the registry (`app/core/registry.py`) for the
   best model, and calls that provider's adapter in
   `app/services/providers/`.
6. If the primary provider throws, routing catches it and tries the fallback
   chain (Sunbird -> Khaya -> HuggingFace, roughly).
7. The provider adapter returns a `TranslationResult`, which the endpoint wraps
   in a `TranslateResponse` (defined in `app/schemas/translate.py`) and returns
   as JSON.

If you understand those seven steps, you understand 80% of the codebase.
Everything else is a variation on that path (batch, speech, key management).

---

## iii. Repo map

Top level:

| Path | What it is |
|------|-----------|
| `app/` | The FastAPI service. This is the deployed product. |
| `fasiri/` | The Python SDK, v1.1.0. **This is the canonical SDK** (see pyproject). |
| `sdk/fasiri_sdk/` | An older, near-duplicate SDK, v1.0.0. See the gaps section. |
| `docs/` | MkDocs Material documentation site. |
| `tests/` | Pytest suite (`test_translate.py`). |
| `nginx/` | Reverse-proxy config for self-hosting behind nginx. |
| `scripts/setup_server.sh` | Bare-metal server provisioning script. |
| `smoke_test.py`, `stress_test.py`, `test_live.py` | Manual scripts that hit a live deployment. Not part of the pytest suite. |
| `get_sunbird_token.py` | Helper to fetch a fresh Sunbird JWT. |
| `Dockerfile`, `docker-compose.yml` | Container build and local multi-service run. |
| `render.yaml`, `railway.toml` | Platform deploy configs (Render is the live one). |
| `requirements.txt` | Runtime + test dependencies for the API. |
| `pyproject.toml`, `setup.py` | Packaging for the `fasiri` SDK (published to PyPI). |
| `mkdocs.yml` | Docs site config and navigation. |
| `env.example` | Template for your local `.env`. |
| `README.md`, `DEPLOYMENT.md`, `CHANGELOG.md`, `TESTING.md` | Project docs. |

Inside `app/` (the part you'll edit most):

| File | Responsibility |
|------|---------------|
| `main.py` | App entry point. Startup lifespan (init DB, register demo/admin keys, run provider diagnostics), CORS, request-timing middleware, global exception handler, router wiring, `/health`. Run with `uvicorn app.main:app`. |
| `core/config.py` | All settings, read from env vars via pydantic-settings. Single source of truth for configuration. |
| `core/database.py` | The Postgres key store (psycopg2, raw SQL, `api_keys` table). Falls back to in-memory if `DATABASE_URL` is unset. |
| `core/security.py` | Key generation (`fsri_` + 40 hex), SHA-256 hashing, lookup, expiry checks, the local dev key. |
| `core/registry.py` | `LANGUAGE_REGISTRY` (every language and its metadata) and `MODEL_REGISTRY` (every model, its provider, quality score). Picks the best model per pair. |
| `api/auth.py` | `POST /auth/keys` (admin-only key issuance) and `GET /auth/keys/me`. |
| `api/translate.py` | `POST /translate` and `POST /translate/batch`. |
| `api/speech.py` | `POST /speech/stt` and `POST /speech/tts` (Sunbird only). |
| `api/languages.py` | `GET /languages` - lists everything with capability flags. |
| `api/debug.py` | `GET /debug/env` and `GET /debug/providers`. Only active when `DEBUG=true`. |
| `middleware/auth.py` | The `require_api_key` dependency every protected endpoint uses. |
| `middleware/ratelimit.py` | Sliding-window rate limiter. Redis in production, in-memory locally. |
| `schemas/translate.py` | Every Pydantic request/response model. If you change an API shape, you change it here. |
| `services/routing.py` | `route_translation` - provider selection and the fallback chain. |
| `services/providers/base.py` | `BaseProvider` abstract class + the result dataclasses (`TranslationResult`, `STTResult`, `TTSResult`). |
| `services/providers/sunbird.py` | Sunbird adapter (translate, STT, TTS). |
| `services/providers/huggingface.py` | HuggingFace Helsinki-NLP adapter (fallback translate). |
| `services/providers/khaya.py` | GhanaNLP / Khaya adapter (West African translate). |
| `utils/lang_detect.py` | Language auto-detection and provider code mapping. |

---

## iv. How the core pieces connect

### Providers and the base contract

Every provider in `app/services/providers/` extends `BaseProvider`
(`base.py`) and must implement `translate(...)`. Speech methods
(`speech_to_text`, `text_to_speech`) are optional - the base class raises
`NotImplementedError`, so a provider that doesn't do speech just doesn't
override them. Right now only Sunbird overrides them.

They all return the same dataclasses (`TranslationResult`, etc.), so the rest of
the system never has to care which provider ran. That uniform return type is
what makes routing and fallback clean.

### The registry

`core/registry.py` holds two things:

- `LANGUAGE_REGISTRY`: for each language code, its name, native name, region,
  family, and any provider-specific codes (Sunbird's `swa`, NLLB's `swh_Latn`,
  etc.).
- `MODEL_REGISTRY`: for each model, which provider serves it, which language
  pairs it covers, and a `quality_score`.

On import it builds a `(source, target) -> best model` index so lookups are
O(1). When you add a language or a model, this is the file you touch.

### Routing and fallback

`services/routing.py` is the decision-maker. For `provider=auto` (the default)
it picks:

- Sunbird if both languages are in the Ugandan set,
- else Khaya if Khaya supports the pair,
- else HuggingFace.

If the chosen provider raises, it walks a fallback chain rather than failing the
request outright. Sunbird down -> try Khaya (if it supports the pair) or
HuggingFace. Khaya down -> HuggingFace. HuggingFace down -> 503. This is why a
provider outage degrades quality instead of taking the whole API down.

### The key store and auth

Keys look like `fsri_<40 hex chars>`. We never store the plain key - only its
SHA-256 hash (`core/security.py`). On startup (`main.py` lifespan) two permanent
keys are loaded from env and upserted into the store:

- `FASIRI_DEMO_KEY` - read access, used by the public demo.
- `FASIRI_ADMIN_KEY` - the only key allowed to issue new keys via
  `POST /auth/keys`.

The store is Neon Postgres (`core/database.py`). If `DATABASE_URL` is unset it
silently falls back to an in-memory dict, which means every restart wipes all
issued keys. That fallback is the exact bug that bit us before, so treat a
missing `DATABASE_URL` in any real environment as a defect, not a convenience.

### Rate limiting

`middleware/ratelimit.py` does a sliding window per key per endpoint group.
If `REDIS_URL` is set it uses Redis (shared across workers - correct for
production). If not, it uses an in-memory window, which is per-worker and
therefore leaky once you run more than one worker. Limits come from
`config.py` (`RATE_LIMIT_RPM`, `RATE_LIMIT_BATCH_RPM`).

---

## v. Running it locally

1. Clone and enter the repo, create a virtualenv:
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   ```
2. Copy the env template and fill it in:
   ```bash
   cp env.example .env
   ```
   For a first run you can leave the provider keys empty - the server boots in
   "stub" mode and still starts. To get real translations, set at minimum
   `HUGGINGFACE_API_KEY`. For Ugandan languages and speech, set
   `SUNBIRD_API_KEY` (use `get_sunbird_token.py` to fetch a fresh JWT). For
   West African languages, set `KHAYA_API_KEY`.
3. To exercise the persistent key store locally, set `DATABASE_URL` to a Neon
   (or any Postgres) connection string. Without it you're on the in-memory
   fallback.
4. Run the server:
   ```bash
   uvicorn app.main:app --reload
   ```
5. Open the interactive docs at `http://localhost:8000/docs`. The startup logs
   print a **dev key** when `DEBUG=true` - copy it and use it as
   `Authorization: Bearer <dev-key>` to try the endpoints.
6. Sanity-check with `GET /health` - it reports the key store mode and which
   providers are live vs stub.

---

## vi. The API surface

All endpoints are under the `/api/v1` prefix except `/health` and `/`.

| Method & path | Auth | Purpose |
|---------------|------|---------|
| `POST /api/v1/translate` | any valid key | Translate one string. Auto-detects source if omitted. |
| `POST /api/v1/translate/batch` | any valid key | Up to 50 items, processed concurrently. |
| `POST /api/v1/speech/stt` | any valid key | Transcribe an uploaded audio file (Sunbird). |
| `POST /api/v1/speech/tts` | any valid key | Synthesise speech from text (Sunbird). |
| `GET /api/v1/languages` | none | List all languages with capability flags and voice IDs. |
| `POST /api/v1/auth/keys` | **admin key** | Issue a new API key. Plain key returned once, never again. |
| `GET /api/v1/auth/keys/me` | any valid key | Inspect the calling key's metadata. |
| `GET /api/v1/debug/env` | none (DEBUG only) | Show which env vars are set, values masked. |
| `GET /api/v1/debug/providers` | none (DEBUG only) | Live connectivity check against each provider. |
| `GET /health` | none | Status, version, key-store mode, provider states. |

Every request/response body is defined in `app/schemas/translate.py`. That file
is the contract - read it before changing any endpoint.

---

## vii. The Python SDK

The SDK (`fasiri/`) has two modes:

- **Cloud mode** - you pass a Fasiri API key and it calls the hosted API.
  ```python
  from fasiri import Fasiri
  client = Fasiri(api_key="fsri_...")
  print(client.translate("Good morning", target="lug"))
  ```
- **Direct mode** - you pass your own provider keys and the SDK routes to the
  providers directly from your machine, with no Fasiri account. Fasiri is then
  just the routing/abstraction layer and you handle provider billing yourself.
  ```python
  from fasiri import Fasiri
  from fasiri.providers import SunbirdProvider, KhayaProvider, HuggingFaceProvider
  client = Fasiri(providers=[SunbirdProvider(api_key="ey..."), ...])
  ```

The SDK mirrors the API: `translate`, `translate_batch`, `transcribe`,
`synthesise`, `languages`, and a set of typed exceptions
(`AuthenticationError`, `RateLimitError`, `UnsupportedLanguageError`,
`ProviderError`).

---

## viii. Common tasks

### Add a new translation model

1. In `app/core/registry.py`, add a `ModelEntry` to `MODEL_REGISTRY` with its
   `model_id`, `provider`, supported `source_langs`/`target_langs`, and a
   `quality_score`. The router automatically picks the highest score for a pair,
   so scoring is how you control priority.
2. If the model needs a new language, add it to `LANGUAGE_REGISTRY` first
   (see below).
3. If it runs on an existing provider (e.g. another Helsinki model on
   HuggingFace), that's all - the adapter already knows how to call it. If it
   needs a new provider, see the next task.

### Add a new provider

1. Create `app/services/providers/<name>.py` with a class extending
   `BaseProvider`. Implement `translate(...)`, returning a `TranslationResult`.
   Override the speech methods only if it does speech.
2. Register it in `app/services/routing.py`: add a lazy singleton getter, wire
   it into `_get_provider`, and add its selection rule to `_best_provider_for`
   and the fallback chain.
3. Add its base URL and credential to `app/core/config.py`, and document the
   env var in `env.example`.
4. Add the provider name to the `Provider` enum in `app/schemas/translate.py`
   if callers should be able to force it via the `provider` field. (See the
   gaps section - Khaya is currently missing from that enum.)

### Add a new language

1. Add a `LanguageInfo` entry to `LANGUAGE_REGISTRY` in `core/registry.py`,
   including any provider-specific codes.
2. Make sure at least one `ModelEntry` covers it (a dedicated model, or the
   HuggingFace wildcard fallback).
3. If it supports speech, add its code to the `_SUPPORTED_STT` / `_SUPPORTED_TTS`
   sets in `app/api/speech.py` and the voice-ID map in `app/api/languages.py`.
4. If it should be auto-detectable, add it to the maps in
   `app/utils/lang_detect.py`.

### Issue, inspect, or rotate keys

- Issue: `POST /api/v1/auth/keys` with the **admin** key in the header and a
  `name` in the body. The plain key is returned once - save it immediately.
- Inspect: `GET /api/v1/auth/keys/me` with any key.
- Rotate the demo/admin keys: generate a new one
  (`python -c "import os; print('fsri_'+os.urandom(20).hex())"`), update
  `FASIRI_DEMO_KEY` / `FASIRI_ADMIN_KEY` in the Render env, and redeploy. They're
  re-upserted on startup.

### Update the docs

The docs live in `docs/` and are configured by `mkdocs.yml`. To work on them:

```bash
pip install -r requirements-docs.txt
mkdocs serve      # live preview at http://localhost:8000
```

Add a new page as a markdown file under the right `docs/` subfolder, then add it
to the `nav:` tree in `mkdocs.yml` or it won't appear. The SDK reference pages
pull docstrings automatically via mkdocstrings, so keeping SDK docstrings good
keeps those pages good for free.

### Deploy

The live service runs on Render from `render.yaml`, built with the `Dockerfile`.
Push to the connected GitHub branch and Render rebuilds. The health check path
is `/health`. Before relying on a deploy, confirm the required env vars are set
in the Render dashboard - see the gaps section, because `render.yaml` does not
currently list all of them.

---

## ix. Known gaps and cleanup backlog

These are real issues in the current tree. They aren't blockers, but the team
should know about them so nobody loses a day rediscovering them. Roughly in
priority order:

1. **`render.yaml` is missing env vars the app needs.** It does not list
   `DATABASE_URL`, `KHAYA_API_KEY`, `FASIRI_DEMO_KEY`, or `FASIRI_ADMIN_KEY`.
   Without `DATABASE_URL` the API silently drops to the in-memory key store and
   loses all keys on every restart - the exact failure we fixed before. Without
   the demo/admin keys, the public demo and key issuance don't work. These have
   to be set manually in the Render dashboard today; they should be in the file.

2. **`env.example` is missing the same four vars.** A new developer copying it
   won't know to set `DATABASE_URL`, `KHAYA_API_KEY`, `FASIRI_DEMO_KEY`, or
   `FASIRI_ADMIN_KEY`. Worth adding, with comments, so local setup matches
   production.

3. **Two SDKs.** `fasiri/` (v1.1.0, cloud + direct mode) and `sdk/fasiri_sdk/`
   (v1.0.0, cloud only) are near-duplicates. `pyproject.toml` packages
   `fasiri/`, so that is the canonical one - but the `Dockerfile` copies `sdk/`
   into the image and the readme points at `sdk/README_SDK.md`. Pick one, delete
   or clearly archive the other, and fix the references. Until then, always work
   in `fasiri/`.

4. **Khaya is missing from the `Provider` enum** in
   `app/schemas/translate.py` (which only has `sunbird`, `huggingface`, `auto`).
   Auto-routing uses Khaya fine, but a caller cannot force `provider=khaya` -
   the request fails validation first. Add it to the enum.

5. **Registry/HuggingFace inconsistency on NLLB.** `registry.py` still lists
   `facebook/nllb-200-distilled-600M` as the wildcard fallback, but the
   `huggingface.py` header notes NLLB was dropped by the HF Inference API in 2025
   and now returns 400. The intended live fallback is `opus-mt-en-mul` (set as
   `DEFAULT_MODEL_ID`). Reconcile these so the registry doesn't point at a dead
   model.

6. **Dead code in `routing.py`.** The bottom of the file is a large
   commented-out previous version of the router. Delete it - git history already
   has it.

7. **`Dockerfile` copies `sdk/`, not `fasiri/`.** Harmless today because the
   running API doesn't import the SDK package, but it's inconsistent with
   `pyproject.toml` and will confuse the next person. Align it when you resolve
   the two-SDK issue.

---

*Maintained by Beta-Tech Labs. When the code changes, change this file in the
same PR - an onboarding doc is only trustworthy if it stays honest.*
