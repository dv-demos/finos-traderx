# FINOS TraderX — Develocity + Provenance Governor demo

This repository is a CI wrapper around [FINOS TraderX](https://github.com/finos/traderX). It
contains no application source. Everything it builds comes from a SHA-pinned, **unmodified**
checkout of the upstream project, cloned at workflow run time.

The point of the demo is to show that a Develocity Build Scan can be tied to a specific published
container image, and that a policy gate can then be evaluated against that image — all against a
project you do not own and cannot change.

## What it does

For each Java service, one matrix leg:

1. Clones `finos/traderX` at a pinned SHA.
2. Builds it with its own Gradle wrapper, with the Develocity plugin **injected at runtime**
   (the upstream build declares no Develocity plugin, and we are not allowed to add one).
3. Tags the resulting Build Scan with CI provenance values via a Gradle init script applied with
   `-I` from *this* repository.
4. Builds the upstream `Dockerfile` and pushes the image to Artifactory.
5. Publishes a Develocity Provenance Governor attestation that links the image digest to the Build
   Scan from step 3.
6. Runs the `publish-artifact` policy scan against the published image.

| Project (under `templates/`) | Type | Image published |
|---|---|---|
| `account-service-specfirst`  | Spring Boot 3.5.16 | yes |
| `position-service-specfirst` | Spring Boot 3.5.16 | yes |
| `trade-processor-specfirst`  | Spring Boot 3.5.16 | yes |
| `trade-service-specfirst`    | Spring Boot 3.5.16 | yes |
| `database-specfirst`         | `application` plugin, H2 fat jar | no — no Dockerfile upstream |

## Pinned upstream

```
https://github.com/finos/traderX.git
a8c9b46c0c9e8f3c688ab41ea77648bcb8c32170   # "Fix prepublish dependency-check args", 2026-07-29
```

Change `TRADERX_SHA` in `.github/workflows/deploy.yaml` to move the pin. Nothing else in this
repository hard-codes upstream state, with the exception of the matrix project list.

## Files

| File | Purpose |
|---|---|
| `.github/workflows/deploy.yaml` | The whole pipeline. |
| `gradle/develocity-scan-tagging.init.gradle` | Build Scan custom values, applied with `-I`. |

## How scan-to-image correlation works

The reference demo
([`dv-demos/apps-java-payment-calculator`](https://github.com/dv-demos/apps-java-payment-calculator/blob/main/.github/workflows/deploy.yaml))
tags its scans from a `.mvn/develocity-custom-user-data.groovy` file committed inside the project
being built. That option does not exist here — the project being built is someone else's repository
at a fixed commit.

Instead, `gradle/develocity-scan-tagging.init.gradle` lives in this repository and is passed to each
build with `-I`. It attaches, among other values:

```
CI build = <run-id>-<run-attempt>/<gradle-root-project-name>
```

TraderX is five independent Gradle builds published to the same Develocity instance from the same
workflow run, so the run id alone does not identify a single scan — hence the project name in the
value. The attestation step then finds exactly one scan per service with:

```
build-scan-queries: 'value:"CI build=${{ env.CI_ID }}/${{ matrix.project }}"'
```

The init script registers itself through `pluginManager.withPlugin(...)` from a `beforeSettings`
hook, so it does not care whether it runs before or after the plugin injection performed by
`gradle/actions/setup-gradle`. If no Develocity plugin is present at all, it does nothing.

## One deliberate difference from the reference

The reference workflow gets `imageDigest` out of the Maven build itself, via its custom-user-data
script, and reads it as a step output. There is no equivalent hook in an unmodified TraderX build,
so this workflow takes the digest from
`docker buildx build --push --metadata-file`, reading `containerimage.digest` and stripping the
`sha256:` prefix. Everything downstream — attestation subject, policy subject, pull URL — is
identical to the reference.

## Required setup

**Secrets**

| Secret | Used for |
|---|---|
| `DV_ACCESS_KEY` | Develocity access key for `https://develocity-field.gradle.com`. Both injects the plugin's credentials and lets the Provenance Governor read the scan. |

**OIDC**

The job runs with `permissions: { contents: read, id-token: write }`.

- Artifactory (`https://develocitytia.jfrog.io`) must have a GitHub OIDC integration named
  `github` configured to trust this repository. `jfrog/setup-jfrog-cli@v5` exchanges the Actions
  token for it and exposes `oidc-user` / `oidc-token`, which are used for `docker login`.
- The Provenance Governor `publish` and `enforce` actions authenticate with the GitHub Actions
  token by default, so no username/password is configured here.

**Repository visibility**

The repository must be publicly readable. On receiving a publish request carrying the `github.*`
annotations, the Provenance Governor calls back to `api.github.com` server-side to enrich the
attestation with the workflow run:

```
GET https://api.github.com/repos/<owner>/<repo>/actions/runs/<run-id>
```

It has no credentials for this repository, so against a private repo that call returns `404` and
publishing fails with `500 External Service Failure`. GitHub returns `404` rather than `403` for
resources the caller cannot see, which makes this look like a missing workflow run rather than a
permissions problem. Either keep the repository public, or arrange for the Provenance Governor to
have a GitHub App installation with read access to it.

**Endpoints**

| Service | URL |
|---|---|
| Develocity | `https://develocity-field.gradle.com` |
| Provenance Governor | `https://develocity-provenance-governor-field.gradle.com` |
| Artifactory Docker repo | `develocitytia.jfrog.io/docker-trial` |

## Known gaps

These are properties of the upstream project, not of this wrapper. They are listed because a demo
that overstates its coverage is worse than one that admits its shape.

- **Upstream runs no tests.** All four test classes are committed to `src/main/test/java`, which is
  not a source set Gradle recognises. On a bare checkout `:test` is `NO-SOURCE` and
  `build/test-results/` is empty. This workflow moves them to `src/test/java` after checkout so
  they actually run — a visible, deliberate fixup step, not an upstream change. Set the
  `relocate_tests` input to `false` on a manual run to see the untouched behaviour, in which case
  **no tests execute at all** and any test-related signal in a Build Scan is absent.
- **Even relocated, the coverage is one test per service, and one of them cannot run.** Each
  service has exactly one `@SpringBootTest` context-load test. Three of them
  (`account-service`, `position-service`, `trade-service`) pass standalone and are relocated.
  `trade-processor-specfirst` is **not** relocated: its context load initialises JPA against
  `jdbc:h2:tcp://localhost:18082/traderx` and fails with `Unable to determine Dialect without JDBC
  metadata` unless a live TraderX H2 server is running. Making it pass would mean starting
  `database-specfirst` as a service alongside the build, which this demo does not do. So the real
  test signal here is three smoke tests, not a test suite.
- **No static analysis anywhere.** There is no Checkstyle, SpotBugs, PMD, Sonar, JaCoCo or
  CycloneDX configuration in any of the five builds. This workflow does not invent any. Adding a
  scan dimension would mean extending the init script, which has not been done.
- **Five disconnected builds.** There is no root `settings.gradle`, no composite build and no
  aggregator. Each project has its own wrapper and its own `settings.gradle`, so there is no
  cross-project dependency graph, no shared build cache benefit between services, and five separate
  Build Scans per run rather than one.
- **`database-specfirst` produces no image.** It has no Dockerfile upstream, so it gets no
  attestation and no policy gate. It is built to prove it still compiles, and nothing more.
- **The policy gate is non-blocking, and it currently fails.** `continue-on-error: true` is kept on
  the `enforce` step, as in the reference demo, so a failing policy is visible in the run summary
  without failing the build. As of the last run, `publish-artifact` evaluates to `UNSATISFIED` on
  three policies:

  | Policy | Why it fails |
  |---|---|
  | `allow-azul-zulu-jdk` | The workflow pins Temurin 21, not Azul Zulu. |
  | `approved-dependency-repos` | Upstream declares `mavenCentral()`, so dependencies do not resolve through Artifactory. |
  | `allow-gradle-and-maven-build-tools` | Not diagnosed. |

  These are inferences from the policy names plus the shapes in
  [develocity-provenance-governor-policies](https://github.com/dv-demos/develocity-provenance-governor-policies);
  the live instance's policy set is named differently from that example repo and cannot be read
  from here, so treat the reasons above as unconfirmed. The first two look fixable — a Zulu
  toolchain, and repository substitution via the init script — but doing so would mean the demo no
  longer builds TraderX the way TraderX builds itself. A gate that visibly catches real differences
  is arguably the better demo.
- **The image tag is unique per run**, not per source revision: `<version>-<run-id>-<run-attempt>`.
  Re-running the workflow on the same upstream SHA produces a new tag and a new attestation.
