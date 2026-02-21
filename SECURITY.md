# Security Policy

## Supported Versions

Security fixes are applied to the latest `main` branch.

## Reporting a Vulnerability

Please do not open public issues for security vulnerabilities.

Report security concerns privately via GitHub Security Advisories:
- Go to `Security` tab in this repository
- Click `Report a vulnerability`

Include:
- A clear description of the issue
- Steps to reproduce
- Potential impact

You can expect an initial response within 5 business days.

## Local Threat Model Notes

### Session Theft

Risk:
- Telegram session artifacts (`telegram.session`, `.data/web.session.txt`) can be abused if copied.

Mitigations:
- Keep session files on trusted local storage only.
- Restrict file permissions for session directories.
- Rotate/recreate sessions if compromise is suspected.

### Exposed Output Files

Risk:
- Generated markdown exports may contain sensitive conversation content.

Mitigations:
- Store exports in private directories.
- Avoid syncing output folders to public cloud drives by default.
- Remove old exports that are no longer needed.

### Insecure Docker Runtime Defaults

Risk:
- Running containers as root with broad volume mounts can leak credentials and outputs.

Mitigations:
- Prefer read-only mounts where possible.
- Avoid mounting entire home directories.
- Run with least-privilege user when containerizing production-like setups.

## Credential Handling

- Keep `TG_API_ID` and `TG_API_HASH` in environment variables or local secret stores.
- Never commit `.env` files with real credentials.
- Treat imported/exported session strings as credentials.
