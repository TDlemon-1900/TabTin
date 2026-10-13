# Import an installed Ollama model

[中文](ollama-byok.md)

Community installations can use Ollama through the existing OpenAI-compatible
BYOK path. Model files remain in Ollama; TabTin stores the connection and model
configuration, not a copy of the weights. No cloud Qwen API subscription is needed.

## Import

Run this administrative command in the Django runtime after creating an account
and organization. Obtain the organization ID from your local installation; do
not commit account IDs, credentials, or database dumps.

```sh
python manage.py import_ollama_model \
  --organization-id YOUR_ORGANIZATION_ID \
  --model qwen3:8b \
  --base-url http://localhost:11434/v1 \
  --context-window 4096 --max-output-tokens 2048 \
  --dry-run
```

Remove `--dry-run` to save. Add `--set-default` to explicitly select this model
as the organization's default. Without it, the command preserves the default.
Personal organizations use their owner's personal BYOK connection; teams use
an organization connection. Run once per desired organization.

For Docker Desktop with Ollama on the host, use
`http://host.docker.internal:11434/v1` instead of localhost. With Compose:

```sh
docker compose exec django python manage.py import_ollama_model \
  --organization-id YOUR_ORGANIZATION_ID --model qwen3:8b \
  --base-url http://host.docker.internal:11434/v1 --set-default
```

The Django image must contain this command: rebuild the image when using an
older source snapshot. Ensure the Ollama listener is reachable from Django.
Use the exact installed model name, including its tag. The command inspects
`/api/show`; it never downloads a missing model. It does not use inherited HTTP
proxies for this local metadata request. If your endpoint requires a key, set
`OLLAMA_API_KEY` in the Django command's environment; otherwise `ollama` is used
as a non-secret placeholder.

If the OpenAI endpoint fails to load a model because the serving context uses
too much GPU memory, create a dedicated alias with an explicit context before
importing it. This reuses the installed weights:

```text
# Modelfile.tabtin
FROM qwen3:8b
PARAMETER num_ctx 4096
```

```sh
ollama create qwen3-tabtin:8b -f Modelfile.tabtin
# Import with --model qwen3-tabtin:8b --context-window 4096
```

Changing TabTin's context field alone does not change Ollama's serving context.
See [Ollama's context configuration](https://docs.ollama.com/api/openai-compatibility#setting-the-local-context-size).

Re-running preserves existing model settings and avoids duplicate connections.
An existing disabled or different connection is rejected; choose another
`--provider-key` for a separate Ollama endpoint. Existing model settings can be
edited in Settings → Model Configuration. The import reuses the normal BYOK
creation and permission checks, acting as the organization's owner.

## Capabilities and verification

The command reads Ollama's advertised `tools` and `vision` capabilities instead
of inheriting cloud OpenAI defaults. For `qwen3:8b`, vision is normally disabled.
Set `--context-window` to the **actual serving context**; the default is a
conservative 4096, not the theoretical maximum reported by the model. Ensure
Ollama serves at least this context. `--max-output-tokens` must be smaller.

In TabTin, refresh the model list, select **Ollama / qwen3:8b**, and verify:

1. Connection test, followed by a real generated reply.
2. Streaming output.
3. A tool call and a follow-up reply after supplying the tool result.

The connection probe alone does not verify generation or Agent behavior.
Hardware and model quality determine latency and tool reliability. Keep model
storage on a disk with enough free space; importing does not relocate it.

Small-context models do not satisfy every auxiliary scene's configured minimum
context or output budget. Importing a model does not bypass scene capability
checks or imply that all long-document workflows are supported.

References: [Ollama OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility),
[Ollama model information](https://docs.ollama.com/api/show),
[Docker Desktop host networking](https://docs.docker.com/desktop/features/networking/networking-how-tos/).
