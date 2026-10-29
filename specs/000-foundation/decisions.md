# Foundation decisions

Status: **accepted**. These close the gaps listed in PLAN.md section 2. Where they extend SPEC.md, fold them into SPEC.md in a follow-up commit.

| # | Decision | Rationale |
|---|---|---|
| D1 | Name stays `sshp`. Verify GitHub / pkg.go.dev / Homebrew availability before fixing the Go module path. | Short to type; matches SPEC. |
| D2 | MIT license. Verify the existing `LICENSE` is MIT. | Simple, permissive. |
| D3 | Prompts: `charmbracelet/huh`. | Maintained; good Windows support. |
| D4 | Tables: stdlib `text/tabwriter`. No table dependency. | Zero extra deps. |
| D5 | `--dry-run` unified diff is implemented in-house. No diff dependency. Any dependency beyond SPEC section 7 still needs approval. | Files are tiny; avoids a dependency. |
| D6 | No SSH config parser in v0.1. Marker-based text splicing only. Parser arrives in v0.3 (`import`, `doctor`). | Keeps MVP small. |
| D7 | Profile `name` matches `[a-z0-9][a-z0-9_-]*`. `alias` is unique across profiles. Paths are stored in `~` form and expanded at use. | Safe in filenames, Host aliases, CLI args; avoids case collisions on Windows/macOS. |
| D8 | Missing or duplicated managed-block marker: refuse, exit code 3, explain. Never guess. No `--force` override. | Protects the user's config. |
| D9 | Backup retention (last 5) is per file. | Each config keeps its own history. |
| D10 | `--config` points to `profiles.toml` only. Tests isolate via a temp `HOME` / `USERPROFILE` / `APPDATA`. | No extra flag surface. |
| D11 | Preserve the file's dominant EOL (LF or CRLF). Write the managed block using it. Bytes outside the block are untouched. | Meets "never change a line outside the block". |
| D12 | `remove --delete-key` refuses (with a warning) if another profile references the same key file. | Prevents deleting a shared key. |

## Follow-ups
- Check name availability (D1) and confirm LICENSE text (D2).
- Fold D7, D8, D11, D12 into SPEC.md section 6 as numbered behavior rules.
