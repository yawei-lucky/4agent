# WSL Codex Proxy Login Notes

## Background

The Codex / VS Code extension runs through this environment chain:

```text
Windows
  -> Codex / VS Code extension
  -> WSL
  -> OpenAI services
```

A previous login issue was confirmed and fixed: the Codex extension can now log in through the WSL proxy.

## Symptom

The machine originally showed this mismatch:

- WSL itself was connected normally.
- A regular interactive WSL terminal could use the proxy.
- The Codex extension still reported a region restriction.
- Direct access to OpenAI services from mainland China failed.

The proxy configuration lived in:

```text
/home/yawei/.bashrc
```

and was enabled by this function:

```bash
wsl_proxy_on
```

However, `.bashrc` had an interactive-shell guard near the top:

```bash
case $- in
    *i*) ;;
      *) return;;
esac
```

That created this behavior difference:

```text
Manual WSL terminal
  -> interactive Bash
  -> .bashrc continues
  -> wsl_proxy_on runs
  -> proxy variables are available

Codex extension WSL agent
  -> non-interactive Bash
  -> .bashrc returns early
  -> wsl_proxy_on does not run
  -> no proxy variables
  -> region restriction / login failure
```

Checks confirmed the diagnosis:

```text
Non-interactive WSL: proxy variables were missing
Interactive WSL: http_proxy, https_proxy, and related variables existed
```

## Fix

Copy the existing proxy environment settings into a Codex-specific environment file:

```text
/home/yawei/.codex-wsl-env
```

Set its permissions to user-only read/write:

```bash
chmod 600 /home/yawei/.codex-wsl-env
```

Then update:

```text
/home/yawei/.profile
```

with this login-shell loader:

```bash
# Load proxy environment for non-interactive Codex WSL agent
if [ -f /home/yawei/.codex-wsl-env ]; then
    . /home/yawei/.codex-wsl-env
fi
```

This gives the Codex extension the proxy variables through WSL's non-interactive login shell:

```text
Codex extension
  -> WSL non-interactive login shell
  -> reads ~/.profile
  -> sources ~/.codex-wsl-env
  -> gets proxy variables
  -> connects and logs in normally
```

## Important Implementation Notes

- Do not print status banners from `/home/yawei/.codex-wsl-env`.
- Avoid output such as `WSL proxy enabled: ...` from files sourced by the Codex agent.
- Extra stdout during agent startup can interfere with extension communication.
- Keep the original `/home/yawei/.bashrc` behavior intact for manual interactive WSL terminals.
- Use a separate Codex-specific file so the non-interactive agent can load only the required environment variables.
- Keep `/home/yawei/.codex-wsl-env` at permission mode `600` because it may contain local proxy details.

## Backup

Before changing `.profile`, keep a timestamped backup. The confirmed backup from this fix was:

```text
/home/yawei/.profile.codex-backup-20260712
```

## Validation

The final checks showed:

```text
Non-interactive WSL: LOGIN_PROXY_OK
Interactive WSL: INTERACTIVE_PROXY_OK
HTTPS network request: target server reachable
Codex extension: login succeeded
```

## Why Not Git Bash

Git Bash is a Windows MSYS2 environment, while the Codex agent runs inside WSL through `/bin/bash`.

Environment variables configured in Git Bash are not automatically inherited by WSL. Therefore, the fix must be applied to WSL's own non-interactive startup path rather than to Git Bash.
