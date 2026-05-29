# Hermes Approval Explainer

A Hermes Agent plugin that explains dangerous command approval requests in
Chinese before Hermes asks the user to accept or deny.

It is designed for users who do not read shell commands comfortably. The plugin
does **not** approve, deny, block, or bypass any command. Hermes still owns the
approval decision and the user still responds to the normal approval prompt.

## Features

- Explains why the agent wants to run the tool or command.
- Explains why Hermes classified the action as dangerous.
- Shows a short path impact summary when paths can be detected.
- Uses Hermes' active/default model through `ctx.llm.complete(...)`.
- Falls back to a local rule-based explanation if the LLM call fails.
- Skips repeated explanations after the user accepted an equivalent approval in
  the same plugin session.
- Logs explanations and approval decisions to JSONL.

## Installation

Install directly from GitHub:

```bash
hermes plugins install CHIM0/hermes-approval-explainer --enable
```

Or install from a full Git URL:

```bash
hermes plugins install https://github.com/CHIM0/hermes-approval-explainer --enable
```

Then restart Hermes / the Hermes gateway so hooks are loaded.

For local development, copy or symlink this directory into:

```text
~/.hermes/plugins/approval-explainer/
```

Then enable it:

```bash
hermes plugins enable approval-explainer
```

## Configuration

Edit:

```text
~/.hermes/plugins/approval-explainer/config.yaml
```

Default config:

```yaml
language: "简体中文"
bullet_count: "3-5"
max_bullet_chars: 45
max_tokens: 260
temperature: 0.1
timeout: 8
provider: ""
model: ""
include_session_metadata: false
skip_repeated_after_accept: true
```

`provider` and `model` are optional. Empty values use Hermes' active model.

If you pin a provider or model, Hermes requires an explicit trust gate in
`~/.hermes/config.yaml`:

```yaml
plugins:
  entries:
    approval-explainer:
      llm:
        allow_provider_override: true
        allowed_providers: ["deepseek"]
        allow_model_override: true
        allowed_models: ["deepseek-v4-flash"]
```

## Repeated Approval Explanations

When `skip_repeated_after_accept: true`, the plugin suppresses repeated
explanations after the user accepts a matching approval in the current plugin
session.

This only skips the explanation. It does not skip Hermes' approval prompt.

The dedupe key is based on:

```text
session_key + pattern_keys + command verb + normalized paths
```

So these are treated as equivalent:

```bash
rm -rf dist/
rm -rf ./dist
```

But this is different:

```bash
rm -rf ~/.ssh/
```

`deny` and `timeout` are not cached, so the next matching approval will still be
explained.

## Logs

The plugin writes JSONL logs to:

```text
~/.hermes/logs/approval-explainer.jsonl
```

Events include:

- `pre_approval_explanation`
- `pre_approval_explanation_skipped`
- `post_approval_response`
- `config_error`

## WebUI Note

In the Hermes CLI/TUI, explanations are printed to stderr before the normal
approval prompt.

Independent WebUIs such as `nesquena/hermes-webui` usually display only the
structured Hermes approval payload, not plugin stderr. For those WebUIs, this
plugin still writes JSONL logs, but the approval card will not show the
explanation unless the WebUI is extended to read the log or Hermes core adds an
`explanation` field to the approval payload.

## Slash Command

The plugin registers:

```text
/approval-explainer-last
```

It shows the most recent explanation generated in the current plugin process.

## Privacy

The plugin sends a compact, redacted context object to Hermes' active model:

- command
- Hermes risk description and pattern keys
- current working directory
- latest user request
- model intent before the tool call, when Hermes exposes it
- tool call summary
- path summaries

It does not send the full conversation transcript.

## License

MIT
