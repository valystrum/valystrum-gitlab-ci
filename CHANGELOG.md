# Changelog

Each release is a tag of this project and a version of its components in the GitLab CI/CD Catalog. Pin the version in
your `include: component` line (`@1.0.0`), or use `@~latest` for the newest release.

## 1.0.0 (2026-10-29)

The first release, for Valystrum v0.21.0 and later. Needs GitLab 17.0 or later.

- `security-scan`: leaked secrets, code analysis (SAST) and IaC checks on a merge request's changed files, the findings
  in the merge request's Code Quality widget, and the job failed on a critical or high finding.
- `sbom`: a CycloneDX SBOM made with the official CycloneDX generators (pinned, their dependencies locked by hash) and
  uploaded to your Valystrum software inventory, with a `complete` input that removes what the SBOM no longer lists.
