# Fork Notes

This repository is a fork of `rtk-ai/rtk` maintained for agent-driven development workflows where `rtk` is used as a transparent command proxy in front of common development tools.

The main goal of this fork is to keep the token-saving behavior of RTK while avoiding surprises for humans and AI agents. If a command works without `rtk`, it should work through `rtk`. If a command does not exist, `rtk` should report the same kind of error the original system would report.

## Why This Fork Exists

The upstream project is optimized for compact output, but agent workflows need stronger compatibility guarantees:

- Command output must stay recognizable to AI tools that diagnose failures.
- Argument forwarding must preserve the behavior of the original command.
- Unsupported or unknown subcommands must pass through instead of being blocked by RTK parsing.
- Missing tools must produce normal system `command not found` output, not RTK-specific wrapper errors.
- Modern JavaScript and Python stacks need first-class handling because they are common in real projects.

This matters because LLM agents often make decisions based on exact failure text. A wrapper error such as `rtk: Failed to run pytest: Failed to spawn process` is less useful than the original system hint, for example:

```text
Command 'pytest' not found, but can be installed with:
apt install python3-pytest
```

## What This Fork Changes

This fork keeps RTK's compact filtering model, but prioritizes transparent execution semantics.

Notable changes include:

- Critical fixes for `git` argument parsing and passthrough behavior.
- Support and fixes for modern JavaScript tooling such as `pnpm`, `vitest`, Next.js, TypeScript, Playwright, and Prisma.
- Python command transparency for tools such as `pytest` and `mypy`.
- Shared spawn-error handling that detects missing commands and returns `127` with normal command-not-found output.
- Linux integration with `/usr/lib/command-not-found` when available, so package installation hints are preserved.
- Reduced RTK-specific noise when a command cannot be resolved.

## Missing Command Behavior

The fork intentionally treats missing commands as a compatibility case, not an RTK failure.

### Before

When a supported command was not installed, RTK often replaced the original system error with an internal wrapper error.

Example with missing `pytest`:

```bash
pytest
```

Native shell output:

```text
Command 'pytest' not found, but can be installed with:
apt install python3-pytest
```

Old RTK output:

```text
rtk: Failed to run pytest: Failed to spawn process: No such file or directory (os error 2)
```

This was technically correct from RTK's point of view, but it was less useful. It hid the operating system's package hint and made the failure look like an RTK bug instead of a missing dependency.

Some Python commands also had another compatibility issue: RTK could silently try a different command when the requested tool was missing. For example, `rtk pytest` could fall back to `python -m pytest`. If `python` was missing or the module was unavailable, the user would see a different error than they would get from plain `pytest`.

That behavior is bad for agent workflows because the agent asked to run `pytest`, but the observed failure could mention `python`, module loading, or RTK internals instead.

### After

With this fork, `rtk` preserves the original failure shape for missing commands.

Example:

```bash
pytest
rtk pytest
```

Both should communicate the same root cause: `pytest` is not installed. `rtk pytest` should not replace that with an internal spawn error.

Expected RTK output on Debian/Ubuntu systems with `command-not-found` installed:

```text
Command 'pytest' not found, but can be installed with:
apt install python3-pytest
```

Expected fallback output when the system helper is not available:

```text
pytest: command not found
```

In both cases, the exit code should be `127`, matching normal command-not-found behavior.

For supported Python commands, this also means RTK should not silently rewrite a missing command into a different command. For example, if `pytest` is missing, `rtk pytest` should report missing `pytest`; it should not fall back to `python -m pytest` and report a different failure.

### Why This Matters

The output is not just cosmetic. AI coding agents use command output to decide what to do next.

Before this fork, an agent seeing this output:

```text
rtk: Failed to run pytest: Failed to spawn process: No such file or directory (os error 2)
```

might investigate RTK, retry with a different wrapper, or miss the correct installation command.

After this fork, the agent sees the same actionable hint as a human shell user:

```text
Command 'pytest' not found, but can be installed with:
apt install python3-pytest
```

That makes the next step obvious: install the missing package, switch to the right environment, or report that the dependency is unavailable.
