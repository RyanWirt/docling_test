# Docling UI with an existing LiteLLM server

Run the official [Docling Serve](https://github.com/docling-project/docling-serve)
browser UI using Docker Compose. This is a local, desktop-accessible web UI, **not
a native desktop application**. LiteLLM and its model backend run separately;
this Compose file does not start either of them.

## Start

Requirements: Docker Engine or Docker Desktop with Compose v2, and an existing
LiteLLM proxy exposing an OpenAI-compatible **vision** model. Docling runs on CPU;
model inference uses whatever hardware your existing backend provides. The
official image is several GB, so the first pull can take a while.

Run these commands from the repository root:

```sh
cp .env.example .env
# Edit .env: set the URL, model alias, and LiteLLM API key.
docker compose up -d --wait
```

Open <http://localhost:5001/ui> (or your `DOCLING_PORT`). Upload a small PDF or
image, set **Pipeline** to **vlm**, and under **Models and presets** select
**VLM preset** = **litellm** explicitly.
Then convert and inspect the Markdown result. The standard pipeline does not
use this VLM preset; selecting `vlm` is essential. Check your LiteLLM request
logs to confirm the configured model alias received the page images.

The API docs are at <http://localhost:5001/docs>. To exercise the same preset
without the UI, upload your own local PDF:

```sh
curl --fail-with-body http://localhost:5001/v1/convert/file \
  -F 'files=@/absolute/path/to/sample.pdf' \
  -F 'pipeline=vlm' \
  -F 'vlm_pipeline_preset=litellm' \
  -F 'to_formats=md'
```

After changing `.env`, rerun `docker compose up -d --wait` to recreate the
container. Stop it with `docker compose down`.

## Connect to your LiteLLM instance

| Setting in `.env` | Meaning |
| --- | --- |
| `LITELLM_CHAT_COMPLETIONS_URL` | **Full** endpoint, e.g. `http://host.docker.internal:4000/v1/chat/completions`, not just the base URL. |
| `LITELLM_MODEL` | Exact `model_name` alias advertised by your proxy, not necessarily the backend's model ID. |
| `LITELLM_API_KEY` | LiteLLM virtual/API key. Empty is only appropriate when proxy authentication is disabled. |
| `DOCLING_PORT` | Local UI/API port; defaults to `5001`. |
| `VLM_RESPONSE_FORMAT` / `VLM_PROMPT` | Model output parser and instruction; default to Markdown transcription. |

- **LiteLLM on the host:** use the supplied `host.docker.internal` URL. The
  Compose host-gateway mapping also supports Linux. LiteLLM must listen on an
  interface reachable from Docker, not only host loopback; use firewall rules
  to limit access.
- **LiteLLM on another machine:** use its reachable hostname/IP, preferably
  with HTTPS. `localhost` inside the Docling container means Docling itself.
- **LiteLLM in another Compose project:** use its host-published port via
  `host.docker.internal`, or attach Docling to its existing Docker network and
  use the LiteLLM service name as the hostname.

The preset forwards the LiteLLM key in the Authorization header using the bearer
scheme and requests one page at a time, with a 180-second per-request timeout
and a 4096-token output budget.
Increase these values in `compose.yaml` for slow backends or dense pages.
Environment values are interpolated into JSON: avoid literal double quotes,
backslashes, or newlines in the model alias, URL, key, and custom prompt.

## Models to test locally behind LiteLLM

Use a backend that accepts OpenAI-style image messages (for example, Ollama,
vLLM, or LM Studio) and route it through your existing LiteLLM proxy. Text-only
models cannot transcribe page images.

| Model | Example local backend ID | Why try it |
| --- | --- | --- |
| Qwen2.5-VL 3B / 7B | `Qwen/Qwen2.5-VL-3B-Instruct` or `Qwen/Qwen2.5-VL-7B-Instruct` (vLLM) | General document/vision baseline; 3B needs less memory, 7B is a quality comparison. |
| Gemma 3 4B | `gemma3:4b` (Ollama) | Small general-purpose vision model; use Markdown defaults. The 1B variant is text-only. |
| Granite Docling 258M | `ibm/granite-docling:258m` (Ollama) | Tiny document-specialized model; set `VLM_RESPONSE_FORMAT=doctags` and `VLM_PROMPT=Convert this page to docling.` |

Availability and memory usage depend on backend support, quantization, image
resolution, and context length. Start with a single-page document. The DocTags
settings are only for document-specialized models; switch back to Markdown
when testing Qwen or Gemma.

For example, if you use Ollama, install a model **on your existing backend**:

```sh
ollama pull gemma3:4b
```

Merge an entry like this into your **existing LiteLLM configuration** (do not
replace your current configuration or start another proxy):

```yaml
model_list:
  - model_name: local-vision
    litellm_params:
      model: ollama_chat/gemma3:4b
      api_base: http://YOUR_OLLAMA_HOST:11434
```

`YOUR_OLLAMA_HOST` must be reachable from **LiteLLM**, independently of the
Docling-to-LiteLLM connection. Reload your proxy as usual and set
`LITELLM_MODEL=local-vision` in this project's `.env`. For the other models,
add corresponding routes using your backend's supported model IDs.

## Checks and troubleshooting

```sh
docker compose config --quiet
docker compose ps
docker compose logs --tail=100 docling
curl --fail http://localhost:5001/health
```

- **401/403:** verify the LiteLLM key has permission for the selected alias.
- **404/model not found:** check the full endpoint and alias in your proxy's
  `/v1/models` response.
- **Connection refused:** check container-to-proxy routing and the proxy's bind
  address. Do not use `localhost` for the proxy unless it shares the container.
- **No model traffic:** ensure the UI uses `pipeline=vlm`, not `standard`.
- **Timeouts/truncated text:** try one page, warm up the backend model, and
  adjust the timeout/output budget above.

The UI/API is bound to `127.0.0.1` because this is a local testing setup, not a
public authenticated service. Do not expose it directly to the internet.
Uploaded pages are sent to the configured model endpoint; use non-sensitive
test documents. `.env` is ignored by Git, but container environment inspection
and `docker compose config` (without `--quiet`) can reveal your key.

Upstream references: [server configuration](https://github.com/docling-project/docling-serve/blob/v1.37.0/docs/configuration.md),
[remote preset example](https://github.com/docling-project/docling-serve/blob/v1.37.0/examples/config.yaml),
[Docling VLM examples](https://docling-project.github.io/docling/examples/vlm_pipeline_api_model/),
[LiteLLM Ollama provider](https://docs.litellm.ai/docs/providers/ollama).