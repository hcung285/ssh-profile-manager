# sshp

Manage multiple SSH keys and Git identities (personal, work, clients) from one CLI. Generates `~/.ssh/config`, rewrites remotes, and sets the right `user.email` per repo.

> **Status:** planning. No code yet. See [SPEC.md](SPEC.md) and [PLAN.md](PLAN.md).

## The problem

- Hand-editing `~/.ssh/config` with one `Host` alias per account is error-prone.
- Cloning with the wrong URL silently uses the wrong key, and commits get the wrong `user.email`.
- `ssh-agent` offers every key to every server, causing "too many authentication failures".

## Planned usage

```sh
sshp add work                                      # generate a key, write SSH config, print the public key
sshp clone work git@github.com:org/repo.git        # clone using the profile's alias and identity
sshp use work                                      # apply a profile to the current repo
sshp doctor                                        # diagnose common misconfigurations
```

`sshp` only edits a clearly marked managed block in `~/.ssh/config` and never reads or prints private keys.

## Documentation

- [SPEC.md](SPEC.md): product specification
- [PLAN.md](PLAN.md): spec-driven implementation plan
- [specs/](specs/): per-feature specs and decisions

## License

[MIT](LICENSE)
