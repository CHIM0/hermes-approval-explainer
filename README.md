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
- Sends OS desktop notifications on macOS and Windows when available.
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
llm_wall_timeout: 8
provider: ""
model: ""
provider_extra_body:
  xiaomi:
    thinking:
      type: disabled
    top_p: 0.95
include_session_metadata: false
skip_repeated_after_accept: true
print_to_stderr: true

desktop_notification:
  enabled: true
  title: "Hermes 请求授权"
  open_url: "hermes://"
  message_chars: null
  fallback_to_stderr: true
  macos:
    prefer_terminal_notifier: true
    activate_bundle_id: "com.nousresearch.hermes"
  windows:
    prefer_burnt_toast: true
    toast_app_logo: "assets/hermes-logo.png"
```

`provider` and `model` are optional. Empty values use Hermes' active model.

`provider_extra_body` applies request-body extras only when the active or
configured provider matches the key. The default `xiaomi` entry disables MiMo
thinking for this short approval-explanation call and adds
`max_completion_tokens` from `max_tokens` at runtime. Other providers, including
DeepSeek, do not receive these MiMo-specific fields.

`timeout` is passed to Hermes' plugin LLM call. `llm_wall_timeout` is the
plugin's own hard wall-clock limit for the whole explanation call. If the LLM
or Hermes auxiliary fallback chain takes longer, the plugin immediately uses
the local rule-based fallback explanation so the approval prompt is not delayed.

`print_to_stderr` controls whether the full explanation is printed to the
terminal. Set it to `false` if you want desktop notifications and logs only.

`desktop_notification.enabled` controls OS notifications. On macOS, the plugin
uses `terminal-notifier` when it is installed so clicking the notification can
activate Hermes Desktop. It falls back to built-in `osascript` notifications
when `terminal-notifier` is unavailable, but those fallback notifications do
not provide a reliable click action.

`desktop_notification.message_chars: null` means the plugin sends the full
explanation to the OS notification backend. The operating system may still fold
or visually truncate long notifications. Set `message_chars` to a positive
integer if you want a plugin-side length limit.

Install the recommended macOS notification backend:

```bash
brew install terminal-notifier
```

On Windows, the plugin uses PowerShell. If the optional `BurntToast` module is
installed, the notification includes an "Open Hermes" protocol button using
`open_url` only when that URL protocol is registered in Windows. Without
BurntToast, it falls back to a basic Windows balloon notification without a
reliable click action.

Install the recommended Windows notification backend:

```powershell
Install-Module BurntToast -Scope CurrentUser -Force
```

The bundled `assets/hermes-logo.png` is used as the BurntToast app logo by
default. Set `desktop_notification.windows.toast_app_logo` to another absolute
path, relative plugin path, or an empty string if you want Windows to use its
default notification icon.

`open_url` defaults to `hermes://`, which can wake the registered Hermes
Desktop app. Hermes Desktop currently does not expose a public deep link for a
specific approval request, so the notification opens the app rather than
jumping to a specific approval row.

If clicking the Windows notification says no app can open `hermes://`, Hermes
Desktop has not registered that URL protocol on that machine. The plugin will
still show the notification, but it will suppress the broken button and record
`click_action: "open_url_unregistered"` in the JSONL log.

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

`pre_approval_explanation` also records notification status, including the
method used and whether the OS notification command succeeded.

## WebUI Note

In the Hermes CLI/TUI, explanations are printed to stderr before the normal
approval prompt.

In Hermes Desktop, the plugin can additionally send a system notification. This
does not modify Hermes Desktop or Hermes core, so the explanation is not
embedded inside the Desktop approval card. The notification can open or
activate Hermes Desktop when the OS notification backend supports a click
action.

Independent WebUIs such as `nesquena/hermes-webui` usually display only the
structured Hermes approval payload, not plugin stderr. For those WebUIs, this
plugin still writes JSONL logs and may send OS notifications, but the approval
card will not show the explanation unless the WebUI is extended to read the log
or Hermes core adds an `explanation` field to the approval payload.

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
