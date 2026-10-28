# ALCF Inference Service for agentic workflows

## TL;DR

DESC users, with a NERSC account, you can point a coding agent (OpenCode, Pi, Claude Code, Codex) at open-weight models hosted by ALCF, through an OpenAI- and Anthropic-compatible API with **no usage caps**. A NERSC account is all you need: authentication uses Globus with a NERSC identity.

The shortest path, using [uv](https://docs.astral.sh/uv/) and [opencode](https://opencode.ai/) installed:

```bash
uv tool install alcf-tokens && uv tool install alcf-ai   # or: pipx install alcf-ai alcf-tokens
alcf-tokens login inference                             # log in with your NERSC identity
alcf-tokens test-token inference                        # confirm the token is accepted
alcf-ai agent configure opencode                        # or: pi | claude | codex
opencode
```

And select the `inkling-bf16`, `muse-spark-bf16` or `nemotron-3-ultra` models from the **Minerva** cluster. It is the only cluster offering Chat Completions, Responses and Anthropic Messages APIs with 1M-token context models.

This can be done on your local machine, directly at NERSC or at CC-IN2P3.

Stuck? Send a direct message to **Patrick Wells** on the Discovery Alliance Slack workspace (see section 5).

## 1. Get access

Access to the inference service is obtained through the Globus compute portal, with NERSC as identity provider.

If you have a NERSC account, you have access. The service accepts Globus logins only from DOE-lab identity providers. `nersc.gov` is one of them, but a university or CNRS identity alone is rejected.

1. **Check.** Open [inference.alcf.anl.gov](https://inference.alcf.anl.gov/), choose *NERSC* as the identity provider and log in. If the chat UI loads, skip to section 2.
2. **If you are refused,** make sure you picked *NERSC* (not your university or CNRS) as the identity provider. If it still fails, see section 5. DESC members should also check the *Getting Started* tab pinned in the DESC channel.
3. **Link identities (optional).** If your everyday Globus account uses another identity, link your NERSC identity to it at [app.globus.org](https://app.globus.org/). Then log in the same way every time. Mixing unlinked identities causes `IdentityMismatchError` (section 6).

To hear about maintenance and new models, join the [service mailing list](https://mailman.cels.anl.gov/mailman3/lists/inference-service-notify.lists.alcf.anl.gov/).

## 2. Install and authenticate

Install two small CLIs (Python 3.10+). `alcf-tokens` handles Globus tokens, and `alcf-ai` handles model discovery and agent setup. They work the same on a laptop and on a Perlmutter or a CC-IN2P3 node.

```bash
# pick one
uv tool install alcf-tokens && uv tool install alcf-ai
pipx install alcf-ai alcf-tokens
pip install alcf-ai alcf-tokens        # inside an activated env
```

Prefer `uv tool` or `pipx`. They put both CLIs on your `PATH` in every shell, so the agent configuration from section 4 can call `alcf-tokens` to refresh tokens automatically and you never have to refresh them by hand. With a plain `pip` install, the CLIs exist only while that env is active, and an agent started from another shell cannot refresh its token.

Log in once, then verify:

```bash
alcf-tokens login inference         # inference scopes only; use your NERSC identity
alcf-tokens test-token inference    # should succeed
alcf-tokens get-token inference     # prints a bearer token for curl or other clients
```

| Item | Value |
| --- | --- |
| Access-token lifetime | 48 h, refreshed automatically by `get-token` |
| Forced re-login | every 30 days |
| Token cache | `~/.globus/app/7f3e61f5-…/alcf_tokens/tokens.json` |
| Reset | `alcf-tokens clear-tokens` |

On Perlmutter, `$HOME` is shared across login nodes, so a single login covers all of them. No browser opens there. Check on first use that the login prints a URL you can open on your laptop.

**Rule for agents:** have the agent fetch a fresh token through a helper command rather than storing a token string. A copied token stops working after 48 h, often in the middle of a session. This is handled by `alcf-ai agent configure`.

## 3. Pick a cluster and model

The recommendation is to use the Minerva cluster with `inkling-bf16` or `nemotron-3-ultra`. Both support tool calling and reasoning, have 1M-token context, speak all three API protocols, and stay hot so there is no cold start. The table lists only models with tool calling that suit agent work.

| Model | Cluster | Base URL path | APIs | Context (tokens) | Input | Always hot |
| --- | --- | --- | --- | --- | --- | --- |
| `inkling-bf16` | Minerva | `/minerva/api/v1` | chat, responses, messages | 1,048,576 | text, image | yes |
| `nemotron-3-ultra` | Minerva | `/minerva/api/v1` | chat, responses, messages | 1,048,576 | text | yes |
| `muse-spark-bf16` | Minerva | `/minerva/api/v1` | chat, responses, messages | 1,048,576 | text, image | no |
| `gpt-oss-120b` | Minerva | `/minerva/api/v1` | chat, responses, messages | 131,072 | text | no |

All paths sit under `https://inference-api.alcf.anl.gov/resource_server`. The full list of models is on [Available Models](https://docs.alcf.anl.gov/services/inference-endpoints/models/).

## 4. Configure your agent

The recommended route is `alcf-ai agent configure <agent>`. It writes the config, and when `alcf-tokens` is on your `PATH` it wires up automatic token refresh. Installing with `uv tool` or `pipx` (section 2) guarantees this. If neither CLI is on `PATH`, it embeds a token that expires after 48 h. You can re-run it at any time.

> [!WARNING]
> Proprietary harnesses (Claude Code and Codex) have limited support for third-party models. Pointing them at ALCF may stop you from using their first-party models (Claude, GPT) in that harness until you undo the change. Prefer OpenCode or Pi for ALCF models. If you do use Claude Code, the `claude-alcf` wrapper in the manual configuration below leaves your normal `claude` setup untouched.

```bash
alcf-ai agent configure opencode  # also: pi | claude | codex
alcf-ai agent configure opencode --include-experimental  # add non-whitelisted models
```

The defaults are model `inkling-bf16` and cluster `minerva`.  

To override the default model and cluster, use the `-m` and `-c` options. This is useful for Claude Code configuration which only specifies one model at a time.

```bash
alcf-ai agent configure claude -m nemotron-3-ultra -c minerva
```

**Want to configure your harnesses manually?** You can skip this if you ran `alcf-ai agent configure` above. It already writes these settings for you.
<details>
<summary>Click to expand the manual configurations</summary>

### OpenCode

OpenCode has first-class support for custom providers. Add this to `~/.config/opencode/opencode.jsonc`:

```json
{
  "provider": {
    "alcf-minerva": {
      "name": "ALCF Inference Service (Minerva)",
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "https://inference-api.alcf.anl.gov/resource_server/minerva/api/v1",
        "apiKey": "{env:ALCF_AI_TOKEN}"
      },
      "models": {
        "inkling-bf16":     { "name": "inkling-bf16",     "timeout": false, "limit": { "context": 1048576, "output": 0 } },
        "nemotron-3-ultra": { "name": "nemotron-3-ultra", "timeout": false, "limit": { "context": 1048576, "output": 0 } }
      }
    }
  }
}
```

The ALCF example targets Sophia with `openai/gpt-oss-120b` at 65,536 context. This version switches to Minerva for its longer context. Launch with a fresh token:

```bash
ALCF_AI_TOKEN="$(alcf-tokens get-token inference)" opencode
```

### Pi

Pi runs a shell command to obtain the key (the `!` prefix), so tokens never go stale. Add this to `~/.pi/agent/models.json`:

```json
{
  "providers": {
    "alcf-minerva": {
      "baseUrl": "https://inference-api.alcf.anl.gov/resource_server/minerva/api/v1",
      "api": "openai-completions",
      "apiKey": "!alcf-tokens get-token inference",
      "compat": { "supportsDeveloperRole": false, "supportsReasoningEffort": false },
      "models": [{ "id": "inkling-bf16" }, { "id": "nemotron-3-ultra" }]
    }
  }
}
```

### Claude Code

Claude Code needs a cluster with native Anthropic Messages support, which in practice means Minerva. Put a small wrapper on your `PATH` so each session starts with a fresh token and your normal `claude` setup stays untouched:

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/claude-alcf <<'EOF'
#!/usr/bin/env bash
export ANTHROPIC_BASE_URL="https://inference-api.alcf.anl.gov/resource_server/minerva/api/v1"
export ANTHROPIC_AUTH_TOKEN="$(alcf-tokens get-token inference)"
export ANTHROPIC_DEFAULT_OPUS_MODEL="${ALCF_MODEL:-inkling-bf16}"
export ANTHROPIC_DEFAULT_SONNET_MODEL="${ALCF_MODEL:-inkling-bf16}"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="${ALCF_MODEL:-inkling-bf16}"
exec claude "$@"
EOF
chmod +x ~/.local/bin/claude-alcf

claude-alcf                              # inkling-bf16
ALCF_MODEL=nemotron-3-ultra claude-alcf  # switch model
```

This wrapper is equivalent to the `env` block in `~/.claude/settings.json` that the ALCF docs show. Those docs hard-code `ANTHROPIC_AUTH_TOKEN`, which expires after 48 h, and the settings file applies to every Claude Code session. A session that outlives 48 h still needs a restart.

### Codex

Codex takes the most setup, so use it only if you need it specifically.

- Use Codex **≤ 0.94** if you want Chat Completions, since 0.95 dropped it. Against Minerva you can stay current and use `wire_api = "responses"`.
- Codex needs a generated `model_catalog_json` describing each model. ALCF provides a jq script in the [Agents guide](https://docs.alcf.anl.gov/services/inference-endpoints/agents/#codex), and you must re-run it after every Codex upgrade.
- The ALCF example uses two provider ids: `alcf-inference-service-minerva-api` in `config.toml` and `alcf_minerva` in the profile. Make them match.

```toml
# ~/.codex/config.toml
model_provider = "alcf_minerva"
model = "inkling-bf16"
model_catalog_json = "~/.codex/catalogs/inkling.json"

[model_providers.alcf_minerva]
name = "ALCF Inference Service (Minerva)"
base_url = "https://inference-api.alcf.anl.gov/resource_server/minerva/api/v1"
env_key = "ALCF_AI_TOKEN"
wire_api = "responses"
```

```bash
ALCF_AI_TOKEN="$(alcf-tokens get-token inference)" codex --disable apps --strict-config -c 'web_search="disabled"'
```

The ALCF example also sets `approval_policy = "never"` and `sandbox_mode = "danger-full-access"`. Leave those out unless you run Codex in a disposable environment.

### Other harnesses

Any client that speaks the OpenAI API can use the base URLs in section 3. If a harness sends requests the service rejects, put a translating shim in front of it: [alcf-proxy](https://alcf-proxy.readthedocs.io/en/latest) or [llm-rosetta](https://samf.sh/posts/2026/06/27).

</details>

## 5. Known issues and troubleshooting

Below is a list of known issues and fixes. If you run into a problem, contact:

- **Patrick Wells** (DESC member): send a direct message on the Discovery Alliance Slack workspace.
- The `#opencosmo` Slack channel, for general questions.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Login refused at the identity step | Non-DOE identity (university, CNRS) | Log in with your NERSC identity, or link it in Globus |
| `Permission denied` / 401 | Token expired, or the 30-day re-login is due | Log out at [app.globus.org/logout](https://app.globus.org/logout), then `alcf-tokens login inference` |
| `IdentityMismatchError` | Logged in with a Globus identity not linked to the earlier one | `alcf-tokens clear-tokens`, then log in again with one identity |
| Request hangs or times out | Model is queued or cold | `alcf-ai ls-jobs <cluster>`, or pick an always-hot model |
| HTTP 429 | Per-second request or tokens-per-minute limit | Wait, back off, send smaller requests ([rate limits](https://docs.alcf.anl.gov/services/inference-endpoints/api/#rate-limits)) |
| Agent fails on Sophia with Responses API | Responses streaming is disabled on Sophia (vLLM) | Use Chat Completions, or switch to Minerva |
| "Model started a function call but did not complete it" | SambaNova tool-call sanitization bug on Metis | Do not use Metis for agents |
| Codex rejects tool output | Tool calls must follow the OpenAI Harmony format | Use `gpt-oss-120b` or a Minerva model |
| Claude Code errors on Sophia or Metis | Neither serves the Anthropic Messages API | Use Minerva |

For anything else, contact the official [ALCF Support](https://docs.alcf.anl.gov/services/inference-endpoints/#contact-us).

## Acknowledgement and sources

If you publish work that used the service, include the Non-INCITE, Non-ALCC acknowledgement from the [ALCF Acknowledgement Policy](https://docs.alcf.anl.gov/policies/alcf-acknowledgement-policy/). Also cite the FIRST paper (Tanikanti et al., SC '25 Workshops, [doi:10.1145/3731599.3767346](https://doi.org/10.1145/3731599.3767346)).

### Sources

- [ALCF Inference Endpoints](https://docs.alcf.anl.gov/services/inference-endpoints/)
- [Agents](https://docs.alcf.anl.gov/services/inference-endpoints/agents/)
- [Available Models](https://docs.alcf.anl.gov/services/inference-endpoints/models/)
- [alcf-ai CLI and SDK](https://docs.alcf.anl.gov/services/inference-endpoints/alcf-ai/)
- [API Usage](https://docs.alcf.anl.gov/services/inference-endpoints/api/)
