# Bring Your Own Key (BYOK)

Use this page when you want to run an OpenAB agent backend against your own API
key or model provider. It collects the BYOK paths that are currently
documented as OpenAB best practice, along with the caveats that matter before
you deploy.

The model names in this page are **validated examples**, not an exhaustive
allowlist. BYOK compatibility is backend-specific and can change with the
backend CLI or image version, so re-verify against the exact version you
deploy.

## Current Status

| Agent backend | BYOK status in OpenAB | Notes |
|---------------|------------------------|-------|
| Kiro CLI | No validated OpenAB BYOK ACP path yet | The current OpenAB Kiro flow uses OAuth login, not provider API keys. OpenAB has not yet validated a Kiro BYOK ACP path; public Kiro BYOK/custom-model requests are still open ([#9367](https://github.com/kirodotdev/Kiro/issues/9367), [#8194](https://github.com/kirodotdev/Kiro/issues/8194)). |
| Codex | Validated for documented examples | OpenRouter and Azure OpenAI-style endpoints were validated end-to-end through OpenAB ACP for the documented examples below. |
| GitHub Copilot CLI | Not yet validated | Upstream BYOK exists, and non-ACP CLI mode worked in our testing, but `copilot --acp --stdio` returned `Authentication required` at `session/new` in Copilot CLI 1.0.59-1.0.60. OpenAB needs ACP, so do not treat Copilot BYOK as validated in OpenAB yet. |

The status table tracks both validated and not-yet-validated backends. The
concrete examples later in this page are limited to validated paths.

## What OpenAB Does

OpenAB does **not** translate provider settings between CLIs. In a BYOK setup,
OpenAB only:

- starts the backend CLI
- forwards selected environment variables to the child process

In a typical Helm deployment, the chart and container image provide the rest:

- `workingDir` and container `HOME`
- the backend-specific config path under `$HOME`
- PVC-backed persistence when `persistence` is enabled

So a working BYOK setup usually needs four pieces:

1. Put the provider key in the OpenAB pod.
2. Forward only the needed key to the backend subprocess.
3. Write the backend-specific config under `$HOME`.
4. Verify the backend CLI directly before debugging OpenAB or ACP.

## Current Kiro CLI Status

OpenAB has not yet validated a Kiro CLI ACP flow that uses provider API keys or
custom endpoints.

- The current [Kiro guide](kiro.md) assumes a one-time OAuth login.
- We do not yet have a validated OpenAB flow for custom provider endpoints or
  provider API keys with Kiro ACP.
- The public Kiro repo still has open BYOK / custom-model requests:
  - [kirodotdev/Kiro#9367](https://github.com/kirodotdev/Kiro/issues/9367)
  - [kirodotdev/Kiro#8194](https://github.com/kirodotdev/Kiro/issues/8194)
- Those issues are supporting context only; the actual OpenAB docs claim here is
  simply that this path is not yet validated.

Until that changes, do **not** present Kiro BYOK as working in OpenAB.

## Currently Validated Backend: Codex

The concrete examples below use Codex because that is the backend we validated
end-to-end with OpenAB ACP in June 2026. This page is intentionally
version-light because provider compatibility changes quickly; re-verify against
the exact image you deploy. At the time of writing, `Dockerfile.codex` in this
repo pins Codex CLI `0.137.0` and `codex-acp` `0.15.0`. For base image, auth
flow, persistence, and sandbox behavior, see [Codex](codex.md).

## Shared Helm Pattern

This is a recommended Codex base layout for the BYOK examples below. Merge it
with your existing Codex deployment such as the one shown in [Codex](codex.md).

These examples assume the chart default `workingDir` / `HOME`:

```yaml
agents:
  kiro:
    enabled: false
  codex:
    enabled: true
    image: ghcr.io/openabdev/openab-codex:latest
    command: codex-acp
    workingDir: /home/node
    envFrom:
      - secretRef:
          name: codex-provider-secrets
    inheritEnv:
      - <PROVIDER_ENV_VAR_NAME>
    persistence:
      enabled: true
```

With that default layout, Codex reads its config from:

```text
/home/node/.codex/config.toml
```

If you override `workingDir`, adjust the paths in this guide to match your
chosen `$HOME`.

`envFrom` makes keys visible to the OpenAB container, but only names in
Helm `inheritEnv` reach the spawned Codex process. The chart uses that value to
generate the `[agent].inherit_env` key in the `config.toml` it mounts for the
agent. If you use the chart's `secretEnv` field instead of `envFrom`, the chart
also adds those names to the generated `inherit_env`, so you do not need to
repeat them in `inheritEnv`.
Alternatively, for non-secret values or local testing, you can pass values
directly through `[agent].env`, but note that this stores plaintext values in
the generated `config.toml` / ConfigMap rather than sourcing them from a Secret.

For Codex, the provider config lives at `$HOME/.codex/config.toml`.

## OpenRouter (Codex example)

OpenRouter works with Codex when the chosen model supports the OpenAI Responses
API.

### Secret and Helm Values

Assume your base Codex deployment already handles Discord bot credentials. This
overlay only adds the provider key:

```bash
kubectl create secret generic codex-openrouter-provider \
  --from-literal=OPENROUTER_API_KEY="$OPENROUTER_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -
```

```yaml
agents:
  codex:
    envFrom:
      - secretRef:
          name: codex-openrouter-provider
    inheritEnv:
      - OPENROUTER_API_KEY
```

### Codex Config

Merge the provider-specific block below into `$HOME/.codex/config.toml`. If you
manage Codex config via a mounted ConfigMap, update that source file instead of
editing the in-pod file directly.

```toml
model = "YOUR_OPENROUTER_MODEL"
model_provider = "openrouter"

[model_providers.openrouter]
name = "OpenRouter"
base_url = "https://openrouter.ai/api/v1"
wire_api = "responses"
env_key = "OPENROUTER_API_KEY"
```

Notes:

- `https://openrouter.ai/api/v1` is the correct base URL for Codex. Do **not**
  point Codex at `/chat/completions`.
- `openai/gpt-oss-120b:free` is one validated end-to-end example in OpenAB, but
  it is **not** the only model that may work.
- Other OpenRouter models may work too if they support the Responses API and
  your Codex build can use them.
- Paid models require credits, and not every OpenRouter model supports the
  Responses API.
- OpenRouter currently documents its Responses API as beta upstream, so
  re-verify behavior when changing models or upgrading your Codex image.

## Azure OpenAI / Azure AI Foundry (Codex example)

Use an Azure OpenAI-style endpoint that exposes the OpenAI Responses API under
`/openai/v1`.

### Secret and Helm Values

Assume your base Codex deployment already handles Discord bot credentials. This
overlay only adds the provider key:

```bash
kubectl create secret generic codex-azure-provider \
  --from-literal=AZURE_OPENAI_API_KEY="$AZURE_OPENAI_API_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -
```

```yaml
agents:
  codex:
    envFrom:
      - secretRef:
          name: codex-azure-provider
    inheritEnv:
      - AZURE_OPENAI_API_KEY
```

### Codex Config

Merge the provider-specific block below into `$HOME/.codex/config.toml`. If you
manage Codex config via a mounted ConfigMap, update that source file instead of
editing the in-pod file directly.

```toml
model = "YOUR_AZURE_DEPLOYMENT_NAME"
model_provider = "azure"
model_reasoning_effort = "xhigh"

[model_providers.azure]
name = "Azure OpenAI"
base_url = "https://YOUR-AZURE-HOST/openai/v1"
wire_api = "responses"
env_key = "AZURE_OPENAI_API_KEY"
```

Notes:

- Replace `YOUR-AZURE-HOST` with the exact host for your Azure deployment.
- Replace `YOUR_AZURE_DEPLOYMENT_NAME` with your Azure deployment name. The
  validated deployment used the name `gpt-5.4`.
- Common host forms include `YOUR-RESOURCE.openai.azure.com`,
  `YOUR-RESOURCE.cognitiveservices.azure.com`, and
  `YOUR-PROJECT.services.ai.azure.com`.
- Keep the base URL rooted at `/openai/v1`.
- The validated example used an Azure deployment named `gpt-5.4`, but that is
  **not** the only expected working choice.
- Other Azure deployment names may work as long as the deployed model is
  available through Azure's OpenAI Responses API and your Codex build can use
  it.
- `model_reasoning_effort = "xhigh"` matches the validated deployment. Lower
  values may also work if you prefer lower cost or latency.

## Verify Codex Before Debugging OpenAB or ACP

After updating the Secret and `config.toml`, verify Codex inside the pod before
debugging OpenAB:

```bash
kubectl exec -it deployment/<your-codex-deployment> -- sh -lc \
  'cd "$HOME" && codex exec --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "Reply with exactly CODEX_OK"'
```

The `--dangerously-bypass-approvals-and-sandbox` flag is recommended only for
manual smoke tests inside an already isolated pod. In Kubernetes, nested Codex
sandboxing can fail because of `bwrap` or user-namespace restrictions; see
[Codex](codex.md#troubleshooting).

This validates provider auth/config inside the container, but it does **not** by
itself prove that OpenAB's spawned ACP child process received the key through
`inheritEnv`, `secretEnv`, or `[agent].env`.

After `codex exec` succeeds, also verify a real OpenAB turn or another ACP turn
launched through your deployed OpenAB backend before treating the setup as
complete.

If this fails, fix the Codex or provider config first. OpenAB cannot compensate
for a broken provider configuration.

## Current GitHub Copilot CLI Status

GitHub documents Copilot CLI BYOK upstream, and Copilot ACP support is still in
public preview:

- [Copilot CLI now supports BYOK and local models](https://github.blog/changelog/2026-04-07-copilot-cli-now-supports-byok-and-local-models/)
- [ACP support in Copilot CLI is now in public preview](https://github.blog/changelog/2026-01-28-acp-support-in-copilot-cli-is-now-in-public-preview/)
- [Using BYOK models in Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-byok-models)
- [Authenticating GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/authenticate-copilot-cli)

In our OpenAB validation:

- Copilot CLI **non-ACP CLI mode** BYOK worked.
- Copilot CLI **ACP mode** did not: `copilot --acp --stdio` returned
  `Authentication required` at `session/new`.
- This was reproduced with Copilot CLI 1.0.59-1.0.60.

Because OpenAB requires ACP, do **not** currently document Copilot CLI BYOK as
validated in OpenAB until that ACP path is validated.

## Switching Providers with Existing Sessions

OpenAB persists Discord thread-to-ACP session mappings in
`$HOME/.openab/thread_map.json`. If you move a live bot from one provider to
another, existing threads may still try to resume old ACP sessions.

Before testing the new provider, use one of these:

- run `/reset` in the Discord thread
- start a fresh thread
- or clear `$HOME/.openab/thread_map.json` before restarting the pod

Clearing `thread_map.json` resets persisted thread-to-session mappings for the
whole deployment, not just one thread.

If `codex exec` works but an existing Discord thread fails right after a
provider change, suspect stale session reuse before suspecting auth.

If you are using another adapter, apply the same principle: avoid reusing old
ACP sessions across provider switches.

## Troubleshooting

- Updating a local values file is not enough; apply the Helm change and restart
  or roll out the pod.
- The warning `Model metadata for ... not found` comes from **Codex's local
  model metadata catalog**, not from the OpenRouter API. If requests succeed,
  treat it as cosmetic rather than an auth failure.

## Related Docs

- [Codex](codex.md)
- [GitHub Copilot CLI](copilot.md)
- [Configuration Reference](config-reference.md)
- [Secrets Management](secrets-management.md)
