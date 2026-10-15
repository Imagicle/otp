# Imagicle Erlang/OTP 26 fork: status

Snapshot of the fork's state. Procedures are in [IMAGICLE.md](IMAGICLE.md).
Update this file whenever a release, branch change, or answered question
changes it.

Last updated: 2026-10-15.

## Releases

| Tag | Commit | Content | Branch CI run (baseline) |
|---|---|---|---|
| `OTP-100.0` | `4477e6f0` | Backport of CVE-2026-42792 (epmd); `erts` and `common_test` bumped to `100.0` | <https://github.com/Imagicle/otp/actions/runs/38360641651> |

## Branches

- Upstream `maint-26` is frozen at `c9ce879f4c` (after OTP-26.2.5.21), checked
  against `erlang/otp` on 2026-10-15.
- `OTP-100.0` was produced on a feature branch that starts at `c9ce879f4c`
  and adds 26 commits: the CVE fix, the Windows CI work,
  the tagging script, and the release preparation.
- `imagicle-maint-26` still equals `maint-26` and does not yet contain
  `OTP-100.0`. Its CI run
  <https://github.com/Imagicle/otp/actions/runs/38736309757> used the upstream
  workflow: the Windows build failed at "Build wxWidgets" and the application
  test jobs were skipped. Because `maint-26` is the merge base, it can be brought
  up to date with a fast-forward:

  ```bash
  git fetch origin --tags
  git push origin OTP-100.0^{commit}:refs/heads/imagicle-maint-26
  ```

## CI baseline

In the `OTP-100.0` baseline run, all Linux jobs passed. On Windows, these
applications failed: common_test, compiler, dialyzer, diameter, emulator, ftp,
sasl, ssl, stdlib. kernel and ssh were cancelled. The release was accepted with
these results.

## Decisions

### OTP major release 100

Fork releases use OTP major release 100, so they cannot be mistaken for, or
collide with, any upstream release. The build derives the OTP release from the
leading number of `OTP_VERSION`; the `OTP-100.0` Windows installer, extracted
and run on 2026-10-15, reports:

```text
erlang:system_info(otp_release)  "100"
?OTP_RELEASE                     100
system_version                   Erlang/OTP 100 [erts-100.0] ...
erlang:system_info(version)      100.0
crypto:info_lib()                OpenSSL 3.5.9
```

Code that selects features by OTP release therefore sees 100, not 26. The Suite
with RabbitMQ 3.13.7 was installed and tested on `OTP-100.0` without issues.

## Open questions

| Question | Context |
|---|---|
| Version numbering after `OTP-100.0` | Proposal: OTP patch releases `100.0.<n>` (`100.0.1`, `100.0.2`, ...), following OTP's convention that patch releases append to the last version. An application changed in a release takes that release's version (`erts-100.0.1` in `OTP-100.0.1`); unchanged applications keep their current version |
| Whether `DEVELOPMENT` belongs in `make/otp_version_tickets` of a release | `OTP-100.0` has it as the first line, followed by the tickets |
| Exact commands used to add the CVE-2026-42792 entry to `openvex.table` and regenerate `otp-26.openvex.json` | `otp-26.openvex.json` is attached to every release and declares `OTP-100.0` / `erts@100.0` as `fixed`. Upstream generates it from `openvex.table` with `.github/scripts/otp-compliance.es` and `vexctl` (see `HOWTO/SBOM.md`); both exist only on upstream `master`, not on `maint-26` |
| Which branch `scripts/create-release-tag <branch>` is meant to move | After creating the tag, the script runs `git branch -f <branch>`. Run from `imagicle-maint-26` with that same name, the tag is created and the script then fails; `maint-26` was not moved by `OTP-100.0` |
| Which Windows test failures are accepted, and whether to encode them in `_gh.spec` files | Would make the Windows test jobs a reliable gate |

## Proposed improvements

| # | Improvement | Why | How |
|---|---|---|---|
| 1 | Block releases of untested commits | Tag runs skip all tests and the release job depends only on documentation and the Windows build | Release job verifies through the API that the tagged commit has a successful branch run, or tag runs stop skipping tests |
| 2 | Protect `imagicle-maint-26` and `OTP-*` tags | The procedures are enforced only by convention | Branch protection: pull request, required CI, at least one review. Tag rule: only maintainers create `OTP-*`. Repository admin setting |
| 3 | Make Windows tests a real gate | Manual comparison with the baseline erodes over time | Classify known Windows failures, encode accepted ones as skips with a reason in `*_gh.spec`, allow one rerun of failed jobs |
| 4 | Make `imagicle-maint-26` the default branch and narrow triggers | The nightly schedule builds the unused `master`; GitHub shows and targets `master` by default; push and pull request each run the full pipeline | Change the default branch; run the schedule weekly to catch runner-image drift; restrict `push` to `imagicle-maint-26` and `OTP-*` tags |
| 5 | Detect new CVEs affecting OTP 26 | The procedures describe how to backport, not how to learn a backport is needed | Enable the scheduled job of `osv-scanner-scheduled` on the fork; named owner subscribed to `erlang/otp` security advisories, with a review cadence |
| 6 | Validate release preparation | Six files are edited by hand and inconsistencies are hard to spot | Readonly CI check: `OTP_VERSION` equals the first line of `otp_versions.table`; `vsn.mk` versions match that line; each changed application has a `notes.xml` section |
| 7 | Check OTP-release gates when Erlang consumers change | The runtime reports OTP release 100 (see Decisions); code compiled or run on it takes branches meant for OTP 27 and later | Include an OTP-release check in the test plan when upgrading RabbitMQ or compiling Erlang code against the fork |
| 8 | Keep this file to what GitHub does not already record | A hand-maintained status file goes stale; releases and tags are already on GitHub | Keep CI baseline, open questions, and decisions; later move them to issues or ADRs |
