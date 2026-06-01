# Codex CLI `command not found` after update on Linux with `nvm`

## Scope

This document is a troubleshooting note for one specific scenario that was observed and resolved:

- OS: Linux
- Node.js managed with `nvm`
- Codex installed globally with `npm`
- Symptom: `codex` stopped working after an update or reinstall attempt

This is not general installation documentation for Codex CLI.

For current official installation and upgrade instructions, check OpenAI's Codex CLI docs:

- https://developers.openai.com/codex/cli

At the time of writing, OpenAI documents a standalone installer as the primary macOS/Linux setup path, although `npm install -g @openai/codex` is still supported for npm-based installs.

## Problem summary

In this case, the `codex` command stopped working in the terminal even though the global npm package still appeared to be installed.

Example symptom:

```bash
codex
```

Output:

```bash
Command 'codex' not found
```

At the same time, the package could still appear in the global npm list:

```bash
npm list -g --depth=0 | grep codex
```

Example:

```bash
├── @openai/codex@0.134.0
```

That means the package metadata may still be present while the executable link or native dependency is broken.

## What was verified

The following points are factual for the npm-based Codex CLI packaging used here:

- `@openai/codex` installs a `codex` command shim
- on Linux x64, the package depends on a platform-specific optional package such as `@openai/codex-linux-x64`
- a broken global npm install can leave the package present while the command is missing or unusable

This matches the package structure and known public issue reports in the Codex repository:

- https://github.com/openai/codex/blob/main/codex-rs/README.md
- https://github.com/openai/codex/issues/13555
- https://github.com/openai/codex/issues/9520

## Likely failure pattern

This document does not prove a single universal root cause, but it does describe a plausible and practical failure pattern for this environment:

1. Codex was installed globally with npm under an `nvm` Node.js version.
2. An update or reinstall left the global package in an inconsistent state.
3. The `codex` executable link in the `nvm` `bin` directory was missing, incorrect, or temporarily renamed.
4. In some cases, the platform-specific optional dependency was also missing.
5. A normal reinstall could fail if temporary npm directories were left behind.

Observed examples that fit this pattern:

- `codex` not found in `PATH`
- temporary entries such as `.codex-*` under the npm global package area
- reinstall errors such as `ENOTEMPTY`
- runtime error: `Missing optional dependency @openai/codex-linux-x64`

## How to diagnose it

### 1. Check whether the shell can find `codex`

```bash
command -v codex
```

If nothing is returned, the shell is not finding the executable.

### 2. Check whether the npm package still exists

```bash
npm list -g --depth=0 | grep codex
```

If `@openai/codex` appears here, the package may still exist even if the command is broken.

### 3. Find the global npm install root

```bash
npm root -g
```

Example with `nvm`:

```bash
/home/USER/.nvm/versions/node/v22.14.0/lib/node_modules
```

### 4. Confirm that the corresponding `bin` directory is in `PATH`

```bash
echo "$PATH"
```

Look for the matching `nvm` path, for example:

```bash
/home/USER/.nvm/versions/node/v22.14.0/bin
```

If the `nvm` bin directory is present in `PATH`, the problem is more likely the Codex install itself than shell configuration.

### 5. Inspect the `codex` link in the `bin` directory

Adjust the path to your Node version:

```bash
ls -la /home/USER/.nvm/versions/node/v22.14.0/bin | grep codex
```

Expected pattern:

```bash
codex -> ../lib/node_modules/@openai/codex/bin/codex.js
```

If the final `codex` link is missing, or if only temporary names such as `.codex-XXXX` exist, the install is likely incomplete.

## Errors that may appear

### `codex: command not found`

This means the shell did not find a `codex` executable in `PATH`.

### `npm error code ENOTEMPTY`

Example pattern:

```bash
npm error code ENOTEMPTY
npm error syscall rename
npm error ENOTEMPTY: directory not empty, rename ...
```

This usually means npm tried to rename or replace a directory but found leftover temporary contents from an earlier failed operation.

### Missing optional dependency

Example:

```bash
Error: Missing optional dependency @openai/codex-linux-x64.
Reinstall Codex: npm install -g @openai/codex@latest
```

This means the top-level package was found, but the platform-specific native package needed at runtime was not available.

## Recovery procedure that solved this case

This was the practical fix for the broken npm-based install:

1. Remove the broken global Codex package directory.
2. Remove leftover temporary `.codex-*` directories.
3. Remove the broken or missing `codex` link from the matching `nvm` `bin` directory.
4. Reinstall `@openai/codex`.
5. Clear the shell command hash and verify the command again.

If you use `nvm`, do not use `sudo` with `npm install -g`, because that can create permission and ownership conflicts inside the `nvm` tree.

### Cleanup commands

Adjust the Node version path to match your environment:

```bash
rm -rf /home/USER/.nvm/versions/node/v22.14.0/lib/node_modules/@openai/codex
rm -rf /home/USER/.nvm/versions/node/v22.14.0/lib/node_modules/@openai/.codex-*
rm -f /home/USER/.nvm/versions/node/v22.14.0/bin/codex
rm -f /home/USER/.nvm/versions/node/v22.14.0/bin/.codex-*
```

### Reinstall

```bash
npm install -g @openai/codex@latest
```

Note: `npm cache clean --force` can be tried if you suspect npm cache issues, but npm's own documentation says cache cleaning is usually unnecessary. It should be treated as optional, not as a default required step.

### Refresh shell command lookup

```bash
hash -r
```

### Verify

```bash
codex --version
command -v codex
```

Expected result:

- `codex --version` returns a valid version string
- `command -v codex` returns the executable path under the active `nvm` Node version

## Why this qualifies as a solved problem

This document describes a real troubleshooting path rather than a theory:

- the symptom is concrete
- the environment is scoped
- the package behavior is verifiable
- the failure modes are consistent with known Codex/npm issues
- the recovery procedure is specific
- success can be confirmed with `codex --version` and `command -v codex`

That makes it a valid problem-solution document for people who hit the same npm-based Linux + `nvm` failure mode.

## What this document does not claim

This document does not claim that:

- every Codex update failure has the same cause
- npm is the preferred install method for all users
- cleaning the npm cache is always required
- this exact fix applies to Homebrew, the standalone installer, Windows, or non-`nvm` setups

## Short version

If `codex` disappears from the terminal on Linux after an npm-managed update, but `@openai/codex` still appears in `npm list -g`, a broken global install is a credible explanation.

For the npm + `nvm` scenario, manually removing the broken package directory, removing leftover `.codex-*` temporary entries, reinstalling `@openai/codex`, and verifying the restored binary path is a practical and technically defensible fix.