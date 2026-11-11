# Software Composition Analysis (SCA) Report: OWASP NodeGoat

## Overview
This repo documents an SCA scan I performed on [OWASP NodeGoat](https://github.com/OWASP/NodeGoat), an intentionally vulnerable Node.js application, as part of a Secure Software Development lab.

SCA checks the third-party libraries a project depends on against databases of known vulnerabilities (CVEs). It does **not** analyse the project's own source code (that is SAST).

## Scan Details

| Item | Value |
|---|---|
| Tool | Trivy v0.76.2 |
| Project | OWASP NodeGoat |
| Date | 11 Nov 2026 |
| Vulnerability DB | Version 2, updated 2026-11-10 |
| Environment | Windows 11, PowerShell |
| Target scanned | `package-lock.json` (npm) |

## Commands Used

```powershell
git clone https://github.com/OWASP/NodeGoat.git
cd NodeGoat
npm install
trivy fs . --format table --output sca_report.txt
trivy fs . --scanners license --format table --output license_report.txt
```

## Findings Summary (Before Remediation)

**Dependency vulnerabilities: 93 total**

| Severity | Count |
|---|---|
| Critical | 12 |
| High | 48 |
| Medium | 22 |
| Low | 11 |
| Unknown | 0 |

### Example findings

| Package | CVE | Severity | Installed | Fixed Version | Issue |
|---|---|---|---|---|---|
| body-parser | CVE-2024-45590 | High | 1.18.3 | 1.20.3 | Denial of Service |
| body-parser | CVE-2026-111181 | High | 1.18.3 | 2.4.0, 1.20.9 | Denial of Service via unhandled decompression stream error |

### Secret detected
Trivy also flagged **1 exposed secret (High)** in `artifacts/cert/server.key`: a private key committed inside the project. Hardcoded keys in a repository are a risk because anyone with repo access can use them. In a real project the key would be revoked, rotated, and removed from version control.

## License Compliance
Trivy's license scan (`--scanners license`) checked **353 packages** in `package-lock.json`:

| Classification | Count |
|---|---|
| Critical / High / Medium | 0 |
| Low | 347 |
| Unknown | 6 |

- Trivy rates permissive licenses (e.g. MIT, Apache-2.0, BSD) as Low risk, so the large majority of packages carry no significant license restrictions.
- No packages were flagged as restricted or forbidden (such as GPL/AGPL), so no copyleft conflicts were found.
- 6 packages have an **Unknown** license. These would need a manual check of each package's repository or `package.json` before using the project commercially.

## Evidence
![Trivy version](1_trivy_version.png)
![Before scan](2_before_totals.png)
![Report summary](3_report_summary.png)
![After scan](4_after_totals.png)

## Analysis
- Most findings come from outdated packages. NodeGoat is intentionally vulnerable, so a high count is expected.
- Many vulnerable packages are **transitive dependencies** (pulled in by other packages), not ones listed directly in `package.json`.
- Most findings have a fixed version available, so the main remediation is upgrading the affected package.

## Remediation
Steps taken:
1. Ran `npm audit fix` to apply safe automatic dependency upgrades.
2. Re-scanned with Trivy (`trivy fs . --format table --output sca_report_after.txt`) and compared the results.

| Severity | Before | After |
|---|---|---|
| **Total vulnerabilities** | **93** | **30** |
| Critical | 12 | 3 |
| High | 48 | 12 |
| Medium | 22 | 11 |
| Low | 11 | 4 |

`npm audit fix` removed 63 of the 93 findings (about 68%), including 9 of the 12 critical ones.

### Remaining issues
- 30 dependency vulnerabilities remain (3 critical, 12 high). These likely need major version upgrades that `npm audit fix` does not apply automatically, as they can break the application. Each would need to be reviewed and tested before upgrading.
- The exposed private key (`artifacts/cert/server.key`) is still flagged. It is not a dependency, so `npm audit fix` does not touch it. It would have to be removed from the repo and the key rotated.

## Lessons Learned
- SCA only finds **already-published** CVEs, so a clean result does not mean "perfectly secure".
- SCA and SAST cover different things. SCA covers libraries, SAST covers my own code, and both are needed.
- Dependencies should be re-scanned regularly because new CVEs appear constantly.
- Running SCA in a CI/CD pipeline catches vulnerable dependencies early, when fixes are cheapest.

## Files
- `sca_report.txt`: full vulnerability scan output
- `license_report.txt`: license scan output
- `1_trivy_version.png`, `2_before_totals.png`, `3_report_summary.png`, `4_after_totals.png`: screenshots of the scan
