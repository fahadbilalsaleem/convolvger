# Security policy

## Supported versions

Only the latest release on PyPI receives security fixes. Fixes are
released as a new version; earlier versions are not patched.

## Reporting a vulnerability

Please report vulnerabilities privately through GitHub:
<https://github.com/fahadbilalsaleem/convolvger/security/advisories/new>.
Do not open a public issue.

Include the version (`convolvger --version`), the command you ran, and
the smallest input that shows the problem. Please do not include real
conversations or anyone's identity: invent the text and ids, as the
project's own test fixtures do.

Convolvger has one maintainer, so replies may take some days. You will
hear back before any fix is published, and you will be credited in the
release notes unless you ask not to be.

## Scope

[docs/threat-model.md](docs/threat-model.md) describes where untrusted
input enters, which components matter most, and what is out of scope.
