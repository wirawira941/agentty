# agentty on Termux (this fork)

Native Termux packaging of [agentty](https://github.com/agentty-org/agentty)
for Android aarch64: a dynamically-linked Bionic PIE, built from source
against Termux's own clang and libraries.

## Install from the release (no building)

Prebuilt packages are published on this fork's releases, e.g.
[v0.9.31-termux.1](https://github.com/wirawira941/agentty/releases/tag/v0.9.31-termux.1):

```sh
pkg install -y libc++ libnghttp2 openssl simdjson
curl -LO https://github.com/wirawira941/agentty/releases/download/v0.9.31-termux.1/agentty_0.9.31_aarch64.deb
dpkg -i agentty_0.9.31_aarch64.deb
agentty --version
```

(`simdjson` is listed explicitly because the 0.9.31 binary links
`libsimdjson.so` at runtime; the recipe has since been fixed so future
`.deb`s declare it in `Depends`.)

The release also carries the bare binary (`agentty_termux`) if you prefer
to drop it into `$PREFIX/bin/agentty` yourself — the runtime packages
above are still required.

Run it against any OpenAI-compatible endpoint, e.g. a local 9Router:

```sh
agentty --provider 127.0.0.1:20128 -k <API_KEY> -m big-pickle
```

If shell commands inside agentty fail to run, start it with
`--sandbox off` — the claybin sandbox needs kernel features
(`SECCOMP_RET_USER_NOTIF`, landlock) that are not available everywhere.

## Why a source build (not the official release binary)

Upstream's `agentty-linux-aarch64` release binary is fully static, but its
ELF type is `ET_EXEC` — Android's linker rejects non-PIE executables
(`error: "..." has unexpected e_type: 2`). So the Termux package is
compiled from source with `AGENTTY_STANDALONE=OFF`, producing an ordinary
dynamic PIE: `ET_DYN`, interpreter `/system/bin/linker64`,
`RUNPATH=$PREFIX/lib`, linked against Termux's `libc++`, OpenSSL 3,
nghttp2 and simdjson.

## How the package is built

`packaging/termux/build.sh` (branch `build-termux`) is a
[termux-packages](https://github.com/termux/termux-packages) recipe:

- **Source:** the upstream *release tarball*
  (`agentty-<version>.tar.gz`), not a git clone and not a GitHub source
  archive. agentty vendors `maya`, `acp-cpp`, `mcp-cpp` and `rag-cpp` as
  git submodules; the release tarball ships them populated under
  `third_party/`, so the build needs no network after the source step.
- **Flags that matter:**
  - `AGENTTY_STANDALONE=OFF` — dynamic Bionic PIE (see above).
  - `AGENTTY_USE_MIMALLOC=OFF` — mimalloc has no Termux package and no
    `find_package` fallback; Bionic's allocator is fine.
  - `MAYA_NATIVE_TUNING=OFF` — never `-march=native` when cross-compiling.
  - `AGENTTY_AUTO_PULL_SUBMODULES=OFF`, `AGENTTY_BUILD_TESTS=OFF`,
    `AGENTTY_COMPILER_CACHE=OFF`.
- **Runtime deps:** `libc++`, `libnghttp2`, `openssl`, `simdjson`.
- **Build deps:** `nlohmann-json`, `simdjson` (a system copy is used via
  `find_package` when present).

## CI and cutting a new release

`.github/workflows/termux-build.yml` (branch `build-termux`) builds the
package in the official termux-packages Docker environment and uploads
the `.deb` + extracted binary as an artifact. To publish a release:

1. Bump `TERMUX_PKG_VERSION` in `packaging/termux/build.sh` and set
   `TERMUX_PKG_SHA256` to the SHA256 of the new upstream release tarball
   (from that release's `SHA256SUMS`).
2. Dispatch the workflow manually (Actions → *Build agentty_termux* →
   Run workflow, branch `build-termux`) with the `release_tag` input set,
   e.g. `v0.9.31-termux.1`. Leave it empty for an artifact-only build.

Notes for maintainers: the workflow file also exists on `master`
(the default branch) because GitHub only shows the *Run workflow* button
for `workflow_dispatch` when the file is present there — its push trigger
only listens to `build-termux`, so the master copy is inert. The publish
step passes `--repo` explicitly to `gh release create` because the
checkout lives in the `agentty-src/` subdirectory (no `.git` at the
workspace root).

## Upstreaming

To make plain `pkg install agentty` work, this recipe would need to be
merged into `termux/termux-packages` as `packages/agentty/build.sh` —
the CI above already builds it exactly that way.
