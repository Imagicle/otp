# Maintaining the Imagicle Erlang/OTP 26 fork

Imagicle ships Erlang/OTP 26, which upstream no longer maintains. This fork
(`Imagicle/otp`) carries security backports on top of the last upstream OTP 26
maintenance branch and publishes its own `OTP-<vsn>` releases. This document
describes how to backport a fix, sync upstream, and cut a release, and how CI
behaves while doing so.

Current state of branches, releases, CI baselines, and open questions is in
[IMAGICLE-STATUS.md](IMAGICLE-STATUS.md).

## Branch model

| Branch / tag | Role | Who writes to it |
|---|---|---|
| `maint-26` | Mirror of `erlang/otp` `maint-26`. Never commit to it. | Fast-forward from upstream only |
| `imagicle-maint-26` | Imagicle release line. Every fork release tag points to a commit on it. | Merges of reviewed topic branches and upstream syncs |
| `imagicle/<topic>` | One backport, CI change, or release preparation | The author, merged through a pull request |
| `OTP-<vsn>` | Annotated release tag; pushing it triggers the release job | Release manager |
| `master` and other `maint-*` | Upstream mirrors, unused by Imagicle | Nobody |

### Merge, do not rebase

`imagicle-maint-26` and its tags are published, and released binaries reference
their commits. Integrate upstream changes and topic branches with merges.
Rebase only unpublished topic branches.

## CI/CD

All build, test, and release work is in `.github/workflows/main.yaml`
("Build and check Erlang/OTP"). The fork's workflow differs from upstream in
that it:

- tests every application on every run, not only the changed ones
  (upstream does this only for labelled PRs and scheduled runs);
- runs the application and system test suites on Windows, not only on Linux;
- builds Windows against a pinned, hash-verified OpenSSL linked statically, and
  fails if `crypto.dll` links OpenSSL dynamically or if the compile-time and
  runtime OpenSSL versions differ;
- finds Visual Studio through `vswhere` instead of a hard-coded path;
- creates releases on the fork (upstream allows them only on `erlang/otp`)
  and attaches the Windows installer and OpenVEX file.

### Triggers

| Event | Effect |
|---|---|
| Push to any branch | Full build plus Linux and Windows test matrices |
| Pull request | Same as push |
| Push of an `OTP-*` tag | Builds and the release job; **all test jobs are skipped** |
| Nightly schedule (01:00 UTC) | Runs on the default branch `master` only |

The other workflows (`sync-github-prs`, `sync-github-releases`, `update-base`,
`pr-comment`) are guarded by `github.repository == 'erlang/otp'` and do nothing
on the fork. `actions-updater` runs weekly and `osv-scanner-scheduled` runs on
push and pull request.

### Jobs

| Stage | Jobs |
|---|---|
| Source and cache | `Build Erlang/OTP (64-bit)` produces the source archive and prebuilt cache used by later jobs |
| Builds | 32-bit, clang, cross-compile, types and flavors, macOS, iOS, Windows (installer) |
| Checks | Documentation build with `xmllint` and HTML link check; static analysis (`make format-check`, dialyzer) |
| Tests | `Test Erlang/OTP (<app>)` per application on Linux; `Test Erlang/OTP (Windows) (<app>)` per application; system tests on both platforms |
| Release | `Release Erlang/OTP`, tags only, depends on documentation and the Windows build |

Test skips go in the `<app>_gh.spec` files
(for example `lib/ssl/test/ssl_gh.spec`, `lib/kernel/test/kernel_gh.spec`);
both platforms use them.

### Reading results

The Linux test jobs are the gate: they must be green. Linux results carry over
to Windows for nearly all fixes.

On Windows, the build and installer jobs must succeed. Windows test jobs are
informational: the Windows test framework in OTP 26 is only partly fixed, and
some suites (for example ssl and ssh) do not work at all, without the
application being broken. Only the Windows tests of the application a change
touches deserve a look, compared with the same application in the baseline run
listed in [IMAGICLE-STATUS.md](IMAGICLE-STATUS.md).

Because tag pushes skip tests, the only test evidence for a release is the
branch run of the tagged commit. Never tag a commit without a reviewed branch
run. After a release, record its branch run as the new baseline.

## Procedures

### 1. Sync upstream `maint-26`

Needed only if upstream publishes a further OTP 26 patch.

```bash
git remote add upstream https://github.com/erlang/otp.git   # once
git fetch upstream maint-26 --tags
git push origin upstream/maint-26:refs/heads/maint-26       # must fast-forward
git switch -c imagicle/sync-upstream-<date> origin/imagicle-maint-26
git merge --no-ff origin/maint-26
```

Expect conflicts in the version files listed in procedure 3 and in
`.github/workflows/main.yaml`. Keep the fork's workflow changes. Open a pull
request into `imagicle-maint-26` and review the CI run.

### 2. Backport a security fix

1. Identify the fixing commits in `erlang/otp`, usually on the oldest maintained
   `maint-*` branch, and the OTP ticket number (`OTP-NNNNN`) from the release
   notes.
2. Branch from `imagicle-maint-26`: `git switch -c imagicle/<cve-id> origin/imagicle-maint-26`.
3. Cherry-pick with provenance: `git cherry-pick -x <sha>...`. Adapt the code to
   OTP 26 where it differs, and keep or port the upstream test.
4. Push, open a pull request into `imagicle-maint-26`, and review CI as described
   in Reading results.
5. Merge, then release through procedure 3.

### 3. Prepare and tag a release

Choose the version following the Erlang/OTP convention:

| Release content | OTP version | Example |
|---|---|---|
| Bug or security fixes | `100.0.<n>` | `100.0.1`, `100.0.2` |
| New features | `100.<y>` | `100.1` |

An application changed in a release takes that release's version (for example
`erts-100.0.1` in `OTP-100.0.1`); unchanged applications keep their version.

Follow the pattern of commit `4477e6f0 Prepare release`, in one commit on a
topic branch:

| File | Change |
|---|---|
| `OTP_VERSION` | New OTP version |
| `<app path>/vsn.mk` | Version of each changed application (for example `erts/vsn.mk`, `lib/common_test/vsn.mk`) |
| `<app path>/doc/src/notes.xml` | Release-notes section for each changed application, citing the OTP ticket and CVE |
| `otp_versions.table` | New first line: `OTP-<vsn> : <changed app-vsn ...> # <unchanged app-vsn ...> :` |
| `make/otp_version_tickets` | Tickets included in the release. A leading `DEVELOPMENT` line only makes some tests auto-skip and may be omitted |
| `openvex.table`, `otp-26.openvex.json` | VEX statements for the CVEs fixed in the release (see below) |

#### Update the VEX files

`openvex.table` is the append-only source; `otp-26.openvex.json` is generated
from it with `vexctl` commands produced by upstream's
`.github/scripts/otp-compliance.es`. The format and use cases are described in
[HOWTO/SBOM.md](https://github.com/erlang/otp/blob/master/HOWTO/SBOM.md#vex).

Constraints for the fork:

- The script exists only on upstream `master`. Extract it to a temporary
  directory; do not copy it into the working tree, where it would overwrite the
  fork's `.github/scripts`.
- It needs Erlang/OTP 27 or later to run (it uses `~"..."` sigils), plus
  [`vexctl`](https://github.com/openvex/vexctl).
- It reads `otp_versions.table` from the current directory, so run it from the
  repository root after the new first line has been added.

Append the statement under `"otp-26"` in `openvex.table`, for example:

```json
{
  "pkg:otp/erts@14.0": "CVE-YYYY-NNNNN",
  "status": {
    "affected": "Update to any of the following versions: pkg:otp/erts@100.0.1",
    "fixed": ["pkg:otp/erts@100.0.1"]
  }
}
```

Then generate the statements:

```bash
tmp=$(mktemp -d)
git fetch upstream master
git archive upstream/master .github/scripts | tar -x -C "$tmp"
escript "$tmp/.github/scripts/otp-compliance.es" vex run --input-file openvex.table -b otp-26          # review the vexctl commands
escript "$tmp/.github/scripts/otp-compliance.es" vex run --input-file openvex.table -b otp-26 | bash
```

#### Tag

After the pull request is merged and its branch run has been reviewed:

```bash
git switch imagicle-maint-26 && git pull --ff-only
scripts/create-release-tag debug                  # prints the tag annotation without tagging
scripts/create-release-tag backup/OTP-<vsn>
git push origin OTP-<vsn>
```

`create-release-tag` builds the annotated tag from the first line of
`otp_versions.table`, then creates a local backup branch named by its argument
at `HEAD` (`git branch -f <name>`). The backup branch is not needed by the
fork; delete it with `git branch -D backup/OTP-<vsn>`. The argument must not be
the checked-out branch.

Pushing the tag runs the release job, which publishes GitHub Release
`OTP <vsn>` with `otp_src_<vsn>.tar.gz`, `otp_doc_html_<vsn>.tar.gz`,
`otp_doc_man_<vsn>.tar.gz`, `otp_win64_<vsn>.exe`, `otp-26.openvex.json`,
`MD5.txt` and `SHA256.txt`. Check that all seven assets are present.

## Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `Build Erlang/OTP (Windows)` fails at "Build wxWidgets" | The branch does not contain the fork's workflow and uses upstream's hard-coded Visual Studio 2022 Enterprise path | Branch from `imagicle-maint-26` |
| All `Test Erlang/OTP` jobs skipped on a branch push | The branch runs the upstream workflow, which tests only changed applications | Branch from `imagicle-maint-26` |
| Tests skipped on a tag run | By design | Review the branch run of the tagged commit |
| Windows test application red | Incomplete Windows test framework or runner instability | Informational; check only the application the change touches (see Reading results) |
| Windows build fails on OpenSSL version or hash | `OPENSSL_VERSION` / `OPENSSL_SHA256` no longer match the published installer | Update both values in `main.yaml` from <https://github.com/slproweb/opensslhashes> |
