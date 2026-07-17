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

## Known upstream test issues with fido2 2.2.x

Upstream v3.2 declares support for `fido2 >= 1.1.1, < 2.3`, but its test suite has a
handful of failures under fido2 2.2.x (tests patch `fido2.features.webauthn_json_mapping`,
which fido2 2.2 removed, plus one error-message assertion). Production code guards that
attribute with `hasattr`, so these are test-only issues. Verified identical failure sets
on pure upstream v3.2 and on this branch.

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

Once 1 and 2 are done, galaxy can depend on `django-mfa2` from PyPI directly and this
repository can be archived.
