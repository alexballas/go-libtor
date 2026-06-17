# Local Patches

`go-libtor` vendors upstream Tor, OpenSSL, libevent, and zlib source trees.
Most updates should prefer replacing vendored source from upstream cleanly, but
this repository currently carries a small patch set that must be reviewed when
those upstreams are bumped.

## Current patch set

### Release packaging for generated OpenSSL files

Some generated OpenSSL files are required at Go module build time. They were
previously ignored by `linux/openssl/.gitignore`, which meant tagged releases
could omit files that the CGO build includes directly.

The tracked generated files currently include:

- `linux/openssl/include/crypto/bn_conf.h`
- `linux/openssl/include/crypto/dso_conf.h`
- `linux/openssl/include/openssl/cmp.h`
- `linux/openssl/include/openssl/configuration.h`
- `linux/openssl/include/openssl/core_names.h`
- `linux/openssl/include/openssl/crmf.h`
- `linux/openssl/include/openssl/ess.h`
- `linux/openssl/include/openssl/fipskey.h`
- `linux/openssl/include/openssl/x509_acert.h`
- `linux/openssl/providers/common/include/prov/der_*.h`
- `linux/openssl/providers/implementations/include/prov/blake2_params.inc`
- `linux/openssl/providers/implementations/keymgmt/lms_kmgmt.c`
- `linux/openssl/providers/implementations/rands/fips_crng_test.c`
- `linux/openssl/providers/implementations/storemgmt/winstore_store.c`

If an OpenSSL update regenerates these files, keep them tracked in the module
release. Do not move them back under `.gitignore` unless the CGO wrappers stop
including them.

Two of these are easy to miss because `go run build/wrap.go` does **not**
produce them on its own:

- `providers/common/include/prov/der_*.h` are generated on demand during a full
  OpenSSL build, **not** by `make build_generated` (which `wrap.go` runs). After
  regenerating, generate them explicitly from a matching upstream checkout, e.g.
  `./Configure linux-x86_64 no-shared no-zlib no-asm no-async no-sctp && make
  build_generated && make providers/common/include/prov/der_digests.h ...` for
  each of `der_digests der_dsa der_ec der_ecx der_hkdf der_ml_dsa der_rsa
  der_slh_dsa der_sm2 der_wrap`, then copy them into
  `linux/openssl/providers/common/include/prov/`. (These are pure ASN.1/OID
  maps, so they are stable across OpenSSL patch releases.)
- `openssl_config/gotor_extra.h` is a hand-maintained file, but `wrapOpenSSL`
  does `os.RemoveAll("openssl_config")` and only rewrites the auto-generated
  headers. Restore it after regenerating (`git checkout -- openssl_config/gotor_extra.h`),
  otherwise the build fails immediately on the `-include .../gotor_extra.h` flag.

### Warning-noise reduction for vendored Linux builds

The repository also carries a small warning-reduction patch set:

- `libtor/linux_tor_preamble.go` and `build/wrap.go` add
  `-Wno-deprecated-declarations` for the vendored C build. This suppresses the
  large volume of OpenSSL 3 deprecation warnings emitted by Tor 0.4.9.x.
- Several vendored OpenSSL files under `linux/openssl/...` now guard their
  local `_GNU_SOURCE` definition with `#ifndef _GNU_SOURCE` so the global CGO
  define does not emit redefinition warnings.
- Vendored Tor `byteorder.h` copies now guard `_le64toh` so
  `tor_config/gotor_extra.h` can provide the helper without macro redefinition
  warnings.

These are intentionally narrow source edits. They should not change runtime
behavior, but they are still local drift from upstream.

### Generator fixes in `build/wrap.go`

- `wrapOpenSSL`'s "generated deps" loop used an out-of-scope `err` (the
  function-level value from the earlier `make --dry-run`) when checking the
  result of `os.Stat(source)`. Whenever any detected source `.c` was absent at
  that point, `os.IsNotExist(nil)` was false and the function returned early
  with a `nil` error, silently skipping the OpenSSL wipe and wrapper
  generation. OpenSSL 3.6.1 happened to have every `.c` present after
  `build_generated`, so it was masked; 3.6.3 hit the empty branch and produced
  zero `linux_openssl_*.go` wrappers. Fixed by keeping `err` in scope with
  `else if`.

### Tor version reported by the embedded build

The per-architecture `config/tor/orconfig.*.h` templates previously hardcoded
the Tor version (`VERSION`, `PACKAGE_VERSION`, `PACKAGE_STRING`). `wrap.go`
renders these templates with `.StrVer` (the version detected from the upstream
`./configure`), so they now use `{{.StrVer}}` and the compiled-in version stays
in sync automatically. `config/tor/micro-revision.i` is still a static string
and should be set to the abbreviated commit of the vendored Tor release.

## Checklist when updating Tor or OpenSSL

1. Bump the commit(s) in `lock.json` and regenerate the vendored trees
   (`go run build/wrap.go --nobuild`). Tor is pinned by `lock.json`, so set its
   commit there before regenerating.
2. Restore `openssl_config/gotor_extra.h` and generate the missing
   `providers/common/include/prov/der_*.h` headers (see above).
3. Re-apply the `_GNU_SOURCE` / `_le64toh` warning guards to the freshly
   regenerated OpenSSL/Tor source files.
4. Review `linux/openssl/.gitignore` and confirm required generated build files
   are still tracked in git.
5. Run a clean build with a fresh build cache (`go clean -cache && go build .`).
   A warm cache will not pick up changes to `#include`d `.c`/`.h` files (they
   are not Go package sources), so a clean build is required to validate a bump.
6. Confirm the embedded Tor reports the expected version at runtime
   (`GETINFO version` over the control port) — this catches stale
   `config/tor/orconfig.*.h` version strings and stale cached objects.
7. Re-check whether the warning-reduction patches are still needed, still
   correct, or have been superseded upstream. If upstream changed the relevant
   files, prefer dropping local edits instead of carrying them forward.
8. Tag a release only after confirming that a fresh consumer build works
   without a local `replace`.
