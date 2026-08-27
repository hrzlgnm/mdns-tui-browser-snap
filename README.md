# mdns-tui-browser snap

Builds [mdns-tui-browser](https://github.com/hrzlgnm/mdns-tui-browser) as a
Snap. The snap downloads the prebuilt upstream release tarball (which already
contains the binary and man page) and verifies it against the inline sha256
digest GitHub computes for the asset. This repository holds only packaging
metadata and CI.

## Install

Once published:

```console
sudo snap install mdns-tui-browser
```

Local build (requires LXD):

```console
sudo snapcraft --use-lxd
sudo snap install ./mdns-tui-browser_*.snap --dangerous
```

## Releasing

1. Tag a release on `hrzlgnm/mdns-tui-browser` (e.g. `v1.35.0`) and let its
   release workflow finish.
2. Run this repository's **Release snap** workflow with `tagName` set to that
   tag.
3. CI builds the snap in LXD, smoke tests it in an Ubuntu 24.04 container, and
   uploads it to the `stable` channel of the Snap Store.

## Store setup (one-time)

- Register the name: `snapcraft register mdns-tui-browser`
- Create exportable store credentials and add them as the
  `SNAPCRAFT_STORE_CREDENTIALS` repository secret:

  ```console
  snapcraft export-login --snaps=mdns-tui-browser --channels=stable login-file
  gh secret set SNAPCRAFT_STORE_CREDENTIALS < login-file
  rm login-file
  ```

If publish credentials are absent, CI still builds and smoke tests the snap;
the publish step is skipped.

## Notes

- Confinement is `strict`; mDNS discovery works through the `network` and
  `network-bind` interface plugs.
- This is a terminal application, so there is no desktop entry or icon.
