# phi-packages

PKGBUILDs and build scripts for every own package. This repo produces packages; it does not install them.

- Builds happen in a **clean chroot** on `zotac`, driven by the user. You write the scripts; you never run a build against a real system.
- The signing key is **never** in this repository, in any form, encrypted or not.
- Package naming: `phi` for the CLI, `phi-<component>` for everything else.
- The repository database lives on the server at `/srv/pkg/phi` and is served over the overlay network. Never write that address or any hostname into a file here — the `pacman.conf` fragment lives in `phios-dotfiles/profiles/*/system/` and the user fills in the machine-specific part.
- The same clean-chroot infrastructure would close `Q-01` in its "local signed repository" variant. Keep it general enough that adding an AUR-sourced PKGBUILD later is configuration, not a rewrite.
