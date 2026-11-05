# CI pipeline: answers and reflection

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

Evidence: [integration error](task-1.png), [failed graph with packaging skipped](task-1-graph.png), and [corrected green pipeline](task-1-fixed-green.png).

## Task 2: matrix and caching

The frontend builds on Node 20 and Node 22. Each matrix entry uploads a different artifact: `frontend-dist-node20` or `frontend-dist-node22`. `fail-fast: false` allows the other matrix entry to finish when one fails; the overall matrix still fails if either entry fails. The [matrix screenshot](task-2.png) shows Node 20 failing while Node 22 succeeds.

The Maven cache key contains the action prefix, runner OS, architecture, Maven identifier, and a hash of dependency configuration. With the defaults used here, setup-java v6 hashes `pom.xml` files and supported Maven wrapper/extension configuration. The npm key contains the action prefix, OS, architecture, npm identifier, and the hash of `frontend/package-lock.json` (or `e2e/package-lock.json` in E2E). The Node version is not part of this npm key; the action caches package downloads, not `node_modules`.

If dependency changes never changed the key, runs would keep restoring an outdated cache. Missing packages would need downloading repeatedly, while immutable cache entries would not be refreshed under that same key. Maven and `npm ci` still resolve/install the declared dependencies; an outdated download cache alone does not imply that the lockfile is ignored.

Sources: [setup-java cache implementation](https://github.com/actions/setup-java/blob/v6/src/cache.ts), [setup-node cache implementation](https://github.com/actions/setup-node/blob/v4/src/cache-restore.ts).

### Cache measurements

Both attempts use [CI run 41990661408](https://github.com/mikhail2574/student-news/actions/runs/41990661408), on the same commit: attempt 1 has cache misses and attempt 2 has exact-key hits. The cold cache came from a new PR cache scope.

| Dependency step | Cold | Warm | Saving |
| --- | --- | --- | --- |
| Maven `dependency:go-offline`, full step | 19.765 s | 11.843 s | 7.922 s |
| Node 22 `npm ci`, full step | 2.829 s | 2.152 s | 0.677 s |

Full-step times are approximated from log timestamps and include command startup. Maven's internal timer reports 15.327 s versus 5.817 s, saving 9.510 s. GitHub rounds the visible step labels to 20/12 s for Maven and 2/2 s for npm.

Evidence: [Maven cold](task-2-maven-cold.png), [Maven warm](task-2-maven-warm.png), [npm cold](task-2-npm-cold.png), [npm warm](task-2-npm-warm.png).

## Task 3: selective E2E runs

Both `push` and `pull_request` triggers in `e2e.yml` ignore `**.md` and `docs/**`. If every changed file matches an ignored path, E2E is not triggered. CI retains its existing triggers.

Skipping would be wrong if documentation files became executable inputs, such as configuration snippets consumed by a build or test. Keep exclusions limited to inert documentation, review them when file usage changes, and add a scheduled or manually triggered full E2E run as a backstop. The current workflow does not yet implement that additional backstop.

[PR #4](https://github.com/mikhail2574/student-news/pull/4) changed only README.md: [CI ran and E2E did not](task-3-readme-only.png). [PR #5](https://github.com/mikhail2574/student-news/pull/5) changed a Java comment under backend/: [both workflows ran](task-3-backend.png). These demonstrations exercise pull-request events; both workflow triggers contain the path filters.

## Task 4: branch protection

`CI status` provides one stable required check for branch protection. Its dependencies and result inspection live with the workflow, so adding a job does not require manually updating a long list of required checks in repository settings. The aggregate job must include every mandatory stage and must reject non-success results.

The rule for `main` requires `CI status`. The [non-draft PR screenshot](task-2.png) shows the failed required check and disabled merge button.

## Reflection (200 words)

This pipeline provides Continuous Integration: changes trigger linting, unit tests, database integration tests, frontend builds, and container packaging. It lacks Continuous Delivery because deploy.yml only prints actions and the packaged images are not published as deployable releases. Continuous Deployment requires more because production deployment is neither implemented nor automatically triggered after successful checks.

To reach Continuous Delivery, packaging must publish immutable backend and frontend images to a registry. That introduces credential exposure, registry availability, and artifact provenance risks. The deployment gate must verify successful CI for the exact commit associated with those images; an incorrect association could release untested code.

The deployment job must authenticate to a real environment and update its workloads. Excessive permissions or an incorrect environment selection could damage production. Database migration handling must become explicit because schema changes can make rollback unsafe or lose data.

Readiness checks, rollout monitoring, and smoke tests must replace printed placeholders. Weak checks could direct traffic to an unhealthy release. Automatic rollback needs procedures and compatible application and database versions.

With these capabilities and a manual production approval, the project could offer Continuous Delivery. Removing that approval and automatically releasing every change would establish Continuous Deployment, increasing exposure to undetected defects.
