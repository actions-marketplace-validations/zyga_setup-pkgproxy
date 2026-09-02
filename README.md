# setup-pkgproxy

A composite GitHub Action that builds and runs
[pkgproxy](https://gitlab.com/zygoon/pkgproxy) as a caching proxy for apt,
snap store, Go module, and LXD image traffic on a GitHub Actions runner.

## What it does

1. Restores a cached `pkgproxy` binary (keyed on `version`, `go-version`, and
   runner arch), or builds it with `go install` on a cache miss.
2. Installs the binary to `/usr/local/bin/pkgproxy` (and a `pkgproxyctl`
   symlink — the binary dispatches on `argv[0]`).
3. Restores a persistent payload cache at `/var/cache/pkgproxy`.
4. Installs and starts a `pkgproxy.service` systemd unit. Its
   `RuntimeDirectory=`/`CacheDirectory=` directives create `/run/pkgproxy`
   and `/var/cache/pkgproxy` and own them appropriately.
5. Optionally points the host's `apt` and `snapd` at the proxy
   (`pkgproxyctl setup apt` / `setup snap`).

The caller is responsible for pointing any other tools at the proxy (e.g.
`pkgproxyctl setup snapcraft`, `pkgproxyctl setup lxd-remote`) and for saving
the payload cache at the end of the job.

## Usage

```yaml
- name: Set up pkgproxy caching proxy
  id: pkgproxy
  uses: zyga/setup-pkgproxy@v1
  # version: defaults to this action's pinned pkgproxy release; only pass it
  # to override.

# ... later, after the job's real work, save the payload cache only on a
# miss (GitHub caches are immutable, so re-saving an exact hit is wasted
# writes) — restore-keys already lets a partial cache warm the next run:
- if: steps.pkgproxy.outputs.package-cache-hit != 'true'
  uses: actions/cache/save@v6
  with:
    path: ${{ steps.pkgproxy.outputs.cache-dir }}
    key: ${{ steps.pkgproxy.outputs.package-cache-key-success }}
```

## Inputs

| Input               | Default            | Description                                                                 |
| -------------------- | ------------------ | ----------------------------------------------------------------------------- |
| `version`            | `v0.6.0`            | pkgproxy version to build (Go module version string, pinned not `@main`).     |
| `go-version`          | `1.27.1`            | Go toolchain used to build pkgproxy on a binary-cache miss.                   |
| `addr`                | `:3142`             | Address pkgproxy listens on.                                                  |
| `binary-cache-key`     | `pkgproxy-bin`      | `actions/cache` key prefix for the built binary.                              |
| `package-cache-key`    | `pkgproxy-cache-v2` | `actions/cache` key prefix for the payload cache.                             |
| `configure-host`       | `true`              | Whether to point the host's apt/snapd at the proxy.                           |

Bumping the pinned pkgproxy `version` is a one-file change here (this
action's `version` default) — callers that don't pass `version:` explicitly
pick it up automatically on their next `@v1` resolution.

## Outputs

| Output                       | Description                                                                 |
| ------------------------------ | ----------------------------------------------------------------------------- |
| `addr`                          | The resolved listen address (echoes the `addr` input).                       |
| `cache-dir`                      | The payload cache directory (`/var/cache/pkgproxy`) — save this at job end.  |
| `package-cache-hit`               | `'true'` when the stable success key restored exactly.                       |
| `package-cache-key-success`        | Stable per-arch key to save under on success.                                |
| `package-cache-key-partial`         | Stable per-arch key to save under when the job fails.                        |

## Versioning

Releases are tagged `vX.Y.Z`, with a floating `vX` tag moved to the latest
release in that major version — pin to `zyga/setup-pkgproxy@v1` and get
compatible updates (including pkgproxy version bumps) automatically.
