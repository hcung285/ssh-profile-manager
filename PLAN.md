# sshp: Implementation Plan (Spec-Driven Development)

`SPEC.md` is the source of truth. This plan never changes behavior; it only sequences the work. If implementation reveals a spec gap, **fix SPEC.md first (own commit), then the code.**

## 1. SDD workflow (per feature)

```
SPEC.md  →  specs/NNN-feature/{spec,plan,tasks}.md  →  failing tests  →  code  →  verify vs. spec
```

1. **Specify**: a short feature spec in `specs/NNN-<feature>/spec.md`: user stories, behavior rules (cite SPEC.md sections), edge cases, and acceptance criteria as Given/When/Then.
2. **Plan**: `plan.md`: packages touched, public function signatures, data shapes, risks.
3. **Tasks**: `tasks.md`: ordered, small (<1 commit) checklist. Each task names the test that proves it.
4. **Test first**: acceptance criteria become table-driven tests / golden files / integration tests *before* implementation.
5. **Implement** the minimum to pass; one commit per task.
6. **Verify**: tick every acceptance criterion; update the traceability matrix (section 6).

Definition of done for any feature: all acceptance criteria have a passing test, `golangci-lint` clean, CI green on Linux/macOS/Windows, README/docs updated.

## 2. Spec readiness: decisions to close before coding

SPEC.md has gaps that would otherwise be decided ad hoc. Resolve each in SPEC.md (or `specs/000-foundation/decisions.md`):

| # | Gap | Proposed decision |
|---|---|---|
| D1 | Final name (SPEC says rename freely) | Check GitHub/pkg.go.dev/Homebrew now; module path depends on it |
| D2 | License: "MIT or Apache-2.0" | MIT (LICENSE already exists in repo; verify which) |
| D3 | Prompt library: `huh` vs `survey` | `charmbracelet/huh` (maintained); needs approval, see D5 |
| D4 | Table lib: `lipgloss` vs `tablewriter` | `tablewriter` (smaller); or plain `text/tabwriter` to avoid a dep |
| D5 | Dependencies not all listed for v0.1 (diff library, test helpers) | Unified diff: small in-house impl or `pmezard/go-difflib`; golden helper: in-house. **Ask before adding anything beyond SPEC section 7** |
| D6 | `ssh_config` parser needed at v0.1? | No. v0.1 only does marker-based text splicing; parser deferred to v0.3 |
| D7 | Profile validation rules (name charset, alias uniqueness, path expansion of `~`) | `name`: `[a-z0-9][a-z0-9_-]*`; alias unique across profiles; store `~` form, expand at use |
| D8 | Behavior when managed block markers are malformed (one marker missing, duplicated) | Refuse with exit 3 and explain; never guess |
| D9 | Backup retention scope ("keep last 5") per file or global | Per file |
| D10 | `--config` meaning: profiles.toml path only, or also ssh config path? | profiles.toml only; add `--ssh-config` (hidden/testing) for integration tests, or rely on `$HOME` override |
| D11 | Windows: `~` expansion, `%APPDATA%`, CRLF in ssh config | Preserve existing line endings; write new block with the file's dominant EOL |
| D12 | `remove` when profile's key is referenced by another profile | Warn; `--delete-key` refuses unless sole reference |

## 3. Architecture (from SPEC section 7)

```
cmd/sshp/main.go            # thin: build root cmd, map errors → exit codes
internal/profile/           # Profile, validation, defaults-by-provider, TOML Store
internal/sshconfig/         # Render(profiles) → block; Splice(file, block) with markers; (parser in v0.3)
internal/keys/              # Generate (ssh-keygen; x/crypto fallback), perms check/repair, fingerprint (pub only)
internal/gitcfg/            # (v0.2+) shell out to git; remote rewrite
internal/doctor/            # (v0.3)
internal/ui/                # prompts, tables, color, diff printing
internal/fsx/               # atomic write, timestamped backup + prune, EOL detection  (cross-cutting; not in SPEC layout, justified by section 6.1-6.2)
```

Design rules that fall out of the spec:
- **Pure core, impure edges**: `Render` and `Splice` are pure string functions (golden-testable). Filesystem, `exec`, and prompts sit behind small interfaces so tests inject fakes.
- **Single `$HOME` resolver** (`paths` helper) so integration tests set a temp home and every path derives from it.
- **Typed errors** carry an exit code (1/2/3), mapped once in `main`.
- **Idempotency by construction**: `apply` = render → splice → compare → write only if changed.
- Private keys are only `os.Stat`ed; no code path opens them. Add a lint/test guard (grep test) that `keys` never calls `os.ReadFile` on a non-`.pub` path.

## 4. Milestones → work packages

Each package below becomes a `specs/NNN-*` folder with its own spec/plan/tasks.

### Phase 0: Foundation (before v0.1)
| WP | Deliverable | Acceptance |
|---|---|---|
| 000 | Close decisions D1–D12; Go module; `.gitignore`; `.golangci.yml`; CI skeleton | `go build ./...` and lint pass on 3 OSes |
| 001 | `fsx`: atomic write, backup + prune-to-5, EOL detection | Crash mid-write leaves original intact; 6th backup deletes oldest; CRLF file stays CRLF |
| 002 | `paths`/env: HOME, `~` expansion, `%APPDATA%` vs XDG | Table tests per OS (build-tagged or injected GOOS) |

### v0.1 MVP (SPEC section 8)
| WP | Spec refs | Deliverable | Key acceptance criteria |
|---|---|---|---|
| 010 | §4, D7 | `profile`: model, provider defaults (github/gitlab/bitbucket/custom), validation | Alias defaults to `<provider>-<name>`; duplicate name/alias rejected; invalid name rejected |
| 011 | §4 | TOML `Store` (load/save/add/remove), atomic, stable ordering | Round-trip equals input; missing file = empty store; unknown fields preserved or rejected (decide in spec) |
| 012 | §4, §6.6 | `sshconfig.Render`: deterministic block with `IdentitiesOnly yes`, optional `Port` | Golden files: 0, 1, many profiles; custom port; stable order |
| 013 | §4, §6.1, 6.2, 6.8, 6.9, D8 | `sshconfig.Splice`: replace/insert/remove managed block | Bytes outside markers identical (property/fuzz test); no block → append; run twice = no diff; foreign `Host` alias collision → refuse unless `--force`; malformed markers → exit 3 |
| 014 | §6.3–6.5 | `keys`: generate ed25519 via `ssh-keygen`, fallback to `x/crypto/ssh`; perms 700/600 (icacls on Windows); fingerprint from `.pub` | Existing key never overwritten without `--yes`; perms repaired; private key content never read/logged |
| 015 | §5 `apply`, §6.1 | `sshp apply` + `--dry-run` unified diff | Dry-run writes nothing; real run creates backup; unchanged → "already up to date", exit 0 |
| 016 | §5 `add` | Wizard (provider, new/existing key, git identity); prints pubkey + provider "add SSH key" URL; non-interactive flags for scripting/tests | Works fully via flags with `--yes`; ends by running apply |
| 017 | §5 `list`, `pubkey` | Table incl. key-exists; agent-loaded column best-effort (`ssh-add -l`, absent agent ≠ error); `pubkey --copy` | Missing key shown, not fatal; clipboard failure degrades to print |
| 018 | §5 `remove` | Remove profile, re-render block; `--delete-key` with confirmation | Block updated; key files only deleted with confirm/`--yes` |
| 019 | §5 global flags, §6.10 | Global flags, exit-code mapping, `--no-color` | Usage error → 2; missing `ssh`/bad perms → 3 |
| 020 | §7, §10 | Integration tests (temp `$HOME`, real binary), CI matrix, README per §10, CONTRIBUTING, CODE_OF_CONDUCT, SECURITY | Quick-start scenario script passes on all OSes |

**v0.1 exit gate** (SPEC §9 subset): hand-written `~/.ssh/config` fixture → `apply` → diff proves zero changes outside the block; second account set up end-to-end by flags in a test.

### v0.2
| WP | Deliverable | Notes |
|---|---|---|
| 030 | `gitcfg` + URL rewrite (§6.7) | Table tests: `git@host:o/r.git`, `https://host/o/r(.git)`, already-aliased, non-matching host, ports (`ssh://`) |
| 031 | `use`, `clone` | Integration with temp git repos; local `user.name`/`email` only |
| 032 | `whoami`, `test` | Parse `ssh -T` output per provider; stub `ssh` in tests via PATH shim |

### v0.3
| WP | Deliverable | Notes |
|---|---|---|
| 040 | SSH config parser (`kevinburke/ssh_config` vs custom; **dependency approval**) + `import` | Non-destructive; import never modifies source file |
| 041 | `doctor` checks | One check per SPEC §9 top-5 (perms, missing key, dup alias, repo on plain host, agent offering too many keys) plus `Host *` override; each check has a fixture and a fix hint |
| 042 | `link` via `includeIf "gitdir:"` | Writes to `~/.gitconfig` + included file; backup rules apply; trailing-slash/Windows path normalization |

### v0.4
Completions (cobra built-in), GoReleaser, Homebrew tap, Scoop bucket, `.deb`/`.rpm`, provider presets (GitLab/Bitbucket self-hosted hostnames).

### v1.0
Freeze `profiles.toml` schema (add `schema_version`, migration test), docs site, coverage gate on `sshconfig`/`fsx` (target ≥90%), fuzz tests on Splice.

## 5. Test strategy (maps to SPEC §7)

| Layer | Technique |
|---|---|
| Render | Table-driven + golden files (`testdata/golden/*.conf`, `-update` flag) |
| Splice | Table-driven + fuzz/property: `outside(before) == outside(after)`; idempotent |
| Store | Round-trip, corrupt-file handling |
| fsx | Temp dirs; simulated rename failure |
| keys | Real `ssh-keygen` in CI; fallback path tested by hiding PATH |
| CLI | Integration: build binary, set `HOME`/`USERPROFILE`/`APPDATA`, run scenarios, assert files + exit codes |
| Windows | ACL tests gated by build tag; CRLF fixtures |

## 6. Traceability matrix (maintain in `specs/TRACEABILITY.md`)

One row per SPEC requirement → WP → test name → status. Seed rows:

| SPEC | Requirement | WP | Test |
|---|---|---|---|
| §6.1 | Backup before write, keep last 5 | 001, 015 | `TestBackupPrune`, `TestApplyCreatesBackup` |
| §6.2 | Atomic write | 001 | `TestAtomicWriteCrashSafe` |
| §6.5 | Never read private key | 014 | `TestKeysNeverOpenPrivate` |
| §6.6 | `IdentitiesOnly yes` always | 012 | golden files |
| §6.8 | Idempotency | 013, 015 | `TestApplyTwiceNoDiff` |
| §6.9 | Alias conflict refusal | 013 | `TestSpliceForeignHostConflict` |
| §9 | Zero change outside block | 013, 020 | `TestOutsideBlockUntouched`, fuzz |

## 7. Risks

| Risk | Mitigation |
|---|---|
| Corrupting a user's real `~/.ssh/config` | Backup + atomic write + splice fuzzing; all integration tests run in temp `$HOME` only |
| Windows ACL / path / CRLF quirks | Windows in CI from day one (WP 000), not at the end |
| Dependency creep | Ask-before-adding rule (SPEC appendix); D5 decided up front |
| Provider `ssh -T` output drift | Isolate parsers per provider, fixtures from real output |
| Scope creep from "Ideas for later" | Out of scope until v1.0 specs are signed off |

## 8. Immediate next steps

1. Review and answer D1–D12 (especially D1 name, D3–D5 dependencies).
2. Create `specs/000-foundation/` and commit the decisions.
3. Start WP 000, then 001, 002, 010 with spec → tests → code.
