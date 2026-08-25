# CI Pipeline Audit

This document provides a comprehensive audit of the `.github/workflows/pipeline.yml` workflow, detailing the intended purpose, identified issues, and required fixes for each job.

---

## 1. `lint`

* **Intended Purpose**: Runs static code analysis (`npm run lint`) to enforce coding standards and catch code style/syntax errors before testing or building.
* **Current Issues**:
  1. Missing execution timeout (`timeout-minutes`). If a command hangs or enters an infinite loop, the runner can consume excessive CI minutes until the default 6-hour limit.
  2. Lacks proper status reporting as the explicit prerequisite for all downstream pipeline jobs.
* **Correct Fix**:
  * Add `timeout-minutes: 10`.
  * Ensure downstream jobs (`unit-tests` and `build`) explicitly declare `needs: [lint]`.

---

## 2. `unit-tests`

* **Intended Purpose**: Executes unit tests (`npm test`) using Jest to verify application logic in isolation.
* **Current Issues**:
  1. Missing `needs: [lint]` dependency declaration. As a result, unit tests execute immediately in parallel with `lint` upon workflow trigger, wasting CI resources if linting fails.
  2. Missing `timeout-minutes` configuration.
* **Correct Fix**:
  * Add `needs: [lint]` with an explanatory comment.
  * Add `timeout-minutes: 15`.

---

## 3. `build`

* **Intended Purpose**: Compiles/bundles application source code into distribution artifacts in the `dist/` directory (`npm run build`).
* **Current Issues**:
  1. Missing `needs: [lint]` dependency declaration. Executes immediately at workflow start instead of waiting for code verification.
  2. Missing artifact upload step (`actions/upload-artifact@v4`). The generated `dist/` directory is lost when the build runner terminates.
  3. Missing `timeout-minutes` setting.
* **Correct Fix**:
  * Add `needs: [lint]` with an explanatory comment.
  * Add `actions/upload-artifact@v4` to store `dist/` under artifact name `app-build`.
  * Add `timeout-minutes: 20`.

---

## 4. `integration-tests`

* **Intended Purpose**: Executes integration tests (`npm run test:integration`) against compiled artifacts (`dist/api.js`).
* **Current Issues**:
  1. Missing `needs: [build]` dependency declaration. Executes in parallel with or prior to build completion.
  2. Missing artifact download step (`actions/download-artifact@v4`). Because GitHub Actions runs each job in an isolated runner environment, `dist/` is missing, causing `src/integration/api.integration.test.js` to fail with `Build output not found`.
  3. Missing `timeout-minutes` setting.
* **Correct Fix**:
  * Add `needs: [build]` with an explanatory comment.
  * Add `actions/download-artifact@v4` to retrieve artifact `app-build` to `dist/` before running tests.
  * Add `timeout-minutes: 30`.

---

## 5. `deploy-staging`

* **Intended Purpose**: Deploys verified build artifacts to the staging environment after all test suites pass, restricted to the `main` branch.
* **Current Issues**:
  1. Missing `needs: [unit-tests, integration-tests]` dependency declaration. Currently runs at pipeline start, incorrectly reporting success without waiting for tests to pass.
  2. Missing branch filter (`if: github.ref == 'refs/heads/main'`). Executes on feature branches and pull requests unnecessarily.
  3. Missing `timeout-minutes` setting.
* **Correct Fix**:
  * Add `needs: [unit-tests, integration-tests]` with an explanatory comment.
  * Add `if: github.ref == 'refs/heads/main'`.
  * Add `timeout-minutes: 15`.

---

## 6. `deploy-production`

* **Intended Purpose**: Deploys application build to production after successful deployment to staging, restricted to the `main` branch.
* **Current Issues**:
  1. Missing `needs: [deploy-staging]` dependency declaration. Runs concurrently with staging deployment and before test verification.
  2. Missing branch filter (`if: github.ref == 'refs/heads/main'`). Triggers on non-main branches.
  3. Missing `timeout-minutes` setting.
* **Correct Fix**:
  * Add `needs: [deploy-staging]` with an explanatory comment.
  * Add `if: github.ref == 'refs/heads/main'`.
  * Add `timeout-minutes: 15`.

---

## 7. `notify`

* **Intended Purpose**: Sends final pipeline notification regardless of pipeline outcome (success, failure, or cancellation).
* **Current Issues**:
  1. Missing `needs:` dependency declaration to position notification after deployment steps.
  2. Missing `if: always()` condition. Defaults to running only when all preceding steps succeed, failing to send alerts when the pipeline fails.
  3. Missing `timeout-minutes` setting.
* **Correct Fix**:
  * Add `needs: [deploy-staging, deploy-production]` (or upstream chain).
  * Add `if: always()`.
  * Add `timeout-minutes: 10`.
