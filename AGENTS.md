# phi-packages — packaging

PKGBUILDs and build scripts for every in-house package (`phi`, `phi-shell`,
and later `phi-notes`, `phi-music`, `phi-media`). **This repo produces
packages; it never installs them, and it never touches a real machine.**

The workspace `AGENTS.md` one level up carries the rules for every repository
and is not repeated here.

## The release boundary

**Only the user builds signed packages and publishes them.** An agent's entire
role in a release is to create and push the git tag on the *source*
repository:

```sh
cd ../phi && git tag v0.21.0 && git push origin v0.21.0
```

After that, an agent may bump `pkgver` here to match the new tag and adjust
the PKGBUILD. It must **not** run `scripts/build`, `scripts/publish`,
`makepkg`, `makechrootpkg`, `repo-add`, `pacman`, or anything else that
produces or moves an artifact. The user does that on `zotac`.

- The **signing key never appears in this repository**, in any form,
  encrypted or not. It was generated once on `zotac` and the private half
  never leaves it.
- The repository database address is never written here. Moving a built repo
  to the server is a manual step the user performs, needing an address no
  repository may hold.

## How this repo is built

- `<component>/PKGBUILD` — one directory per package, named like the package:
  `phi` for the CLI, `phi-<component>` for everything else, matching the
  source repository.
- `pkgver` is hand-bumped and always names a real git tag (`v<pkgver>`) in the
  source repo: `source=(git+https://…#tag=v${pkgver})`, never `latest`, never
  a `-git` snapshot. `pkgrel` increments for a packaging-only rebuild with no
  source change.
- `build()` for `phi` is
  `go build -trimpath -ldflags "-s -w -X phi/internal/build.Version=${pkgver}"`.
  `makedepends` includes `go` and `git`, since a clean chroot has neither.
- `package()` regenerates the zsh completion and the man page by running the
  freshly built binary — nothing generated is checked in.
- `scripts/build <component>` — `mkarchroot` / `makechrootpkg -c` against a
  clean chroot at `~/.cache/phi-packages/chroot`. **User-run, on `zotac`
  only.**
- `scripts/publish <repo-dir> <pkg>…` — detached-sign plus `repo-add -s`,
  requiring `GPGKEY` explicitly. **User-run.**

## Constraints that stay

- Every dependency in every PKGBUILD must be an official Arch package
  (`core` / `extra` / `multilib`). No AUR `depends` or `makedepends`.
- Keep the chroot infrastructure general enough that it stays configuration,
  not a rewrite, if the package set grows.
