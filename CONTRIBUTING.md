# Contributing to darling-fedora44

Thank you for your interest in contributing! This fork exists specifically to
make [Darling](https://github.com/darlinghq/darling) build and run on
**Fedora 44 with Clang 22**. All fixes here are intended to eventually be
submitted upstream to the main Darling project.

---

## About This Fork

This is a compatibility fork — not a divergent project. The goal is to:

1. Keep Darling buildable on Fedora 44 / Clang 22
2. Track upstream Darling as closely as possible
3. Submit fixes upstream via PRs when the upstream project is receptive

All changes are documented in the [README](README.md) and the
[build guide](misc_scripts/) with rationale for each fix.

---

## How to Contribute

### Reporting Build Failures

If you find a new build failure on Fedora 44 / Clang 22 (or a newer version):

1. Open a GitHub Issue with:
   - The full error message
   - The file and line number
   - Your Fedora version (`cat /etc/fedora-release`)
   - Your Clang version (`clang --version`)

### Submitting Fixes

1. Fork this repo
2. Create a branch: `git checkout -b fix/description-of-fix`
3. Make your fix — keep it minimal and targeted
4. Update `.gitmodules` / submodule commits if the fix is in a submodule
5. Add an entry to the fix summary table in README.md
6. Open a PR against `master`

### Submodule Fixes

Most fixes live in submodules. When fixing a submodule:

```bash
# Work inside the submodule
cd src/external/<submodule-name>
# make your changes
git add -A
git commit -m "Fix Clang 22: description of fix"
cd ../../../

# Update the parent repo to point to the new submodule commit
git add src/external/<submodule-name>
git commit -m "Update <submodule-name> submodule"
```

### Fix Guidelines

All fixes should follow these principles used throughout this fork:

- **Prefer casts over changing type declarations** where the original type is
  semantically correct (e.g. `(const void **)&ptr` rather than changing the
  variable type)
- **Prefer `-Wno-incompatible-pointer-types` in CMakeLists** over individual
  file casts when a whole subproject uses the same pattern throughout
- **Never use `-w` (suppress all warnings)** — use specific suppressions only
- **Document the reason** for each fix in the commit message
- **Keep fixes minimal** — change only what is needed to fix the error

---

## Updating from Upstream Darling

To pull in upstream changes:

```bash
cd ~/darling

# Add upstream remote if not already present
git remote add upstream https://github.com/darlinghq/darling.git

# Fetch upstream
git fetch upstream

# Merge upstream master into our master
git checkout master
git merge upstream/master

# Update submodules
git submodule update --recursive

# Reapply the mach_vm.c patch if the xnu submodule was reset
python3 misc_scripts/fix_mach_vm.py

# Rebuild
cd build
cmake .. -DTARGET_i386=OFF
make -j$(nproc)
```

Note: Merging upstream may reintroduce build failures if upstream changed files
we patched. Check the build log and apply fixes as needed, following the
patterns documented in the README.

---

## Submitting Fixes Upstream

If you want to submit a fix to the main Darling project:

1. Identify which upstream submodule repo owns the file
   (e.g. `src/external/foundation` → `https://github.com/darlinghq/darling-foundation`)
2. Fork that upstream repo
3. Submit a PR there first
4. Once merged upstream, the main Darling repo can update its submodule pointer
5. Then this fork can pull from upstream

Because Darling development has been slow, opening a GitHub Issue on the
[main Darling repo](https://github.com/darlinghq/darling) with a link to this
fork and a description of the fixes may also be effective.

---

## Code of Conduct

Be respectful. This is a small community project. Constructive feedback and
collaboration are welcome; hostility is not.

---

## License

This fork is licensed under the **GNU General Public License v3.0**, the same
as the upstream Darling project. See [LICENSE](LICENSE) for the full text.

By contributing, you agree that your contributions will be licensed under GPLv3.
