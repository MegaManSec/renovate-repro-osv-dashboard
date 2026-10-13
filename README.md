# Renovate reproduction: Dependency Dashboard OSV section

Run Renovate on this repository with the config in `renovate.json`.

1. `fs@0.0.1-security` is flagged by npm as malicious. The dashboard's repository problems list
   shows `WARN: Dependency is currently using a malicious version` without naming the dependency
   or its file.
2. `lodash.pick@4.4.0` is affected by GHSA-p6mc-m468-83gw, which OSV records with
   `last_affected: 4.4.0` and no `fixed` event. No release above 4.4.0 exists. The dashboard
   reports `1/1 CVEs have Renovate fixes` and opens nothing; with the summary set to
   `unresolved` the list is empty.
