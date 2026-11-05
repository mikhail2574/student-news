# CI pipeline: answers, reflection, and evidence

## Task 1: integration test stage

`integration-test` runs after `frontend-lint` and `backend-test`, in parallel with any remaining frontend builds, and before `package`. Cheap failures therefore stop the database job before it starts. Running integration tests later would waste time packaging broken code; running them earlier would start PostgreSQL even when lint or unit tests already fail.

The integration job uses the same checkout, Temurin JDK 21, and Maven cache as the unit-test job. Its command is:

```sh
mvn -B test -Dtest='*IntegrationTest,*E2ETest'
```

Packaging is gated by this line in `.github/workflows/ci.yml`:

```yaml
needs: [frontend-lint, backend-test, frontend-build, integration-test]
```

With the default success condition, a failed dependency prevents `package` from running. `ci-status` uses `if: always()` to inspect all required results and fails on failure, cancellation, or skipped jobs.

Testcontainers starts PostgreSQL from the Java tests using the runner's Docker daemon. In `AbstractPostgreSQLIntegrationTest`, `@Container` manages its lifecycle and `@DynamicPropertySource` supplies the generated JDBC URL and credentials to Spring. A workflow `services:` block instead starts a container before test execution, with connection settings supplied through workflow configuration. GitHub-hosted Ubuntu runners provide Docker, so these tests need no additional PostgreSQL service.

## Task 2: matrix and caching

The frontend builds on Node 20 and Node 22. Each matrix entry uploads a different artifact: `frontend-dist-node20` or `frontend-dist-node22`. `fail-fast: false` allows the other matrix entry to finish when one fails; the overall matrix still fails if either entry fails.

The Maven cache key contains the action prefix, runner OS, architecture, Maven identifier, and a hash of dependency configuration. With the defaults used here, setup-java v6 hashes `pom.xml` files and supported Maven wrapper/extension configuration. The npm key contains the action prefix, OS, architecture, npm identifier, and the hash of `frontend/package-lock.json` (or `e2e/package-lock.json` in E2E). The Node version is not part of this npm key; the action caches package downloads, not `node_modules`.

If dependency changes never changed the key, runs would keep restoring an outdated cache. Missing packages would need downloading repeatedly, while immutable cache entries would not be refreshed under that same key. Maven and `npm ci` still resolve/install the declared dependencies; an outdated download cache alone does not imply that the lockfile is ignored.

Sources: [setup-java cache implementation](https://github.com/actions/setup-java/blob/v6/src/cache.ts), [setup-node cache implementation](https://github.com/actions/setup-node/blob/v4/src/cache-restore.ts).

### Cache measurements

Compare the same workflow, commit, job, and step across attempts. Confirm a cache miss/hit in the setup step logs. Report cache restoration separately from dependency installation: setup-java restores Maven's cache, but the Maven test step also compiles and tests. Its total time is not a pure dependency-download measurement.

| Measurement | Cold attempt | Warm attempt | Saving |
| --- | --- | --- | --- |
| Maven dependency resolution (dedicated step required for a pure measurement) | Pending | Pending | Pending |
| Frontend Node 22: Install dependencies (`npm ci`) | Pending | Pending | Pending |

Saving in seconds = cold duration minus warm duration. Do not substitute durations from different workflows or Node versions.

## Task 3: selective E2E runs

Both `push` and `pull_request` triggers in `e2e.yml` ignore `**.md` and `docs/**`. If every changed file matches an ignored path, E2E is not triggered. CI retains its existing triggers.

Skipping would be wrong if documentation files became executable inputs, such as configuration snippets consumed by a build or test. Keep exclusions limited to inert documentation, review them when file usage changes, and add a scheduled or manually triggered full E2E run as a backstop. The current workflow does not yet implement that additional backstop.

For the demonstration, use separate PRs based on a branch that already contains the workflow changes. A README-only commit added to a PR that also changes workflows is insufficient: pull-request path filtering considers the PR's complete diff. Feature branches do not trigger the current push workflow, which only targets main/master; PRs targeting main can demonstrate both filters without pushing demonstration changes to main.

## Task 4: branch protection

`CI status` provides one stable required check for branch protection. Its dependencies and result inspection live with the workflow, so adding a job does not require manually updating a long list of required checks in repository settings. The aggregate job must include every mandatory stage and must reject non-success results.

Configure an active rule targeting `main`, require status checks before merging, and select the exact check name `CI status`. A disabled merge button on a draft PR is not evidence of status-check enforcement: use a non-draft PR with failing CI and show the required-check explanation.

## Reflection (200 words)

This pipeline provides Continuous Integration: changes trigger linting, unit tests, database integration tests, frontend builds, and container packaging. It lacks Continuous Delivery because deploy.yml only prints actions and the packaged images are not published as deployable releases. Continuous Deployment requires more because production deployment is neither implemented nor automatically triggered after successful checks.

To reach Continuous Delivery, packaging must publish immutable backend and frontend images to a registry. That introduces credential exposure, registry availability, and artifact provenance risks. The deployment gate must verify successful CI for the exact commit associated with those images; an incorrect association could release untested code.

The deployment job must authenticate to a real environment and update its workloads. Excessive permissions or an incorrect environment selection could damage production. Database migration handling must become explicit because schema changes can make rollback unsafe or lose data.

Readiness checks, rollout monitoring, and smoke tests must replace printed placeholders. Weak checks could direct traffic to an unhealthy release. Automatic rollback needs procedures and compatible application and database versions.

With these capabilities and a manual production approval, the project could offer Continuous Delivery. Removing that approval and automatically releasing every change would establish Continuous Deployment, increasing exposure to undetected defects.

## Evidence checklist

- [x] Baseline CI passed: [run 41988188831](https://github.com/mikhail2574/student-news/actions/runs/41988188831).
- [x] Baseline E2E passed: [run 41988188726](https://github.com/mikhail2574/student-news/actions/runs/41988188726).
- [ ] Integration failure log with the database-layer error, while unit tests pass.
- [ ] Failed run graph showing skipped packaging, with the `needs` explanation above.
- [ ] Node 20 fails deliberately while Node 22 completes successfully.
- [ ] Protected, non-draft PR shows merging blocked by required `CI status`.
- [ ] Corrected demonstration PR has a green pipeline.
- [ ] Cold and warm cache logs, matching step durations, and calculated savings.
- [ ] README-only PR: CI runs and E2E does not.
- [ ] Backend-change PR: both CI and E2E run.

Screenshots and unfinished measurements must be added from actual GitHub runs; this checklist does not claim they have already been collected.
