# AGENTS.md — mdns-tui-browser-snap

Repackages the prebuilt upstream `mdns-tui-browser` Linux build (a versioned
tarball asset) as a strictly-confined snap. The defining feature of this repo
is that the snap **build verifies the upstream tarball's SLSA build
provenance** before packaging it, so a compromised upstream publish step
cannot silently swap the binary.

## Provenance verification (the important mechanism)

We verify with `gh attestation verify` using the **offline bundle** flow:

- `gh attestation verify <artifact> --repo hrzlgnm/mdns-tui-browser --bundle
  <bundle.jsonl> --custom-trusted-root trusted_root.jsonl --cert-identity
  "<pin>"`
- Auth is **only** needed to *fetch* the attestation from the GitHub API
  (`gh attestation download` / `gh attestation trusted-root`). The actual
  verify with `--bundle --custom-trusted-root` is pure crypto and runs with
  **no auth and no network** (confirmed locally with an empty gh config).
- Therefore the build container never needs a token. Do **not** try to pass
  `GH_TOKEN` into the `--use-lxd` container.

### Why no token in the container (snapcraft --use-lxd gotchas)

- `snapcraft --use-lxd` builds inside an LXD container and does **not** forward
  the host process environment. `sudo env GH_TOKEN=… snapcraft pack` does
  nothing inside the container.
- `build-environment` values are written **verbatim** into the container's
  scriptlet env. A `GH_TOKEN: ${GH_TOKEN}` placeholder is NOT expanded from
  the host, so at build time it becomes `GH_TOKEN="${GH_TOKEN}"` and fails
  under `set -u` with `GH_TOKEN: unbound variable`. This is why the offline
  bundle approach exists instead.

### Versioned asset (unlike the sibling snaps)

The upstream tarball asset name embeds the tag:
`mdns-tui-browser-${tag}-Linux-x86_64.tar.gz`. This is why this repo's pin
action emits both `tag` and `asset` outputs — the asset cannot be derived from
a static name, and the fetch step needs both:

```yaml
- name: Pin snap version to latest upstream release
  id: pin
  uses: ./.github/actions/pin-release-version

- name: Fetch attestation bundle
  uses: ./.github/actions/fetch-attestation
  with:
    repo: hrzlgnm/mdns-tui-browser
    tag: ${{ steps.pin.outputs.tag }}
    asset: ${{ steps.pin.outputs.asset }}
```

(`release.yml` passes `tag: ${{ inputs.tagName }}` into `pin-release-version`
as well.)

### Host fetch step (local composite action, used in BOTH workflows)

Before `snapcraft pack`, **`.github/actions/fetch-attestation`** downloads the
versioned asset, runs `gh attestation download` + `gh attestation
trusted-root`, and writes the bundle (`sha256*.jsonl`) and `trusted_root.jsonl`
into the project dir. It takes `repo`, `asset`, and `tag`. The **project
directory is mounted into the LXD container** (`$SNAPCRAFT_PROJECT_DIR`), so
files we generate on the host runner are visible to the build without any env
plumbing.

> This is a **per-repo local copy** (duplicated in each snap repo that uses
> the offline bundle flow). It is intentionally **not** shared via
> `hrzlgnm/actions` — each repo keeps its own copy.

### Inside `snap/snapcraft.yaml` (override-build)

The verify runs over `release.tar.gz` (the downloaded versioned tarball):

```bash
bundle="$(ls "$SNAPCRAFT_PROJECT_DIR"/sha256*.jsonl | head -1)"
"$gh_bin" attestation verify release.tar.gz \
  --repo hrzlgnm/mdns-tui-browser \
  --bundle "$bundle" \
  --custom-trusted-root "$SNAPCRAFT_PROJECT_DIR/trusted_root.jsonl" \
  --cert-identity "https://github.com/hrzlgnm/mdns-tui-browser/.github/workflows/build-reusable.yml@refs/tags/${tag}"
```

- `core24`'s apt `gh` (2.45) predates the `attestation` subcommand, so the
  build bootstraps a static CLI: `gh_ver=2.98.0`, tarball
  `gh_2.98.0_linux_amd64.tar.gz`, pinned SHA256
  `3b8ac6b30336802fc1a858d7c084e11cdf24ac1a761ca90b68022d7d729208de`.
- This is a **binary** attestation (over the tarball). Do **not** pass
  `--source-digest` here.
- `cert-identity` pin:
  `https://github.com/hrzlgnm/mdns-tui-browser/.github/workflows/build-reusable.yml@refs/tags/${tag}`

## CI structure

- **`ci.yml`** — runs on push to `main` and on PRs. Builds the snap with
  `sudo snapcraft pack --use-lxd` and runs the provenance verify. This is the
  normal way changes are validated. **A push to `main` is enough to exercise
  the verify** — nothing else needs to be triggered.
- **`release.yml`** — `on: workflow_dispatch` only (input `tagName`). Builds,
  smoke-tests in an LXD container, then **publishes to the Snap Store stable
  channel**. The `publish` job is entirely gated behind the
  `SNAPCRAFT_STORE_CREDENTIALS` secret (`if: env.SNAPCRAFT_STORE_CREDENTIALS
  != ''`); with no secret set, publish auto-skips and nothing is released.

## Working conventions (do not repeat past mistakes)

- **Never run `gh workflow run` for the Release snap workflow unless explicitly
  asked.** Pushing to `main` already triggers `ci.yml`, which builds and runs
  the verify. Only trigger `release.yml` when actually publishing.
- **Do not push or commit without authorization.**
- **Verify assumptions before asserting them.** E.g. it was wrongly assumed
  that `gh` needs auth inside the container; testing `gh attestation verify
  --bundle` with an empty gh config proved otherwise. Prefer a quick local
  test over a guess.
- Keep the host fetch step in sync between `ci.yml` and `release.yml`.

## Sibling repos (same pattern)

The same offline-bundle verification is applied across the snap repos that
repackage prebuilt upstream binaries. Each now has its own `AGENTS.md`:

- `mdns-browser-snap` — binary attestation over `mdns-browser_linux_x64`,
  cert pinned to `desktop-reusable.yml`, no `--source-digest`. Uses
  `pin-upstream-tag`.
- `zux-snap` — binary attestation over `zux_linux_x64`, cert pinned to
  `release.yml`, no `--source-digest`. Uses `pin-upstream-tag`.

## Shared action

The actual `snapcraft pack` invocation lives in the shared
`hrzlgnm/actions/.github/actions/build-snap` (referenced as `@v2.7.0` from
`release.yml`). `v2.7.0` runs `sudo env GH_TOKEN="$GH_TOKEN" snapcraft pack
--use-lxd`; it does **not** inject a token (not needed with the offline
bundle approach). `ci.yml` calls `snapcraft pack` directly instead.
