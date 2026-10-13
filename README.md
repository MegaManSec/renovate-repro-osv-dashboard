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
Renovate. It writes `pnpm-lock.yaml` for cases 1, 2 and 5 with `resolution-mode=lowest-direct`, so each
importer locks the lowest version its range allows (vitest 3.2.4 and 4.1.4, js-yaml 4.1.0). Renovate only uses
`update-lockfile` when a locked version exists, so these cases need the lockfile.

Cases 1 to 3 use OSV data (`osvVulnerabilityAlerts`). Each case gets its own branch through
`vulnerabilityAlerts.groupName: "{{{parentDir}}} security"`.

The "current behavior" below is what Renovate 44.117.2 did in a private monorepo with the same shape. The Mend-hosted app (Renovate 44.155.2) first ran on this repository on 2026-10-13, after the lockfile workflow committed the lockfiles, and produced the [Dependency Dashboard](https://github.com/MegaManSec/renovate-repro-osv-dashboard/issues/8), which has no "Pending Status Checks" section. Case 1: [#6](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/6) changes `^3.2.4` to `^4.0.0` and locks the root importer at `4.1.4(vite@7.3.7)`; [#5](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/5), on the case's group branch, only updates `packageManager`. Case 2: [#1](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/1) locks `legacy` (`specifier: ^3.2.4`) at `4.1.11(vite@7.3.7)`. Case 3: [#3](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/3) updates `lodash` only, and the dashboard shows `next 15.5.18` → `[Updates: 15.5.27]` under Detected Dependencies without saying it is held. Case 4: `cases/4-skipped-only-package-file/package.json` is an empty block under Detected Dependencies. Case 5: the row reads `js-yaml ^4.1.0` → `[Updates: ^4.1.0]`, while [#2](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/2) locks js-yaml 4.3.2. Case 6: [#4](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/4) links `https://renovatebot.com/diffs/npm/vitest@>2.1.9 <=3.1.2>vite/6.2.5/6.4.3`. [#7](https://github.com/MegaManSec/renovate-repro-osv-dashboard/pull/7) (vitest v4 for case 6) and the rate-limited `ubuntu` update are side effects, not cases.

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

### Case 4: `cases/4-skipped-only-package-file`

PR: https://github.com/renovatebot/renovate/pull/47005

`package.json` has one dependency, `"sibling": "file:./sibling"`, which npm extraction marks with
`skipReason: file`. `sibling/` only exists so that the path resolves, and is excluded with
`ignorePaths`.

Current behavior: Detected dependencies hides the skipped dependency but still lists the package
file. Under `npm`, `cases/4-skipped-only-package-file/package.json` is an empty `<details>` block,
and it is counted in `npm (N)`.

Expected behavior: the package file is not listed, and `npm (N)` equals the number of package files
shown under it.

### Case 5: `cases/5-lockfile-only-update`

PR: https://github.com/renovatebot/renovate/pull/47006

`package.json` declares `js-yaml: ^4.1.0`, locked at 4.1.0 by the lockfile workflow. The Detected
dependencies row only shows updates when one exists, so a package rule enables routine updates for
this dependency, with `rangeStrategy: "update-lockfile"`. The update keeps `^4.1.0` as its
`newValue` and only changes the lockfile. js-yaml 4.1.0 is also affected by OSV advisories (the
newest fix is 4.3.2), so the update may come from the vulnerability alert instead; its default
`rangeStrategy` is also `update-lockfile`, and the row is the same.

Current behavior: the row reads `js-yaml ^4.1.0 → [Updates: ^4.1.0]`, the range that is already
there.

Expected behavior: the row shows the new locked version, for example
`js-yaml ^4.1.0 → [Updates: lock file @ 4.3.2]`.

### Case 6: `cases/6-pnpm-override-selector`

PR: https://github.com/renovatebot/renovate/pull/47007

`package.json` has the `pnpm.overrides` selector `vitest@>2.1.9 <=3.1.2>vite: 6.2.5`. Extraction
keeps the selector as `depName` and sets `packageName: vite` for the lookup. No lockfile is needed:
the Detected dependencies list shows the selector as the dependency name. vite 6.2.5 is affected by
OSV advisories, so the case's security group should also get a PR that updates the override.

Current behavior: in any PR that updates the override, the Change column of the PR body links to
`https://renovatebot.com/diffs/npm/vitest@>2.1.9 <=3.1.2>vite/6.2.5/<new>`, which contains the whole
selector (including spaces) and does not resolve.

Expected behavior: the link uses the looked up package,
`https://renovatebot.com/diffs/npm/vite/6.2.5/<new>`. The selector stays the dependency name in the
dashboard, branch names and `matchDepNames`.
