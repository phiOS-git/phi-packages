# phi-packages

PKGBUILDs and build scripts for every own package (master plan §3.1). This
repository produces packages; it never installs them, and it never touches a
real machine — `scripts/build` and `scripts/publish` are run by the user, on
`zotac`.

## Layout

```
<component>/PKGBUILD    one directory per package, named like the package
scripts/build            drives a clean-chroot build for one package
scripts/publish           signs a build and adds it to a local repo database
```

## Naming

`phi` for the CLI, `phi-<component>` for everything else (`phi-shell`,
`phi-notes`, ...), matching the source repository each package builds from.

## Versioning

`pkgver` is bumped by hand and always names a real git tag (`v<pkgver>`) in
the package's source repository — never `latest`, never a `-git`
VCS-snapshot package. `source=()` pins the build to that tag
(`git+https://.../phi.git#tag=v${pkgver}`), so a clean-chroot build is
reproducible: the version that ends up installed always corresponds to an
exact, inspectable point in the source repository's history. The tag itself
is not cryptographically verified — the security boundary is the package and
database signing in `scripts/publish`, below, not the source tag.

`pkgrel` increments for a rebuild of the same `pkgver` (a packaging-only
fix — a changed `build()`, a new dependency — with no change to the tagged
source). Any change to the tagged source itself gets a new `pkgver` and a new
tag, `pkgrel` reset to 1.

## Building

```
scripts/build <component>
```

One-time: creates a chroot at `~/.cache/phi-packages/chroot` (override with
`PHI_PACKAGES_CHROOT`) via `mkarchroot`. Every build then runs
`makechrootpkg -c`, which resets the chroot to its clean snapshot first — the
same starting point every time, regardless of what a previous build left
behind. Needs `base-devel`, `devtools`, `go`, `pacman-contrib` (master plan
§15.8), installed by hand on `zotac` — the only build machine, chosen for its
32 GB of RAM; `mini` never compiles anything.

## Publishing

```
scripts/publish <repo-dir> <package-file>...
```

Detached-signs each package with the dedicated phi packaging key (generated
once on `zotac`, exported public-only to every machine's pacman keyring — the
private key never leaves `zotac` and is never written to any repository, in
any form), then `repo-add -s` to fold it into `<repo-dir>/phi.db.tar.zst`,
signing the database too. `SigLevel = Required` on every consuming machine
(below) then verifies both.

`<repo-dir>` is a plain local directory — this script never touches a remote
host. Moving its contents to `mini:/srv/pkg/phi`, where `darkhttpd` serves it
over the overlay network, is a separate manual step: it needs `mini`'s
overlay address, which by `CLAUDE.md` rule 5 may never be written into any
repository, this one included.

## Registering a host

Every machine (master plan §3.1: "registrato in `/etc/pacman.conf` di ogni
macchina") needs two things, both declared by `phios-dotfiles` and applied by
hand — never by `phios-install`, which only ever shows the difference:

- `profiles/base/system/etc/pacman.d/phi-mirrorlist` — a real, diffable file
  at its real absolute path, holding the `Server =` line. It ships with
  `<mini-overlay-address>` and `<port>` as literal placeholders; nothing here
  or anywhere else in either repository ever fills them in. Don't try to
  `sed` both a placeholder and its replacement value in one command — it's
  easy to end up substituting a string for itself and silently changing
  nothing (this happened on the real `zotac` run). Just overwrite the whole
  line: `echo "Server = http://<addr>:<port>/" | sudo tee
  /etc/pacman.d/phi-mirrorlist`.
- `profiles/base/manual.txt` — the two-line `[phi]` section that
  `Include=`s the file above into `/etc/pacman.conf` (which itself has no
  drop-in mechanism for a repository section — the same limitation
  `profiles/gaming/manual.txt` already documents for `multilib`), plus the
  `pacman-key --add`/`--lsign-key` step for the packaging public key.

The exported public key file (`phi-packaging-key.asc`) only exists on
`zotac`, where it was generated — `pacman-key --add` needs it present
locally on every host, so `scp` it to `razer` and `mini` before running that
step there. Nothing here does that automatically; it is not a file either
repository tracks (the key is `[USER]` material, never committed).
