# Bug Tracking

## Tooling

Use **GitHub Issues** in the `mawood40/campsites` repository (or the canonical org repo if transferred). Labels and templates should be added when the project goes public; until then, freeform issues are acceptable.

## Labels (recommended)

| Label | Use |
|-------|-----|
| `bug` | Regressions or incorrect behavior vs PRD / FRD |
| `security` | Authz bypass, secret leak, unsafe upload, etc. |
| `priority/p0` | Data loss, auth broken, production down |
| `priority/p1` | Major feature broken, no workaround |
| `priority/p2` | Minor defect or cosmetic |
| `area/spa` | Frontend |
| `area/bff` | BFF service |
| `area/backend` | Backend service |
| `area/infra` | Docker, deploy, CI |

## Reporting expectations

1. **Repro steps** — environment (local compose vs deployed), user role, exact clicks or API calls.
2. **Expected vs actual** — cite PRD section or FRD if relevant.
3. **Logs** — request ID (`X-Request-ID`) from browser devtools or server logs; redact tokens and passwords.
4. **Screenshots** — for UI issues (optional but helpful).

## Triage

- Security issues: restrict visibility if GitHub supports private security advisories; otherwise contact maintainers directly until a policy exists.
- Link fixing PRs with `Fixes #N` in the commit or PR description for auto-close.

## Relation to testing

Failing **automated** tests should block merge on `main`; track flaky tests as `bug` + `area/ci` with high priority until stabilized.
