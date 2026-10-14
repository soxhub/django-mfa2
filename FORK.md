# Why this fork exists

This is a soxhub fork of [mkalioby/django-mfa2](https://github.com/mkalioby/django-mfa2),
consumed by [soxhub/galaxy](https://github.com/soxhub/galaxy) as a git dependency pinned
to a commit on the `auditboard-customizations` branch. The branch currently tracks
upstream v3.2 (merged for `fido2` 2.x support, which modern `cryptography` releases
require) plus the customizations below.

## Customizations on top of upstream

1. **Forced FIDO2 re-verification mid-session** (`mfa/FIDO2.py`,
   `authenticate_complete`): upstream only logs the user in via the MFA flow when the
   request is unauthenticated. Galaxy sets `u2f_verify_this` in the session to force an
   already-authenticated user to re-verify with their security key (e.g. step-up auth
   after login), so the fork also completes the login flow when that session flag is set.
2. **Template tweaks**: a "Register a New Key" link on the FIDO2 recheck page
   (`mfa/templates/FIDO2/recheck.html`) and an explicit "Delete" label next to the
   trash icon on the key list (`mfa/templates/MFA.html`).
3. **U2F removed**: `mfa/U2F.py`, its URLs, templates, static JS, and tests are deleted,
   and `python-u2flib-server` is no longer a dependency. Galaxy only enables FIDO2, and
   u2flib's verify path calls `pubkey.verifier()`, which current `cryptography` removed.
   Galaxy prod had no U2F keys when this was removed (October 2026, FIDO2 keys only).
   The `u2f_verify_this`/`u2f_verified` session keys are FIDO2 step-up state and stay.
4. **PyJWT instead of python-jose**: Trusted Device signatures (`mfa/models.py`,
   `mfa/TrustedDevice.py`) use PyJWT. python-jose and its `ecdsa` dependency have
   unfixed CVEs. Both libraries produce the same HS256 tokens, so stored signatures and
   existing `deviceid` cookies still verify.

## Known upstream test failures

Upstream v3.2's suite does not pass cleanly. Some tests patch
`fido2.features.webauthn_json_mapping`, which fido2 2.2 removed (production code guards
it with `hasattr`), and several config, Trusted Device, and MFATestCase tests use
test-only URLconfs that do not define the `home` route the templates reverse. These are
test-only issues, present on pure upstream v3.2 too.

CI (`.github/workflows/tests.yml`) runs the suite and compares the failing tests with
`.github/known-test-failures.txt`. It fails on any new failure, and also when a known
failure starts to pass so the list stays accurate. Update the list in the same PR when
a change fixes or intentionally changes one of those tests.

## What it would take to drop the fork

The customizations are small, so getting back to upstream PyPI releases is mostly
integration work, not feature work:

1. **Upstream or replace the `u2f_verify_this` hook.** Either propose a "force
   re-verification" session flag upstream, or implement step-up verification in galaxy
   itself (e.g. wrap/override `authenticate_complete`, or use upstream's recheck
   machinery (`MFA_RECHECK`) if it covers the use case).
2. **Move the template tweaks into galaxy.** Django template resolution lets galaxy
   override `mfa/templates/*` by shipping its own copies in an app listed before `mfa`
   in `INSTALLED_APPS`; the two tweaks above are cosmetic and easy to carry there.

Customizations 3 and 4 (U2F and python-jose removal) would also need to go upstream, or
galaxy would get python-u2flib-server and python-jose back. Once that is done, galaxy can depend on `django-mfa2` from PyPI directly and this
repository can be archived.
