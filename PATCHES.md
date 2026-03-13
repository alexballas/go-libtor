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

## Checklist when updating Tor or OpenSSL

1. Replace or regenerate the vendored upstream trees.
2. Review `linux/openssl/.gitignore` and confirm required generated build files
   are still tracked in git.
3. Run a clean build with a fresh module/build cache and check for missing
   generated headers or generated `.c`/`.inc` files.
4. Re-check whether the warning-reduction patches are still needed, still
   correct, or have been superseded upstream.
5. If upstream changed the relevant files, prefer dropping local edits instead
   of carrying them forward automatically.
6. Tag a release only after confirming that a fresh consumer build works
   without a local `replace`.
