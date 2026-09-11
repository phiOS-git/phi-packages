# phi-packages — packaging

PKGBUILDs and build scripts for every in-house package (`phi`, `phi-shell`,
and later `phi-notes` / `phi-music` / `phi-media`). **This repo produces
packages; it never installs them, and it never touches a real machine.**

Part of the phiOS workspace. The workspace `AGENTS.md` (one level up, or in
`docs/archive/` of a standalone clone) carries the rules for every
repository — **branch locally, only `main`/`dev` on the remote; only
official Arch packages; no secrets in a public repo.** Not repeated here.

When your work matches an entry in the workspace's `docs/TODO.md`, claim it
with `[taken]` and report the result in `docs/VERIFICATION.md` — see *The
TODO / VERIFICATION loop* in the workspace `AGENTS.md`.

## The release boundary — read this

**Only the user builds signed packages and publishes them.** An agent's
entire role in a release is to create and push the git tag on the *source*
repository:

```
cd ../phi && git tag v0.17.0 && git push origin v0.17.0
```

Then, in this repo, an agent may bump `pkgver` to match the new tag and
adjust the PKGBUILD. It must **not** run `scripts/build`, `scripts/publish`,
`makepkg`, `makechrootpkg`, `repo-add`, `pacman`, or anything that produces
or moves an artifact. The user does that on `zotac`.

- The **signing key never appears in this repository**, in any form,
  encrypted or not. It was generated once on `zotac`; the private half
  never leaves it.
- The repository database address (`mini:/srv/pkg/phi`, served over the
  overlay network) is never written here — moving a built repo there is a
  manual step the user does, needing an address no repository may hold.

## How this repo is built

- `<component>/PKGBUILD` — one directory per package, named like the
  package. `phi` for the CLI, `phi-<component>` for everything else,
  matching the source repository.
- `pkgver` is hand-bumped and always names a real git tag (`v<pkgver>`) in
  the source repo. `source=(git+https://…#tag=v${pkgver})` — never
  `latest`, never a `-git` snapshot. `pkgrel` increments for a
  packaging-only rebuild (changed `build()`, new dependency) with no source
  change.
- `build()` for `phi`:
  `go build -trimpath -ldflags "-s -w -X phi/internal/build.Version=${pkgver}"`.
  `makedepends` includes `go` and `git` (a clean chroot has neither by
  default).
- `package()` regenerates the zsh completion and man page by running the
  freshly built binary — nothing generated is checked in.
- `scripts/build <component>` — `mkarchroot` / `makechrootpkg -c`, clean
  chroot at `~/.cache/phi-packages/chroot`. **User-run, on `zotac` only.**
- `scripts/publish <repo-dir> <pkg>…` — detached-sign + `repo-add -s`,
  requires `GPGKEY` explicitly. **User-run.**

## Constraints that stay

- Every dependency in every PKGBUILD must be an official Arch package
  (`core` / `extra` / `multilib`). No AUR `depends` or `makedepends`.
- Keep the chroot infrastructure general enough that it stays
  configuration, not a rewrite, if the package set grows.
