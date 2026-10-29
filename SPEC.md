# sshp: SSH & Git identity manager

> Working name: `sshp`. Rename freely (check that the name is free on GitHub, crates.io/pkg.go.dev, and Homebrew before you commit to it).

## GitHub "About" text (copy-paste)

**Description:** Manage multiple SSH keys and Git identities (personal, work, clients) from one CLI. Generates `~/.ssh/config`, rewrites remotes, and sets the right `user.email` per repo.

**Topics:** `ssh`, `git`, `github`, `gitlab`, `cli`, `multi-account`, `ssh-config`, `developer-tools`, `golang`

---

## 1. Problem

Developers who use more than one Git account (personal GitHub, company GitHub, GitLab, client repos) hit the same problems:

- Hand-editing `~/.ssh/config` with one `Host` alias per account is error-prone.
- Cloning with the wrong URL (`github.com` instead of the alias) silently uses the wrong key.
- Commits get authored with the wrong `user.name` / `user.email`.
- `ssh-agent` offers every key to every server, causing "too many authentication failures" or pushes under the wrong account.
- Nobody remembers which key belongs to which account.

## 2. Goals

- One command to create a profile: generate a key, write the SSH config, and print the public key to paste into GitHub.
- One command to apply a profile to the current repo (remote URL + local git identity).
- Safe by default: never destroys the user's existing config; never reads or prints private keys.
- Cross-platform: Linux, macOS, Windows (OpenSSH).
- Single static binary, no runtime dependencies.
- Readable, diff-friendly output, so users can still understand and edit the generated config.

## 3. Non-goals

- Not an SSH agent or a secrets vault.
- Not a replacement for `gh` / `glab` (no API calls in v1, no OAuth).
- Not a general SSH connection manager (bastions, tunnels). Focus on Git identities first.

## 4. Concepts

**Profile**: a named identity.

| Field | Example | Notes |
|---|---|---|
| `name` | `work` | Unique, used as CLI argument |
| `alias` | `github-work` | SSH `Host` alias (defaults to `<provider>-<name>`) |
| `provider` | `github` | `github`, `gitlab`, `bitbucket`, `custom` |
| `hostname` | `github.com` | Defaults from provider |
| `user` | `git` | Defaults from provider |
| `identity_file` | `~/.ssh/github-work` | Private key path |
| `git_name` | `Cung Huynh` | Optional |
| `git_email` | `cung@company.com` | Optional |
| `port` | `22` | Optional |

**Managed block**: `sshp` only edits the section between these markers in `~/.ssh/config`. Everything outside is left untouched.

```sshconfig
# >>> sshp managed block (do not edit by hand) >>>
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/github-work
    IdentitiesOnly yes
# <<< sshp managed block <<<
```

**Source of truth**: `~/.config/sshp/profiles.toml` (Windows: `%APPDATA%\sshp\profiles.toml`). The SSH config block is rendered from it.

```toml
[[profile]]
name = "work"
provider = "github"
identity_file = "~/.ssh/github-work"
git_name = "Cung Huynh"
git_email = "cung@company.com"
```

## 5. CLI commands

| Command | Description |
|---|---|
| `sshp init` | Create config dir, scan existing `~/.ssh` and offer to import. |
| `sshp add <name>` | Interactive wizard: provider, key (generate new ed25519 or use existing), git identity. Prints the public key and the provider's "add SSH key" URL. |
| `sshp list` | Table of profiles: name, alias, host, key path, key exists?, loaded in agent? |
| `sshp edit <name>` | Change fields of a profile. |
| `sshp remove <name>` | Remove from config. `--delete-key` to also remove key files (asks for confirmation). |
| `sshp import` | Parse existing `~/.ssh/config` and `*.pub` files into profiles. Non-destructive. |
| `sshp apply` | Re-render the managed block (idempotent). `--dry-run` shows a diff. |
| `sshp use <name>` | In the current repo: rewrite `origin` to the profile alias and set local `user.name` / `user.email`. |
| `sshp clone <name> <url>` | Clone with the URL rewritten to use the profile alias, then apply identity. |
| `sshp link <dir> <name>` | Bind a directory (e.g. `~/work`) to a profile via git `includeIf "gitdir:"`, so every repo under it uses that identity automatically. |
| `sshp whoami` | Which profile and key the current repo would use, and which account the server reports. |
| `sshp test [name]` | Run `ssh -T` against each profile and show the authenticated username. |
| `sshp pubkey <name>` | Print the public key. `--copy` copies to clipboard. |
| `sshp doctor` | Diagnose: key permissions, missing key files, duplicate aliases, repos using `github.com` directly, unrelated `Host *` rules overriding things, agent state. |

Global flags: `--dry-run`, `--yes`, `--no-color`, `--config <path>`, `--verbose`.

## 6. Behavior details

1. **Backup**: before any write to `~/.ssh/config` or a `.gitconfig`, save `<file>.sshp-backup-<timestamp>` (keep last 5).
2. **Atomic writes**: write to a temp file, then rename.
3. **Key generation**: shell out to `ssh-keygen -t ed25519 -C "<email>" -f <path>`, or use Go's `x/crypto/ssh` if `ssh-keygen` is not found. Passphrase is optional and prompted by `ssh-keygen`/library; never stored.
4. **Permissions**: ensure `~/.ssh` is `700` and private keys `600` (on Windows, check/repair ACLs via `icacls`).
5. **Never read private key contents** beyond what's needed to verify the file exists and has correct permissions. Never log or print them.
6. **`IdentitiesOnly yes`** is always set so the agent doesn't offer other keys.
7. **URL rewrite** supports `git@github.com:org/repo.git` and `https://github.com/org/repo.git` and converts to `git@<alias>:org/repo.git`.
8. **Idempotency**: running any command twice produces the same result.
9. **Conflicts**: if a `Host` alias already exists outside the managed block, warn and refuse unless `--force`.
10. **Exit codes**: `0` success, `1` generic error, `2` usage error, `3` environment problem (missing ssh, bad permissions).

## 7. Suggested tech stack

- **Language**: Go (single binary, easy cross-compilation, good Windows support). Rust is a fine alternative.
- **CLI**: `spf13/cobra`; prompts via `charmbracelet/huh` or `survey`; tables via `lipgloss`/`tablewriter`.
- **Config**: `pelletier/go-toml/v2`.
- **SSH config parsing** (for `import`/`doctor`): `kevinburke/ssh_config` or a small custom parser.
- **Git operations**: shell out to `git config`, do not reimplement.
- **Release**: GoReleaser, Homebrew tap, Scoop bucket (Windows), `.deb`/`.rpm`, GitHub Actions.
- **Tests**: table-driven unit tests; integration tests using a temp `$HOME`; golden files for rendered config.

### Suggested layout

```
sshp/
├── cmd/sshp/main.go
├── internal/
│   ├── profile/        # model, validation, TOML store
│   ├── sshconfig/      # render + managed-block editing + parser
│   ├── gitcfg/         # git config / includeIf / remote rewrite
│   ├── keys/           # keygen, permissions, fingerprint
│   ├── doctor/         # checks
│   └── ui/             # prompts, tables, colors
├── testdata/
├── docs/
├── .github/workflows/  # CI: test, lint, release
├── .goreleaser.yaml
├── LICENSE             # MIT or Apache-2.0
├── README.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
└── SECURITY.md
```

## 8. Milestones

**v0.1 (MVP)**
- Profile model + TOML store
- `add`, `list`, `remove`, `apply`, `pubkey`
- Managed block rendering with backup and atomic write
- Linux/macOS/Windows CI

**v0.2**
- `use`, `clone`, `whoami`, `test`
- URL rewriting for SSH and HTTPS

**v0.3**
- `import` of existing config, `doctor`
- `link` with `includeIf`

**v0.4**
- Shell completions (bash/zsh/fish/pwsh)
- Homebrew + Scoop packaging
- GitLab/Bitbucket provider presets

**v1.0**
- Stable config format, docs site, full test coverage on config editing

**Ideas for later**: TUI mode, `ssh-add` integration with expiry, 1Password/Bitwarden agent support, per-profile `commit.gpgsign` / SSH commit signing setup, team-shareable profile templates (no keys).

## 9. Success criteria

- A new user can go from zero to a working second GitHub account in under 2 minutes.
- Running `sshp apply` on a machine with a hand-written `~/.ssh/config` never changes a line outside the managed block.
- `sshp doctor` catches the top 5 real-world misconfigurations (wrong permissions, missing key, duplicate alias, repo using plain `github.com`, agent offering too many keys).

## 10. README outline

1. One-line pitch + short terminal GIF
2. The problem (3 bullets)
3. Install (brew / scoop / go install / binaries)
4. Quick start: `sshp add work` → `sshp clone work git@github.com:org/repo.git`
5. Commands reference
6. How it works (managed block, files touched, what it never touches)
7. FAQ (passphrases, Windows, HTTPS remotes, existing config)
8. Contributing / License

---

## Appendix: prompt for an AI coding agent

Paste this into Claude Code (or similar) at the root of the empty repo:

```text
Build "sshp", a cross-platform CLI in Go that manages multiple SSH keys and
Git identities. Read SPEC.md in this repo first and follow it.

Start with milestone v0.1 only:
- Go module, cobra CLI, internal/ package layout as in SPEC.md section 7
- Profile model + TOML store at ~/.config/sshp/profiles.toml
- Commands: add, list, remove, apply, pubkey
- Render a managed block into ~/.ssh/config between the markers defined in
  SPEC.md; never modify content outside it; back up before writing; write
  atomically
- Support a --dry-run flag that prints a unified diff
- Never read or print private key contents
- Table-driven unit tests with golden files for the rendered config, and
  integration tests that use a temporary $HOME
- GitHub Actions CI (Linux, macOS, Windows), golangci-lint, MIT license,
  README following the outline in SPEC.md section 10

Work in small commits. Ask before adding any dependency not listed in SPEC.md.
```
