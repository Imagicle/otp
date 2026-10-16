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
these results: the Windows test framework was fixed only enough to validate the
epmd change, and suites such as ssl and ssh do not work on Windows.

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

### Version numbering

Follow the Erlang/OTP convention: `100.0.<n>` for bug and security fixes,
`100.<y>` for features. An application changed in a release takes that
release's version. Procedure in [IMAGICLE.md](IMAGICLE.md#3-prepare-and-tag-a-release).

### Linux tests are the gate

Linux test results are trusted to carry over to Windows, which holds for nearly
all fixes. Windows build and installer must succeed; Windows test jobs are
informational until the Windows test framework is fixed.

### Release-preparation details

- The leading `DEVELOPMENT` line in `make/otp_version_tickets` of `OTP-100.0`
  only makes some tests auto-skip; it has no other effect and may be omitted.
- The argument of `scripts/create-release-tag` names a local backup branch
  created at `HEAD`, used during development to keep work safe across rebases.
  The fork does not need it.
- VEX files are maintained with upstream's `otp-compliance.es`, as described in
  [HOWTO/SBOM.md](https://github.com/erlang/otp/blob/master/HOWTO/SBOM.md#vex).

## Open questions

None.

## Proposed improvements

| # | Improvement | Why | How |
|---|---|---|---|
| 1 | Block releases of untested commits | Tag runs skip all tests and the release job depends only on documentation and the Windows build | Release job verifies through the API that the tagged commit has a successful branch run, or tag runs stop skipping tests |
| 2 | Protect `imagicle-maint-26` and `OTP-*` tags | The procedures are enforced only by convention | Branch protection: pull request, required CI, at least one review. Tag rule: only maintainers create `OTP-*`. Repository admin setting |
| 3 | Make the CI verdict match the gate | Windows test jobs are informational but turn the run red, so a red run says nothing and real Linux failures are easy to miss | Mark Windows test jobs `continue-on-error`, or run them only for applications whose Windows suites work (for example epmd). Fixing the Windows test framework is a larger, separate effort |
| 4 | Make `imagicle-maint-26` the default branch and narrow triggers | The nightly schedule builds the unused `master`; GitHub shows and targets `master` by default; push and pull request each run the full pipeline | Change the default branch; run the schedule weekly to catch runner-image drift; restrict `push` to `imagicle-maint-26` and `OTP-*` tags |
| 5 | Detect new CVEs affecting OTP 26 | The procedures describe how to backport, not how to learn a backport is needed | Enable the scheduled job of `osv-scanner-scheduled` on the fork; named owner subscribed to `erlang/otp` security advisories, with a review cadence |
| 6 | Validate release preparation | Six files are edited by hand and inconsistencies are hard to spot | Readonly CI check: `OTP_VERSION` equals the first line of `otp_versions.table`; `vsn.mk` versions match that line; each changed application has a `notes.xml` section. Drop the backup-branch step from `scripts/create-release-tag` |
| 7 | Check OTP-release gates when Erlang consumers change | The runtime reports OTP release 100 (see Decisions); code compiled or run on it takes branches meant for OTP 27 and later | Include an OTP-release check in the test plan when upgrading RabbitMQ or compiling Erlang code against the fork |
| 8 | Keep this file to what GitHub does not already record | A hand-maintained status file goes stale; releases and tags are already on GitHub | Keep CI baseline, open questions, and decisions; later move them to issues or ADRs |
