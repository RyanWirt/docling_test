# Docling + LiteLLM + a small local VLM

Run the official Docling browser UI and LiteLLM dashboard with Docker Compose.
Optionally protect **both UIs and their APIs** with Keycloak SSO. This is a
local development/test stack, not a native desktop app or production deployment.

```text
Browser → [optional oauth2-proxy + Keycloak] → Docling UI → LiteLLM → Ollama
Browser → [optional oauth2-proxy + Keycloak] → LiteLLM dashboard → PostgreSQL
```

## 1. Configure

Requirements: Docker/Compose **2.24.4+**, Python 3 (only for generating local
secrets), and internet access for image/model downloads. The stack runs on CPU;
no GPU configuration is required. Allow several GB of disk and RAM for the
images/services. Model memory usage and latency depend on your machine.

From the repository root, copy the example **only for a new setup**:

```sh
cp .env.example .env
```

Generate random credentials into the ignored `.env` using Python's standard
library. This fills empty secret fields only, so rerunning it preserves existing
credentials. Keep `.env` private; do not commit or share it.

```sh
python3 - <<'PY'
import base64
import secrets
from pathlib import Path

path = Path(".env")
text = path.read_text()
names = (
    "LITELLM_API_KEY", "POSTGRES_PASSWORD", "LITELLM_UI_PASSWORD",
    "KEYCLOAK_ADMIN_PASSWORD", "KEYCLOAK_TEST_PASSWORD",
    "OIDC_CLIENT_SECRET", "OAUTH2_COOKIE_SECRET",
)
for name in names:
    if name == "OAUTH2_COOKIE_SECRET":
        value = base64.urlsafe_b64encode(secrets.token_bytes(32)).decode()
    else:
        value = ("sk-" if name == "LITELLM_API_KEY" else "") + secrets.token_hex(32)
    text = text.replace(name + "=\n", name + "=" + value + "\n")
path.write_text(text)
path.chmod(0o600)
PY
```

If upgrading from the external-LiteLLM setup, merge the new example fields into
your existing `.env` before generating secrets. For the bundled stack set
`LITELLM_CHAT_COMPLETIONS_URL=http://litellm:4000/v1/chat/completions`,
`OLLAMA_MODEL=ibm/granite-docling:258m`, `VLM_RESPONSE_FORMAT=doctags`, and
`VLM_PROMPT=Convert this page to docling.`. Do not overwrite credentials for an
existing external proxy.

## 2. Start the local stack

```sh
export COMPOSE_FILE=compose.yaml:compose.local.yaml
docker compose config --quiet
docker compose up -d --wait --wait-timeout 900
```

The examples use a POSIX shell. Alternatively, pass
`-f compose.yaml -f compose.local.yaml` to every Compose command (and append
`-f compose.sso.yaml` for SSO), including `logs` and `down`.

On first startup, `model-pull` requests **`ibm/granite-docling:258m`** from
Ollama's model registry. This tiny document-specific vision model runs locally
on CPU and returns DocTags. LiteLLM waits for the download to succeed; Docling
waits for LiteLLM. An exited **0** `model-pull` container is expected.
The weights persist in the `ollama-data` volume. Downloads still require network
access even though inference is local; Ollama cloud inference is disabled.

| UI without SSO | Login |
| --- | --- |
| Docling: <http://localhost:5001/ui> | None (local-only) |
| LiteLLM dashboard: <http://localhost:4000/ui> | `admin` / `LITELLM_UI_PASSWORD` from `.env` |

In Docling upload a small PDF/image, set **Pipeline → vlm**, then under
**Models and presets** explicitly choose **VLM preset → litellm**. The
standard pipeline does not use LiteLLM. Inspect the conversion and LiteLLM
request logs. The LiteLLM alias is `local-vision`, independent of the Ollama
model ID. PostgreSQL persists the dashboard's data in `litellm-data`.

After changing `.env`, rerun the startup command to recreate affected services.

## 3. Optional Keycloak SSO

Keep the same `.env` and select the additional override:

```sh
export COMPOSE_FILE=compose.yaml:compose.local.yaml:compose.sso.yaml
docker compose config --quiet
docker compose up -d --wait --wait-timeout 900
```

The SSO override **removes** Docling's port 5001 and LiteLLM's port 4000
from the host. Use these URLs instead, with **localhost**, not `127.0.0.1`:

| Endpoint | Credentials |
| --- | --- |
| Docling: <http://localhost:4180/ui> | Keycloak `tester` / `KEYCLOAK_TEST_PASSWORD` |
| LiteLLM: <http://localhost:4181/ui> | Same Keycloak user, then LiteLLM `admin` / `LITELLM_UI_PASSWORD` |
| Keycloak admin: <http://localhost:8080/admin/> | `admin` / `KEYCLOAK_ADMIN_PASSWORD` |

The imported `docling` realm supplies the preset **tester** user, verified test
email, and confidential `docling-ui` client with exact callback URLs and PKCE.
No fixed password or client secret is committed. Both gateways use the same
Keycloak session: after signing into one, the second should not prompt for the
Keycloak password again.

SSO is an **authentication gateway**, not LiteLLM's paid/native SSO integration
or a mapping of Keycloak roles to LiteLLM roles. LiteLLM still requires its own
dashboard admin login; the Keycloak test user alone does not become a LiteLLM
administrator. Gateway authentication applies to API requests too, including
requests carrying a LiteLLM API key. Docling calls LiteLLM privately on the
Docker network, so its model requests do not require a browser session.

Keycloak's browser-facing issuer is `http://localhost:8080/realms/docling`.
Gateways validate that issuer while using `http://keycloak:8080` for token,
userinfo, and signing-key retrieval. This avoids container-localhost routing
problems without disabling issuer, signature, or email verification.

Wait for Keycloak's login page on the first start; container startup alone
does not mean the realm import has finished. To sign out, visit each gateway's
`/oauth2/sign_out`, **then sign out of Keycloak** at
<http://localhost:8080/realms/docling/account/>. Clearing only a gateway cookie
does not end the shared Keycloak SSO session.

Keycloak stores its development database in `keycloak-data`. Realm imports are
**first-start only**: changing the test password or client secret in `.env`
will not update an already imported realm. Update the user/client in the admin
console as well, or reset **only the disposable Keycloak volume**:

```sh
docker compose down
# List volume names first; the project prefix depends on your checkout.
docker volume ls
# Replace <project> with your Compose project name. Deletes users/sessions:
docker volume rm <project>_keycloak-data
docker compose up -d --wait --wait-timeout 900
```

The Keycloak bootstrap admin password also only applies when initializing its
database. Similarly, changing PostgreSQL's password requires updating the
existing database password, not just its environment variable.

To disable SSO, run `docker compose down` with the SSO file list still selected,
then select `COMPOSE_FILE=compose.yaml:compose.local.yaml` and start again.
This intentionally restores the unauthenticated local Docling port.

## Existing external LiteLLM (original mode)

Use **only** `compose.yaml`; no local model, database, or SSO is required:

```sh
unset COMPOSE_FILE
# Edit .env for your existing endpoint, key, alias, and model output format.
docker compose -f compose.yaml up -d --wait
```

Set `LITELLM_CHAT_COMPLETIONS_URL` to the full container-reachable endpoint,
e.g. `http://host.docker.internal:4000/v1/chat/completions`, and
`LITELLM_MODEL` to your proxy's exact alias. Set `LITELLM_API_KEY` to the
existing proxy key (empty only if authentication is disabled).
For general vision models use `VLM_RESPONSE_FORMAT=markdown` and
`VLM_PROMPT=Convert this page to markdown. Do not miss any text and only output the bare markdown!`.
The base Compose file retains its external-host URL and Markdown defaults
when these variables are absent.

`localhost` inside Docling means Docling itself, not the host or another
container. A host proxy must listen on an interface reachable from Docker;
use a firewall to limit access. For remote endpoints prefer HTTPS.

## Other local models

To switch the bundled model, set `OLLAMA_MODEL` and the matching parser/prompt
in `.env`, then rerun `docker compose up -d --wait --wait-timeout 900`.

| Model | Settings / tradeoff |
| --- | --- |
| `ibm/granite-docling:258m` | Default, small document VLM. `doctags` + `Convert this page to docling.` |
| `gemma3:4b` | General-purpose vision; use the Markdown settings above. Larger/slower on CPU. The 1B variant is text-only. |
| Qwen2.5-VL 3B / 7B | Alternative via a supported external backend, e.g. vLLM with `Qwen/Qwen2.5-VL-3B-Instruct`; use Markdown settings. |

The one-shot download service runs again on `up`; existing model blobs are
reused. Quantization, context length, and page resolution affect memory use.
Start with one page; no model's extraction accuracy is guaranteed.

## Checks, API use, and shutdown

Use the same `COMPOSE_FILE` selection as for startup:

```sh
docker compose ps -a
docker compose logs --tail=100 model-pull litellm docling
docker compose exec ollama ollama list
docker compose exec docling python -c \
  "import urllib.request; print(urllib.request.urlopen('http://localhost:5001/health').read().decode())"
```

Without SSO, API docs are at <http://localhost:5001/docs>; a conversion request:

```sh
curl --fail-with-body http://localhost:5001/v1/convert/file \
  -F 'files=@/absolute/path/to/sample.pdf' \
  -F 'pipeline=vlm' \
  -F 'vlm_pipeline_preset=litellm' \
  -F 'to_formats=md'
```

With SSO use the browser UI/API docs at port 4180 after logging in; an
unauthenticated curl is intentionally redirected to login, not to the API.
Confirm that ports 4000/5001 are no longer published with `docker compose ps`.

- **Model download fails:** inspect `model-pull` logs and allow access to
  Ollama's registry/blob hosts. After fixing connectivity, rerun `up`.
  LiteLLM will not start against an incomplete download.
- **401/403:** check the LiteLLM key/alias, or sign into Keycloak when using SSO.
- **SSO redirect/cookie errors:** use `localhost` consistently, confirm ports
  8080/4180/4181 are free, and check `keycloak`/`docling-sso` logs. These callback
  ports are intentionally fixed; changing them requires updating the realm too.
- **No model traffic:** choose `pipeline=vlm` and the `litellm` preset.
- **Timeouts/truncated text:** warm up the model, try one page, and adjust
  Docling's 180-second timeout/4096-token budget in `compose.yaml`.

Stop with `docker compose down`; volumes survive. **`docker compose down -v`
deletes all selected-stack data**, including downloaded models, dashboard data,
and Keycloak users/sessions when SSO is selected.

### Security boundaries

All published ports bind to loopback. Ollama and PostgreSQL are never
host-published. The SSO override removes direct UI/API ports rather than leaving
an authentication bypass. Services still trust each other on the Compose
network; untrusted local processes/Docker administrators are outside this
demo's security boundary.

Keycloak `start-dev`, its file database, HTTP, and non-secure local cookies are
**development only**. Do not expose this stack publicly. Production needs HTTPS,
secure cookies, real issuer/redirect hostnames, a supported Keycloak database,
secret management, authorization policy, and backups. Test only non-sensitive
documents. Container inspection and `docker compose config` without `--quiet`
can disclose environment credentials. LiteLLM's master key is for this local
demo; use restricted virtual keys for shared deployments.

Images for the added stack are pinned by digest for reproducibility (LiteLLM
1.106.2, Ollama 0.40.8, Keycloak 26.8.1, oauth2-proxy 7.15.5). Model tags resolve
when pulled; the Ollama volume caches the resulting model blobs.

References: [Docling configuration](https://github.com/docling-project/docling-serve/blob/v1.37.0/docs/configuration.md),
[remote preset](https://github.com/docling-project/docling-serve/blob/v1.37.0/examples/config.yaml),
[LiteLLM dashboard](https://docs.litellm.ai/docs/proxy/ui),
[Ollama provider](https://docs.litellm.ai/docs/providers/ollama),
[Keycloak imports](https://www.keycloak.org/server/importExport),
[oauth2-proxy Keycloak OIDC](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/keycloak_oidc/).