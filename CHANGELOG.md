# Changelog

## 1.1.5

- Suppress the Windows BurntToast "Open Hermes" button when `open_url` uses an unregistered URL protocol.
- Record Windows notification click behavior as `open_url_button`, `open_url_unregistered`, or `none`.

## 1.1.4

- Document optional notification dependencies for macOS and Windows.

## 1.1.3

- Fix Windows notification argument passing so balloon fallback receives a non-empty message.

## 1.1.2

- Improve Windows notification diagnostics by recording PowerShell stdout/stderr in JSONL logs.
- Make the Windows balloon fallback more reliable when BurntToast is not installed.
- Record the concrete Windows notification method as `burnt_toast` or `windows_balloon`.

## 1.1.1

- Add `llm_wall_timeout` so slow auxiliary-model fallback does not block the approval prompt.
- Add provider-scoped `provider_extra_body`, including MiMo/Xiaomi defaults that disable thinking for short approval explanations without sending MiMo fields to other models.

## 1.1

- Add macOS and Windows desktop notifications for approval explanations.
- Add configurable terminal output with `print_to_stderr`.
- Add notification configuration for Hermes Desktop activation/deep-link attempts.
- Record notification delivery status in JSONL logs.

## 0.1.0

- Initial release.
- Explain dangerous Hermes approval requests in Chinese.
- Use Hermes plugin LLM access with fallback explanation.
- Cache accepted approval explanations per plugin session.
- Add local `config.yaml`.
