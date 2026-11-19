# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

`tang` — the server half of Network Bound Disk Encryption (NBDE): an HTTP
server that advertises public keys and performs key recovery exchanges with
Clevis clients. C99, meson build. Upstream is `latchset/tang`; this fork
(`sarroutbi/tang`) carries the hybrid post-quantum (ML-KEM) work on feature
branches such as `tang-kem-pub`, rebased onto `upstream/master`.

## Build and test

```sh
meson setup build          # once
ninja -C build             # build (must be warning-free; werror is on)
meson test -C build        # full suite (unit + end-to-end)
meson test -C build test-pqc rec-standalone   # subset
```

Requires `jose >= 8` (with jansson), llhttp or http-parser headers; `a2x`
(optional, man pages), `socat` (optional, some e2e tests), `curl`.

- `has_pqc` is a configure-time probe: if jose supports the ML-KEM KEM API,
  the build uses `src/pqc.c` (full hybrid protocol, links libcrypto);
  otherwise `src/pqc_noop.c` (KEM endpoints return 404). The probe is
  *executed* when possible (`compiler.run()`), not just linked: jose
  exports the KEM API unconditionally and may register no ML-KEM hook
  (e.g. built with `-Dmlkem=disabled`); `compiler.links()` is only a
  cross-build fallback. `pqc_is_available()` re-verifies at runtime.
  `libcrypto` is a dependency **only** when `has_pqc` — do not make it
  unconditional.
- The probe result is cached. After swapping jose builds on the machine,
  run `meson setup --reconfigure <builddir>` or use a fresh build dir.
- Local build dirs (`build*`, `builddir_test`) are untracked; never commit
  them.

## Layout

- `src/tangd.c` — main + regex routing table (`s_dispatch`)
- `src/http.[ch]` — HTTP parsing (llhttp or legacy http-parser via
  `USE_LLHTTP`) and `http_reply()`
- `src/keys.[ch]` — key store: `read_keys()`, `find_jwk()`, `jwk_sign()`,
  advertisement payloads
- `src/pqc.[ch]` — hybrid ML-KEM protocol (`/adv-kem`, `/rec-kem`);
  `pqc.h` holds the protocol constants (`KEM_VERSION`, `ECMR_ALG`,
  `DIR_REQUEST`, `DIR_RESPONSE`)
- `src/kem_alg.h` — KEM algorithm list shared by `keys.c` and `pqc.c`;
  keep it self-contained (no `http.h` — avoid the layering inversion)
- `src/pqc_noop.c` — 404 stubs for builds without ML-KEM support
- `src/tangd-keygen.in`, `src/tangd-rotate-keys.in` — installed scripts
  (meson `configure_file`); they source `src/tangd-utils.sh` (shared
  jose algorithm-support detection)
- `tests/helpers` — shared shell helpers; single-sources the protocol
  constants from `src/pqc.h` (extracted with sed) — do not hardcode
  `"hybrid-v1"` and friends in tests
- `tests/test-pqc.c.in` — unit tests that `#include "pqc.c"` directly,
  with a stubbed `http_reply`
- `tests/adv`, `tests/rec` — shell test *libraries*, sourced by the
  `*-standalone` / `*-socat` variants
- `tests/noop` — e2e test for the no-PQC build (uses `tangd-noop`)

## Code conventions

- C99, `warning_level=3` with `werror=true`: new code must compile
  warning-free.
- Opening brace of a function on its own line; 4-space indent; no tabs.
- Cleanup attributes everywhere: `json_auto_t`,
  `__attribute__((cleanup(str_cleanup)))`,
  `cleanup_tang_keys_info`.
- Secret hygiene: KEM shared secrets must be cleansed before being freed
  (`OPENSSL_cleanse` / `OPENSSL_clear_free`; include the NUL terminator —
  use `len + 1` for C strings). Use the existing `wipe_jwk_secret()` /
  `wipe_and_release()` helpers.
- HTTP status mapping: 400 malformed request, 403 wrong key type/usage,
  404 unknown thumbprint, 500 internal errors. Shared path handling goes
  through `load_thp_keys()` (one status mapping for the KEM endpoints).

## Hybrid protocol notes (`/rec-kem`)

- Enriched JWE protected headers carry `kem_ver`, `suite`
  (`ECMR+ML-KEM-<variant>`), `kid` (ECMR thumbprint), `kem_kid`, `msg_dir`.
- `crit` lists exactly the four extension parameters — never `kid` (a
  standard JOSE header, RFC 7515 §4.1.4); unknown `crit` entries are
  rejected (RFC 7515 §4.1.11).
- Anti-downgrade: the response `suite` is derived from the server key's
  algorithm; the client-declared `suite`, `clevis_kem_pub->alg` and the
  path key algorithm must all agree.
- Validate the (plaintext, base64) protected header before doing crypto.
- Wire-format changes usually require a coordinated Clevis-side change;
  note it in the commit message.

## Git workflow

- `origin` = this fork, `upstream` = latchset/tang. Feature branches are
  rebased onto `upstream/master` before merge; keep the final tree
  content-identical across history rewrites (`git diff <backup> HEAD`
  must be empty).
- Commits are GPG-signed (`commit.gpgsign=true`). If the agent's commit
  fails with a pinentry timeout, the passphrase cache expired — retry, or
  commit unsigned and let the user amend-sign.
- Do **not** use `git add -A`: the worktree carries untracked local files
  (`.claude/`, `cscope.*`, `cppcheck-report.xml`, `sonar-project.properties`,
  `opencode-session-notes.md`, build dirs). Stage files explicitly.
- AI assistance is credited with `Co-Authored-By:` trailers (e.g.
  `Co-Authored-By: GLM 5.3 <noreply@z.ai>`).

## Testing notes

- Tests skip gracefully when jose lacks ML-KEM (`jose_supports_alg()` in
  `tests/helpers` uses the `jose alg` listing — do not probe with
  `jose jwk gen`).
- e2e tests start `tangd` from the build tree (found via `PATH`) on random
  ports.
- On machines whose jose supports ML-KEM, `tangd-noop` is built so the
  hidden-endpoint behavior of the no-PQC build is still exercised by
  `tests/noop`.
