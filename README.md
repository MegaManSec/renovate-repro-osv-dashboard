# Renovate reproduction: Dependency Dashboard OSV section

Run Renovate on this repository with the config in `renovate.json`.

1. `fs@0.0.1-security` is flagged by npm as malicious. The dashboard's repository problems list
   shows `WARN: Dependency is currently using a malicious version` without naming the dependency
   or its file.
2. `lodash.pick@4.4.0` is affected by GHSA-p6mc-m468-83gw, which OSV records with
   `last_affected: 4.4.0` and no `fixed` event. No release above 4.4.0 exists. The dashboard
   reports `1/1 CVEs have Renovate fixes` and opens nothing; with the summary set to
   `unresolved` the list is empty.

## Cases under `cases/`

These cases share the root `renovate.json`. The settings added for them do not change the two root
findings above: the `ignorePaths` list repeats the `config:recommended` defaults, the `enabled: false` rule
only matches `cases/**`, and the root package file has no fixable advisory.

Run the **Generate pnpm lockfiles** workflow (Actions tab, `workflow_dispatch`) once before running
Renovate. It writes `pnpm-lock.yaml` for cases 1 and 2 with `resolution-mode=lowest-direct`, so each
importer locks the lowest version its range allows (vitest 3.2.4 and 4.1.4). Renovate only uses
`update-lockfile` for vulnerability alerts when a locked version exists, so these cases need the
lockfile.

All three cases use OSV data (`osvVulnerabilityAlerts`). Each case gets its own branch through
`vulnerabilityAlerts.groupName: "{{{parentDir}}} security"`.

The "current behavior" below is what Renovate 44.117.2 did in a private monorepo with the same
shape, and what our reading of the code on `main` predicts. It has not yet been run on this
repository.

### Case 1: `cases/1-vulnerability-range-floor`

Discussion: (link to be added)

The workspace root declares `vitest: ^3.2.4` (locked 3.2.4). vitest 3.2.4 is affected by
GHSA-5xrq-8626-4rwp (fixed in 3.2.6 and 4.1.0) and GHSA-82fw-gwwq-j7x9 (fixed in 4.1.11), so the fix
target is 4.1.11. `other/` declares `vitest: 4.1.4` and is excluded with `ignorePaths`; it stands
in for another importer that already has 4.1.4 in the lockfile.

Current behavior: the vulnerability PR changes `^3.2.4` to `^4.0.0`. That range admits 4.0.0 to
4.1.10, all affected by GHSA-82fw-gwwq-j7x9, and pnpm can keep the already locked 4.1.4.

Expected behavior: the new range excludes vulnerable versions, for example `^4.1.11`, which is
what `vulnerabilityAlerts.rangeStrategy: "bump"` produces.

### Case 2: `cases/2-lockfile-update-other-importers`

Discussion: (link to be added)

The workspace root declares `vitest: ^4.1.4` (locked 4.1.4), affected by GHSA-82fw-gwwq-j7x9 with
the fix 4.1.11 inside its range. `legacy/` declares `vitest: ^3.2.4` (locked 3.2.4) and is excluded
with `ignorePaths`; it stands in for an importer whose own update is handled in another branch.

Current behavior: Renovate runs `pnpm update --no-save vitest@4.1.11 --lockfile-only --recursive`.
The lockfile then records `legacy` with `specifier: ^3.2.4` and `version: 4.1.11`.

Expected behavior: only importers whose range admits 4.1.11 change; `legacy` keeps 3.2.4.

### Case 3: `cases/3-held-group-member`

Discussions: (links to be added, one for the dashboard and one for the fix target)

`next` 15.5.18 is affected by 12 advisories. Two CRITICAL ones (GHSA-2xp9-vwfh-vxw4,
GHSA-p293-qw3h-jr36) are fixed in 15.5.24 (2026-08-25); the newest two (GHSA-4jqv-mc3x-m676,
GHSA-mcj8-r9mp-w47p, MODERATE) are fixed in 15.5.27 (2026-09-30). `lodash` 4.17.20 has fixes
released long ago (4.18.0, 2026-03-31). Both are in the same group, and
`vulnerabilityAlerts.minimumReleaseAge` is 45 days.

The case needs the age of 15.5.27 to be below `minimumReleaseAge` and the age of 15.5.24 to be at
or above it. With 45 days this holds from 2026-10-13 to 2026-11-13; later, raise the value to more
than the age of 15.5.27.

Current behavior: Renovate selects 15.5.27 for `next`, marks it `pendingChecks`, and drops it from
the group branch, which is created with `lodash` only. The dashboard does not list `next` under
"Pending Status Checks". No update to 15.5.24 is offered, although it fixes both CRITICAL
advisories and passes the age check.

Expected behavior: the held `next` update is shown on the dashboard with the release it waits
for. For the fix target, see the linked idea discussion.
