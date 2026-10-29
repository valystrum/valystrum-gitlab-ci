# Valystrum for GitLab CI/CD

Two GitLab CI/CD components:

- **`security-scan`** scans each merge request for **leaked secrets**, **code weaknesses (SAST)** and **IaC
  misconfiguration** (Terraform, CloudFormation, Azure Resource Manager templates, Kubernetes, Helm, Dockerfile),
  shows the findings in the merge request's Code Quality widget, and fails the job on a critical or high finding.
- **`sbom`** makes a **CycloneDX** bill of materials with the official CycloneDX generators and uploads it to your
  Valystrum software inventory, where its components are matched against CVE feeds and your licence policy.

Both are part of the Gold plan and above. This project is licensed under the Apache License 2.0 ([LICENSE](LICENSE));
the Valystrum platform it calls is not. The GitHub equivalents are `valystrum/valystrum-pipeline` and
`valystrum/valystrum-sbom`; the request, the signature, the retries and the scan's fail-open rule are the same. Releases
are listed in [CHANGELOG.md](CHANGELOG.md).

> **Status, 2026-10-29.** Written for Valystrum v0.21.0, live on app.valystrum.io since 2026-10-29. So far these
> components have run only under `gitlab-ci-local`, not on a GitLab instance: the first customer pipeline is their
> first real run.


## Quick start

```yaml
include:
  - component: $CI_SERVER_FQDN/valystrum/valystrum-gitlab-ci/security-scan@1.0.0
    inputs:
      url: https://app.valystrum.io/api/v1/<org>/pipeline/scan
  - component: $CI_SERVER_FQDN/valystrum/valystrum-gitlab-ci/sbom@1.0.0
    inputs:
      url: https://app.valystrum.io/api/v1/<org>/sbom/ingest
      complete: true

valystrum-sbom:
  before_script:
    - npm ci --ignore-scripts --no-audit --no-fund
```

[`examples/.gitlab-ci.yml`](examples/.gitlab-ci.yml) is a complete pipeline to copy, with the `workflow:` rules and the
`stages:` list that step 3 explains.

## Setup, from the key to the first green pipeline

**Requirements.** A GitLab version that has CI/CD components (`include: component`, generally available from GitLab
17.0) and, for step 2's **Masked and hidden** visibility, GitLab 17.4 or later. A Linux runner whose executor runs the
job's `image` (Docker or Kubernetes); on a shell executor the machine itself needs bash, git, curl and openssl, and jq
unless it is x86_64 or aarch64 and can reach github.com. Each job writes its script to a file of its own under
`TMPDIR` (`/tmp` by default), runs it and removes it, so jobs that run at once on one machine do not meet. Besides your Valystrum URL, the jobs reach github.com when
their image has no jq (see the scan's `image` input), and the SBOM job reaches the package sources of each generator it
runs: your npm registry, PyPI (or the index pip is configured with), the Go module proxy and sum.golang.org, or your
Maven repositories.

**1. Issue the project's CI key in Valystrum.** An organisation owner or admin opens **Settings, CI pipeline keys**
and issues a key for the project's full path exactly as GitLab names it in `CI_PROJECT_PATH`, for example
`acme/platform/web-api` (nested groups included, case included). The secret (`cs_wh_...`) is shown once. The same key
signs the scan and the SBOM upload. Each project takes one of the plan's repository slots; rotating a key does not.
Every key names one project: there is no organisation-wide key, so a project without a key of its own is refused with
HTTP 401 `Invalid signature: sign with this repository's CI key`, on the scan and on the SBOM upload alike. Over the API, from a signed-in admin
session:

```bash
curl -X POST https://app.valystrum.io/api/v1/<org>/pipeline/config \
  -H "content-type: application/json" -H "cookie: <your signed-in session cookie>" \
  -d '{"repo": "acme/platform/web-api"}'
```

**2. Store the secret as a CI/CD variable** (project **Settings, CI/CD, Variables**) named `CS_WEBHOOK_SECRET`, with
**Masked and hidden** visibility. Leave **Protect variable** off: a protected variable reaches only pipelines of
protected branches, so a merge request from a feature branch would get none and its scan job would fail with
`VALYSTRUM_SECRET is empty`. Anyone who can run a pipeline in the project can use the key. It signs only this
project's scans and SBOM uploads, and it reads back only what those two calls answer: a scan's id, verdict, findings
and the checks that ran, with a blast radius taken from your Valystrum inventory (the id of a cloud resource whose
name or provider id, or else an asset whose name, contains the resource the most severe IaC finding names, how many
direct relations it has, the number of your organisation's failing automated controls, counted up to ten, and the
codes of up to five of them), and an upload's counts with the id of the asset its components were linked to. The jobs
also take a variable named `VALYSTRUM_SECRET` if you prefer that name.

**Rotating and revoking the key.** Issuing a key again for the same project in **Settings, CI pipeline keys** rotates
its secret. The old secret stops working at once, so update `CS_WEBHOOK_SECRET` straight away. Until you do, the scan
job passes with an error line in its log (with `fail-on-error: true` it fails) and the SBOM job fails with HTTP 401
`Invalid signature`. Over the API, a second `POST .../pipeline/config` for the same project leaves its secret alone
unless the body carries `"regenerateSecret": true`. The same page revokes a key
(`DELETE /api/v1/<org>/pipeline/config/<keyId>`), which returns its repository slot. Gold includes 100 repositories and
Enterprise a custom number. From Gold, 25 more are an add-on at €100 a month or €1,000 a year.

**3. Include the components** as in the quick start, with `<org>` replaced by your organisation's slug.
**Settings, CI pipeline keys** shows the scan's URL; the SBOM's is the same with `sbom/ingest` in place of
`pipeline/scan`. Pin a released version (`@1.0.0`), not `@~latest`, so a new release reaches your pipelines only when
you move the pin.

The scan job runs in merge request pipelines, so the project must create them. If your `.gitlab-ci.yml` has
`workflow: rules`, they must let `merge_request_event` pipelines through, or the scan never runs on a merge request
and nothing fails to tell you. Without any workflow rules, a push to a branch with an open merge request starts both a
branch pipeline and a merge request pipeline; the `workflow:` block in
[`examples/.gitlab-ci.yml`](examples/.gitlab-ci.yml) starts one or the other, never both. Both jobs run in the `test`
stage unless you set `stage`, and a `stages:` list of your own must contain that stage.

GitLab adds a job to a merge request pipeline only when the job's own `rules` (or `only`) let `merge_request_event`
in; a job without them runs in branch and tag pipelines alone. With the example's `workflow:`, a branch with an open
merge request gets no branch pipeline, so your own jobs that have no `rules` stop running for it: give them rules as
well, for example `- if: $CI_PIPELINE_SOURCE == "merge_request_event"` and `- if: $CI_COMMIT_BRANCH`. The example's
`workflow:` also starts no pipeline for a tag: add `- if: $CI_COMMIT_TAG` to it if you build or release from tags.

**4. Make the scan a merge gate.** In **Settings, Merge requests**, turn on **Pipelines must succeed**. The scan job
fails on a critical or high finding; only that setting stops the merge.

**5. Open a merge request.** The `valystrum-security-scan` job runs in the merge request pipeline and on the default
branch, and `valystrum-sbom` runs on the default branch. A green first run reads `Scan complete: 0 finding(s),
passed=true.` and `Ingested N components into Valystrum as source 'acme/platform/web-api'; removed 0 it no longer
lists.` The scan then shows in Valystrum's **Dev Pipeline** module, its findings in **Findings**, and the components
under **SBOM & Licences**. **Settings, CI pipeline keys** shows when each key was last used: a key still marked never
used after the jobs ran means their requests named another path, or another organisation's slug in the URL (see HTTP
401 `Invalid signature` under [Troubleshooting](#troubleshooting)). A key is marked used before its signature
is checked, so a wrong secret marks it too.

### On another GitLab instance

A CI/CD component can only be included from the instance it lives on. On a self-managed instance, either mirror this
project there and include it from your mirror, as a component by your own path
(`$CI_SERVER_FQDN/<your group>/valystrum-gitlab-ci/security-scan@1.0.0`) or by project
(`include: project: <your group>/valystrum-gitlab-ci`, `ref: 1.0.0`, `file: templates/security-scan.yml`, with the
same `inputs:`), or include the file by URL and pin its content:

```yaml
include:
  - remote: https://gitlab.com/valystrum/valystrum-gitlab-ci/-/raw/1.0.0/templates/security-scan.yml
    integrity: sha256-<base64 SHA-256 of that file>   # GitLab 17.9 and later
    inputs:
      url: https://app.valystrum.io/api/v1/<org>/pipeline/scan
```

## The security scan

### What it checks

| Check | Engine | Covers |
|---|---|---|
| Code analysis (SAST) | Opengrep over a licence-clean rule pack (GitLab's MIT, Apache-2.0 and LGPL-3.0 rules) | JavaScript and TypeScript, Python, Java, Go, C#, Scala, Kotlin, Swift, Objective-C |
| IaC | Trivy's embedded misconfiguration checks, medium and above | Terraform, CloudFormation, Azure Resource Manager templates (JSON), Kubernetes, Helm, Dockerfile |
| Secrets | Valystrum's secret scanner, over the lines the change adds | AWS keys, GitHub and Slack tokens, Stripe live keys, private keys, Google API keys, JWTs, generic API keys |

Ruby, PHP and C are not analysed: the licence-clean rule pack Valystrum uses has no rules for Ruby or PHP, and its C
rules are under a licence Valystrum does not ship.

Code analysis and the IaC checks read each changed file whole, so a critical or high weakness already in a file the
merge request touches fails the job too, even on a line the change did not touch. Secrets are looked for only in the
lines the change adds, so a secret already in the file is not reported. The secret scan skips lockfiles,
`node_modules/`, `vendor/`, `dist/`, `build/`, `out/`, `coverage/` and `.next/`, minified, `.snap` and `.map` files,
and `*.example`, `*.sample`, `*.template` and `*.dist` files, and it ignores values that read as placeholders
(`example`, `xxxx`, `placeholder`, `changeme`, `your_key`, `dummy`, `redacted`, `<...>`). AWS access keys, GitHub
tokens, Stripe live keys and private keys are critical, and Google API keys and Slack tokens are high. JWTs and generic
API keys are medium, so they are reported but do not fail the job. The two engines run side by side with 20 seconds
each, and the platform waits 25 seconds, so a large change set can come back as `timeout`.

Code analysis severities map from Opengrep's to Valystrum's four: ERROR is high, WARNING (and a rule's MEDIUM) is
medium, INFO is low, and a rule tagged with a directly exploitable weakness (CWE-78, 89, 94, 95, 502, 798 or 918:
command, SQL, code and eval injection, unsafe deserialization, hardcoded credentials, SSRF) is raised to critical. An
IaC finding keeps Trivy's severity. So an ERROR rule's finding fails the job, and a WARNING rule's is reported only,
unless its weakness is one of those raised to critical.

### Scan inputs

| Input | Default | What it does |
|---|---|---|
| `url` | required | `https://app.valystrum.io/api/v1/<org>/pipeline/scan` |
| `repo` | `$CI_PROJECT_PATH` | The project as its CI key names it. It picks the key, so it must match exactly |
| `paths` | empty | Space-separated pathspecs that narrow the diff, and so the secret scan (`infra/ app/`). The files sent for code analysis are every changed file all the same |
| `send-file-contents` | `true` | Send the changed files, which code analysis and the IaC checks need. Off, only secrets are scanned |
| `fail-on-findings` | `true` | Fail the job on a critical or high finding. Off, the job only reports |
| `fail-on-error` | `false` | Also fail the job when the scan could not run (no jq, no answer, a refusal such as a wrong secret), the platform could not complete it, or its code and IaC scanner did not finish the files sent |
| `stage` | `test` | The stage the job runs in |
| `image` | `buildpack-deps:trixie-scm` by digest | The job's image: it needs bash, git, curl and openssl. The default has no jq, so the job fetches jq 1.8.2 from github.com on every run and checks it against its SHA-256 (x86_64 and aarch64 only); a runner that cannot reach github.com needs an image with jq. Pin your own by digest too |

To change anything else (rules, `needs`, `tags`, `allow_failure`), define a job named `valystrum-security-scan` in your
`.gitlab-ci.yml`: GitLab merges its keys over the component's. Keywords under your pipeline's `default:` (a
`before_script`, `after_script`, `cache`, `services`, `tags`) apply to both jobs too. If your default `before_script`
needs a toolchain the scan's image lacks, give `valystrum-security-scan` its own `before_script: []`, or
`inherit: default: false`.

### What it compares, and the branch

In a merge request pipeline the job diffs the source branch against the commit it shares with the target branch
(`CI_MERGE_REQUEST_DIFF_BASE_SHA`), and sends the source branch (`CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`) and the merge
request's number (`CI_MERGE_REQUEST_IID`). On a push to the default branch it diffs against the commit before the
push. It also sends the pipeline's id (`CI_PIPELINE_ID`), the project's default branch (`CI_DEFAULT_BRANCH`) and the
commit's time: Valystrum orders a branch's scans by the pipeline id, and a scan of the default branch decides a
finding once it has read the file. A shallow clone (GitLab's default depth is 20) is deepened first, so the diff has
its base. A merge request from a fork sends the fork's branch under its own name, so when you run fork pipelines in
the parent project, a fork's `main` (or whatever your default branch is called) counts as your default branch and
decides findings. Run them only from a fork branch with another name.

In a merged results pipeline (GitLab Premium and Ultimate), the job runs on the merge of the source branch into the
target branch, so the files it reads are the merged result. The commit it sends is the source branch's head
(`CI_MERGE_REQUEST_SOURCE_BRANCH_SHA`).

Each scan that reads a file in full is its branch's read of it. Once the default branch has read a file, its scans
decide that file's findings: a fix closes a finding when it reaches the default branch, and a merge request or an old
branch that still carries the finding neither reopens nor holds it. Until the default branch has read the file, every
branch that read it holds the finding while its latest scan still finds it, so a fix on a merge request closes a
finding only when the default branch has not read that file and no other branch still finds it. Scanning the default
branch as well (the component does) is how it reads the code. A pipeline that finishes after a newer one of the same
branch cannot reopen what the newer scan closed, and a retried job keeps its pipeline's id.

A scan closes a finding only when it reads the finding's file and no longer finds it there. A code or IaC finding
needs a file the code and IaC scanner read to the end: with `send-file-contents: false`, for a file over the size caps
or a binary file, or when that scanner timed out or failed, the file is not read and nothing in it closes. A secret
needs the whole file sent, which Valystrum reads again in full (the scan itself reads only the added lines); a file the
secret scan skips, such as a lockfile, is never read for secrets. A file the change deletes is read for every kind. A
closed finding is resolved with a dated note (`Fixed: not found by the code scan of 28 Oct 2026`), its open
remediation items move to ready for verification, for a person to verify and close, and a linked Jira or ServiceNow
ticket gets the note as a comment. A finding a person reopened stays open until a scan finds it again, and only a later
scan that no longer finds it closes it. A resolved finding that a scan finds again reopens, and its items waiting for
verification go back to in progress, with a comment on their ticket.

A leaked secret is never closed by a scan: taking it out of the code does not close its finding, since whoever copied
it still has it. Once the deciding scan no longer sees it, the finding stays open with the note `No longer visible to
the code scan of 28 Oct 2026: confirm you rotated it, then close it`, its open remediation items move to ready for
verification as a fixed finding's do, and you then close it yourself. A file the change deletes closes its other
findings by the same branch rule as a fix. A finding someone marked accepted risk or a false positive keeps that
verdict. A finding that only a merge request's branch ever carried stays open after that merge request is closed
without merging: it closes when a scan of the default branch reads the file without it (a push there that changes the
file), or when you close it in **Findings**.

A file the change renames or moves takes its findings to its new path, and if that move fails, the project's next scan
finishes it. Git has to report the change as a rename (the `rename from` and `rename to` lines of the diff). A file
edited so much that git reads it as a deletion and a new file, or one moved into or out of the `paths` you narrow the
diff to, is not moved: the new path's findings are raised afresh, and the old path's close as a deleted file's when the
diff shows it deleted (a file moved into your `paths` from outside them leaves them open, since the diff does not show
its old path). Only a scan of a branch that decides the finding moves it, so once the default branch has read the old
file, a merge request that renames it moves nothing and raises what it still finds under the new path as new findings;
when the change reaches the default branch, the old path's open findings close as fixed although the code only moved.
A finding is not moved onto a path that already has a finding of the same rule. A finding you accepted or marked a
false positive moves with its verdict; where a file's findings do not move, the verdict stays on the old path.

### What blocks a merge, and what does not

The job fails when the scan reports any **critical or high** finding (with `fail-on-findings: true`). A scanner
outage never blocks you unless you ask it to: when the job has no jq and cannot fetch one, when the scan endpoint
cannot be reached or refuses the request, when the platform answers that it could not complete the scan, or when its
code and IaC scanner did not finish the files sent, the job passes, and its log says what did not run.
`fail-on-error: true` fails the job in all four cases. Turn it on once the setup works, or a rotated secret or a plan
change turns the gate off without failing a pipeline.

`fail-on-error` does not cover a change too large to send whole. Files over the caps (see
[What is sent, and what never is](#what-is-sent-and-what-never-is)) are not sent and so not analysed; the log counts
them (`Sending N file(s) for code analysis, M skipped (size/binary/cap)`), and the scanner itself skips a file over
512,000 bytes. A request over 2 MB is sent without the files, with a warning, and only secrets are checked. In all of
these the job passes on what was checked.

Each scan is judged on its own findings. Marking a finding accepted risk or a false positive in Valystrum keeps that
verdict on the finding there (a resolved one reopens when a scan finds it again), but no verdict changes the gate: the
next change to that file that still trips the rule fails the job again.

### Where the findings show

- The merge request's **Code Quality** widget, from the `codequality` report (critical as blocker, high as critical,
  medium as major, low as minor). Inline annotations in the diff need a GitLab tier that has them. GitLab's widget
  compares the merge request's report with the target branch's latest one and lists only the difference (findings
  are matched by rule, file and line). Each scan's report covers only its own change, so the widget can list a
  finding of the default branch's last push as fixed, and leaves out one the target branch's report already had. The
  job log and `summary.md` list all of the scan's findings.
- The job log: a summary table of the 20 most severe findings and one line for every finding
  (`CRITICAL SECRET-aws_access_key deploy.env:1 ...`).
- `valystrum-reports/` as a job artifact, kept for 30 days, failed runs included (`summary.md` with the 20 most severe
  findings, the Code Quality report), and the variables `VALYSTRUM_PASSED`, `VALYSTRUM_FINDINGS` and
  `VALYSTRUM_SCAN_ID` for later jobs (a `dotenv` report). `VALYSTRUM_PASSED` reads `true` and `VALYSTRUM_FINDINGS`
  `0` also when nothing was checked: when the scan did not run, could not be completed, or had no changes to scan
  (`VALYSTRUM_SCAN_ID` is then empty, or names the scan that could not complete). No variable says which checks ran,
  so read `summary.md`, or set `fail-on-error: true`, before treating `VALYSTRUM_PASSED` as a clean scan. GitLab
  passes the variables to jobs of later stages and to jobs that name `valystrum-security-scan` in `needs`; after a
  scan that failed on findings, a later job runs only with `when: always` or `when: on_failure`.
- Valystrum: the **Dev Pipeline** module and **Findings**, one finding per project, rule and file. Dev Pipeline says
  why each open finding is still open: still there, keeps coming back, rotation to confirm (a secret), reopened by a
  person, on another branch (naming the branches that still have it), no branch on record, or not rechecked (no later
  scan of its branch read its file in full). A leaked secret also lands in **Data Security, Secrets**.

A detected secret is only ever shown as a masked preview (`AKIA••••3456`), in the job log and in Valystrum. On a public
project, anyone can read job logs and artifacts unless **Project-based pipeline visibility** (formerly **Public
pipelines**) is off (**Settings, CI/CD, General pipelines**). The log and `valystrum-reports/` name each finding's rule, file and line, with a secret shown only as its
masked preview.

No comment is posted on the merge request: the GitHub action's pull request comment needs a GitHub token, which these
components do not send. The blast radius shows on the scan's page in Valystrum's **Dev Pipeline** module.

## The SBOM upload

The job runs on the default branch only. Unlike the scan, it does not fail open and has no `fail-on-error`: it fails
the job on every way it can go wrong, so a missing inventory shows red instead of passing as an empty success. That
covers no manifest, 0 components, a generator that fails, jq that cannot be fetched, and an upload refused, or
unanswered after its four attempts. If that must not fail the pipeline, set `allow_failure: true` on the
`valystrum-sbom` job.

New components are checked against the vulnerabilities Valystrum already holds as soon as they are uploaded, and the
nightly match adds the ones it learns of later. Each component's declared licence is resolved to an SPDX id and
checked against your organisation's licence policy; a licence the policy flags raises one finding per source and
licence, not one per package, and the default policy scores an unrecognised licence high. When `asset` (or the source
label) names an asset, the components are linked to it and to the service it belongs to, both as edges on the
security graph.

### SBOM inputs

| Input | Default | What it does |
|---|---|---|
| `url` | required | `https://app.valystrum.io/api/v1/<org>/sbom/ingest` |
| `source` | `$CI_PROJECT_PATH` | The label the components are grouped under. It also picks the CI key, so it must equal the project the key was issued for |
| `path` | `.` | The directory holding the manifests. Its subdirectories are not searched |
| `ecosystem` | `auto` | Every generator whose manifest is present, or a space-separated subset of `npm python go maven` |
| `asset` | empty | The name or hostname of the Valystrum asset these components belong to. Empty, they are linked to the asset whose name or hostname equals the source (case aside), or to none |
| `complete` | `false` | `true` when this job's SBOM is the whole of the source: the source's components it no longer lists are removed in Valystrum and their CVE and licence findings resolve, provided every component in it has a version (a bill with an unversioned component only adds). Leave it `false` when several jobs upload parts of one source |
| `stage` | `test` | The stage the job runs in |
| `image` | `node:26.12.1-trixie@sha256:b5f0e27e7423ea242b37af44c8c856e28f7e37d4fac23cb430cb618874ff8517` | An image with your project's toolchain (node and npm, python3 with venv, go, or mvn), bash, curl and openssl. The npm generator needs Node.js 20.18 or later and npm 9 or later |

A `requirements.txt` whose lines are not pinned with `==` gives components without a version, so with `complete: true`
its upload still only adds: it is accepted as usual, nothing is removed, and `VALYSTRUM_REMOVED` reads 0. With
`ecosystem: auto`, every manifest found needs its toolchain in the image (python3 with venv, go or mvn), or the job
fails. The default image is Node.js's, so set `ecosystem` to the ecosystems it has, or set `image` to one that has the
others.

The job runs one at a time (`resource_group: valystrum-sbom`), so two uploads never overlap. It does not order them:
when an older pipeline's job runs after a newer one's (GitLab's default `unordered` process mode, or a retried job),
its `complete` upload puts the older set back until the next pipeline of the default branch. To run waiting uploads in
pipeline order, set the resource group's process mode to `oldest_first` with GitLab's API
(`PUT /projects/:id/resource_groups/valystrum-sbom`). Install your dependencies first, in a `before_script` on the
`valystrum-sbom` job: the npm generator reads the installed `node_modules`, not the lockfile alone, and an SBOM with no
components fails the job.

### The generators

| Manifest | Generator | How it and everything it installs are locked |
|---|---|---|
| `package-lock.json` | `@cyclonedx/cyclonedx-npm` 6.0.1 | its whole tree from a lock file carried in the script (`npm ci`), each package by its SHA-512, fetched through your npm registry; nothing is resolved |
| `poetry.lock`, `Pipfile.lock` or `requirements.txt` (the first found) | `cyclonedx-bom` 7.6.0 | its whole tree from a hashed requirements file carried in the script (`pip --require-hashes --no-deps`), each wheel by its SHA-256, in a virtual environment; nothing is resolved |
| `go.mod` | `cyclonedx-gomod` 1.12.0 | `go run` at that version builds from the module's own `go.sum`, every module checked against Go's checksum database (sum.golang.org) |
| `pom.xml` | `cyclonedx-maven-plugin` 2.9.3 | the plugin's version fixes every dependency's version, each checked against the repository's checksums (`--strict-checksums`) |

Each generator is pinned to its newest release of 2026-10-28. The Python lock covers CPython 3.9 and later on Linux
(x86_64 and aarch64); the npm generator runs without its optional schema validators, which Valystrum does not need
because it reads the components itself. Several ecosystems in one directory are merged into
one component list. The SBOM is kept as the job's `cyclonedx` report (`valystrum-reports/sbom.cdx.json`), only when the
job succeeds and for your instance's default artifact expiry, and the job sets `VALYSTRUM_COMPONENTS` and
`VALYSTRUM_REMOVED` for later jobs.

Not read: Yarn and pnpm (the npm generator runs only beside a `package-lock.json`), `npm-shrinkwrap.json` alone,
Gradle, .NET, Cargo, Composer and RubyGems. A directory with none of the four manifests fails the job with
`No supported manifest found`. Only the npm generator needs the dependencies installed first; the Python generator
reads its lock or requirements file, and the Go and Maven generators resolve the modules themselves.

The npm generator is installed in a directory of its own, through the registry your project's npm configuration
names. A registry that needs credentials needs them in the user configuration (`~/.npmrc`), not only in the
project's `.npmrc`: copy them there in the job's `before_script`, for example `cp .npmrc ~/.npmrc`.

### Several directories

The component's job is always named `valystrum-sbom`, so including the component twice gives one job, not two. The
component's own job reads the directory its `path` input names; for manifests in further directories, add a job per
directory that extends it and installs that directory's dependencies:

```yaml
valystrum-sbom-billing:
  extends: valystrum-sbom
  variables:
    VALYSTRUM_PATH: services/billing
  before_script:
    - (cd services/billing && npm ci --ignore-scripts --no-audit --no-fund)
```

Each such job uploads under the project's one source, so leave `complete` false on all of them. Otherwise each upload
removes the others' components. To remove what is gone, either scan once from a manifest that covers every module (a
Maven aggregator `pom.xml` is built with `makeAggregateBom`) with `complete: true`, or give each module its own source.
In that case, issue a key for exactly that label (for example `acme/mono/billing`; each label takes a repository
slot), store its secret as a variable of its own, and set `VALYSTRUM_SOURCE`, `VALYSTRUM_SECRET: $<that variable>`
and, so that its upload removes what the module no longer lists, `VALYSTRUM_COMPLETE: "true"` on the module's job.

### A project that is renamed, moved or archived

Components are kept per source, and only a complete upload or a person removes them. A project that stops uploading
keeps its components and their open CVE and licence findings. That includes an archived project and one renamed or
moved, whose `CI_PROJECT_PATH` (and with it the default `source`) has changed. Such a source shows as stale under
**SBOM & Licences** after 30 days. On the source's page, **Retire source**, confirmed in a second step, removes its
components and resolves their open CVE and licence findings, noted that the source was retired. Accepted risks and
false positives stay as they are. A member who may upload an SBOM may retire a source, there or with
`POST /api/v1/<org>/sbom/retire` and `{"source": "<the label>"}` from a signed-in session. A later upload under the
same label starts it again.

The scan's findings are kept per project path too: after a rename or move, the new path's scans do not read the
findings recorded under the old one, so close those in **Findings**. Alternatively, keep the old name: leave the key as
it is and set the `repo` and `source` inputs to the path it names.

A project that still exists but rarely changes shows as stale too. Both jobs also run in a scheduled pipeline of the
default branch (**Build, Pipeline schedules**), which keeps its SBOM fresh; the scan then reads the last commit's change
again.

## What is sent, and what never is

The scan sends the project path, the branch (a merge request's source branch), the commit, the merge request's number,
the commit's committer time (`committedAt`, never later than now), the project's default branch (`defaultBranch`), the
pipeline's id (`sequence`, a number that grows with each run), the diff (base64) and, with `send-file-contents`, the
changed files' contents: at most 300 files, each at most 512 KB, about 1.5 MB in all, binary files never. A request
over 2 MB is sent without the files, and only secrets are scanned; one still over 2 MB without them is refused with
HTTP 413.
The SBOM upload sends the CycloneDX SBOM the generators made (with several generators, their component lists merged
into one), from which Valystrum reads each component's name, version, purl and licences, with the source label, the
asset name and `complete`. Each request is signed with HMAC-SHA256 over `"<unix seconds>.<body>"` under your CI
secret, in the header `X-CS-Signature: t=<seconds>,v1=<hex>`; Valystrum accepts a signature once and within five
minutes, so every retry is signed afresh. The job retries on no answer, 408, 429 and 5xx (three attempts for the scan,
four for the SBOM).

Never sent: the CI secret itself, other CI/CD variables, files the change did not touch, the repository's history, or
anything the job's image holds besides the files above. Valystrum does not keep the diff or the files: it scans them
and keeps the scan record (project, branch, merge request, commit, its time, the pipeline's id and the default branch,
the verdict, what was checked and the blast radius, and, when a renamed file's findings could not be moved, its old and
new paths until a later scan moves them), each finding (rule id, `file:line`, the rule's message, which a rule can
write to name the flagged expression, and its remediation hint; a secret only as its masked preview), also as a
Finding in the Findings register with every branch's latest read of its file (the branch, the commit's time or the
arrival, the pipeline's id, and whether the finding was there), a leaked secret in **Data Security, Secrets** as its
masked preview, its location and a one-way fingerprint, and the SBOM's components. All of it is kept for as long as
your organisation exists in Valystrum.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `VALYSTRUM_SECRET is empty` (the job fails whatever `fail-on-error` says) | The pipeline cannot read `CS_WEBHOOK_SECRET`: the name differs, the variable is protected and the branch is not, or the merge request comes from a fork (its pipeline runs in the fork's project, without your variables; run fork pipelines in the parent project if you trust the change, from a fork branch not named like your default branch: see [What it compares, and the branch](#what-it-compares-and-the-branch)) |
| `This is not a git checkout` (the job fails whatever `fail-on-error` says) | The job ran without the repository's working copy, for example with `GIT_STRATEGY: none`: the scan needs the clone to diff the change |
| `No changes to scan.` | The diff is empty: nothing changed under `paths`, or the base was not reachable |
| `Base '...' is not in this clone; falling back to the previous commit` | The base could not be fetched, so only the last commit was scanned. Set the project's Git depth (**Settings, CI/CD, General pipelines**) to 0, or `GIT_DEPTH: 0` on the job |
| `No branch named (VALYSTRUM_BRANCH)` | A pipeline with no branch, such as a tag pipeline your own `rules` added (the component runs only in merge request pipelines and on the default branch): the scan gates the change but closes nothing in Valystrum |
| HTTP 401 `Invalid signature: sign with this repository's CI key` (the scan and the SBOM upload alike) | One answer for each of these, so it tells nobody which organisations exist or which projects have a key. A wrong or rotated secret in `CS_WEBHOOK_SECRET` (see **Rotating and revoking the key** under Setup). The organisation slug in the job's `url` is wrong. Or there is no active key for this project (there is no organisation-wide key to fall back on): step 1 was not run for it, or the key names the project differently from `CI_PROJECT_PATH` (case and groups included), or the project was moved or renamed. Rebind the key: `PATCH /api/v1/<org>/pipeline/config/<keyId>` with `{"repo": "<new path>"}` (the key's id is in `GET /api/v1/<org>/pipeline/config`). The rebind keeps the secret, so `CS_WEBHOOK_SECRET` needs no change; it is made over the API from a signed-in admin session, not in **Settings, CI pipeline keys**. If the new path ever had a key of its own, revoked or not, the rebind fails: issue a key for the new path in step 1 instead, store its secret in `CS_WEBHOOK_SECRET`, and revoke the old path's key. A moved or renamed project then uploads its SBOM under its new path; the components uploaded under the old path stay in Valystrum, with their open findings, until you retire that source (see [A project that is renamed, moved or archived](#a-project-that-is-renamed-moved-or-archived)) |
| HTTP 401 `Signature timestamp outside the five-minute window` | The runner's clock is off by more than five minutes |
| HTTP 401 `Signature already used` | The same signed request reached Valystrum twice. Each attempt, retries included, is signed afresh, so look for a proxy between the runner and Valystrum that replays requests |
| HTTP 403 `FeatureLocked` | The organisation is below the Gold plan |
| SBOM upload: HTTP 403 `Organization is suspended` | The organisation is suspended in Valystrum |
| SBOM upload: HTTP 400 `No valid components found in SBOM payload` | No component in the SBOM has both a name and a version |
| HTTP 413 | The scan's request is over 2 MB even without file contents (narrow it with `paths`), or the SBOM is over 8 MB |
| HTTP 429 | More than 60 scans, or 30 SBOM uploads, a minute for the organisation, counted only once a request's signature checks out; or more than 300 scan requests, or 150 SBOM requests, a minute from one address, signed or not (runners behind one NAT share an address). See **Known issues** below the table |
| Every run warns `could not complete the scan (scan_error)` | The platform's scanner failed after it accepted the request |
| Code analysis reported as `not_configured`, `timeout` or `error` | Only the checks named ran; with `fail-on-error: true` the job fails. A `timeout` is a change set the engines could not finish in their 20 seconds |
| Code analysis and IaC checks reported as `no_files` | No file was analysed: `send-file-contents` is false, every changed file was deleted, binary or over a size cap, or the request went over 2 MB and was sent without the files. Only secrets were checked, and the job does not fail for it, `fail-on-error` included |
| `jq is not installed, and jq 1.8.2 could not be fetched` | The image has no jq and the runner cannot reach github.com (or is not x86_64 or aarch64): use an image with jq. The scan job then passes unless `fail-on-error` is true; the SBOM job fails |
| `python3`, `go` or `mvn`: `command not found` in the SBOM job, or `ensurepip is not available` | `ecosystem: auto` found that ecosystem's manifest, but the image lacks its toolchain: set `ecosystem` or `image` |
| `No supported manifest found` or `0 components` | No manifest in `path` (a Yarn, pnpm, Gradle, .NET, Cargo, Composer or RubyGems project has none the job reads: see [The generators](#the-generators)), or the dependencies were not installed before the job |
| `$'\r': command not found` | The job ran a copy of the script with Windows line endings; use the component as published |

**Known issues.** The scan tries up to three times when there is no answer or the answer is 408, 429 or a 5xx,
waiting up to 120 seconds for each attempt. A retry after a timeout or a 5xx that came after the platform had started
the scan runs it a second time, and you see two scans for one commit. A scan refused with 429 passes unless
`fail-on-error` is true.

## Updating

Releases follow semantic versioning. Move your `@1.0.0` pin to a new release when you choose to; the rules the scan
applies live in Valystrum and change with the platform, not with this project. The script inside each template is
written from one canonical copy in the Valystrum repository, which also serves the Azure Pipelines, AWS CodeBuild and
Google Cloud Build integrations, so a fix lands in all of them in the same change; it reaches your pipelines when you
move your pin to the release that carries it.
