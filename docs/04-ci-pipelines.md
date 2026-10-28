# 4. CI pipelines: `.gitlab-ci.yml` deep dive

A **pipeline** is a set of **jobs** grouped into **stages**. Jobs in one stage run in parallel; a later stage starts when the previous one succeeds (unless you use `needs`, below). The pipeline is defined in `.gitlab-ci.yml` at the repository root.

```mermaid
flowchart LR
    subgraph verify
        F["format"]
        S["secret scan"]
    end
    subgraph build
        B["build + unit tests"]
    end
    subgraph scan
        SAST["SAST / dependency scan"]
        Q["quality gate"]
    end
    subgraph package
        IMG["image build + push"]
    end
    subgraph deploy
        GO["bump GitOps tag"]
    end
    F --> B
    S --> B
    B --> SAST
    B --> Q
    B --> IMG
    SAST --> IMG
    Q --> IMG
    IMG --> GO
```

## 4.1 Global keywords

| Keyword | Meaning | Caveat |
|---|---|---|
| `stages` | Ordered stage names. Defaults: `.pre`, `build`, `test`, `deploy`, `.post` | A job with no `stage` runs in `test` |
| `include` | Pull YAML from `local`, `project`, `remote`, `template`, or `component` sources | Resolved before the rest of the file; limited to 150 includes per pipeline and a 30 s resolve time |
| `default` | Fallback values for job keywords (`image`, `before_script`, `retry`, ...) | A job's own value replaces the default; they are not merged |
| `variables` | CI/CD variables for all jobs | Job-level values override |
| `workflow` | Decides whether the pipeline is created at all (`workflow:rules`), its `name`, and `auto_cancel` | If no rule matches, no pipeline runs; only `when: always` or `never` are valid there |

## 4.2 Job keywords

| Keyword | Use |
|---|---|
| `script`, `before_script`, `after_script` | Shell commands; `script` is required |
| `image`, `services` | Container for the job; sidecar service containers (databases for integration tests) |
| `rules` | Decide whether the job exists and its attributes (`when`, `allow_failure`, `variables`). Replaces the legacy `only`/`except` |
| `needs` | Start as soon as listed jobs finish (a DAG) instead of waiting for the whole previous stage. Artifacts are downloaded only from jobs in `needs` |
| `dependencies` | Restrict which earlier jobs' artifacts are fetched |
| `artifacts` | Files kept after the job (for later jobs or download). `expire_in` controls retention. Job-level `artifacts` replaces `default:artifacts` |
| `cache` | Reuse files (NuGet packages) between runs. Not shared between protected and unprotected branches by default; at most four caches per job; `cache:key` cannot contain `/`. `cache:policy` is `pull`, `push` or `pull-push` |
| `environment` | Names a deployment target; enables deployment tracking and rollback in the UI |
| `extends` | Inherit from a hidden job (name starts with `.`) |
| `resource_group` | Serialise jobs so only one runs at a time (for example one deploy per environment) |
| `trigger` | Start a downstream or child pipeline; trigger jobs forward variables by default |
| `parallel`, `parallel:matrix` | Run several copies, or a matrix of variable combinations |
| `id_tokens` | Request an OIDC JWT for the job (to authenticate to Vault) |
| `retry`, `timeout`, `interruptible`, `allow_failure`, `coverage` | Resilience and reporting controls |

## 4.3 `rules`: when jobs run

Rules are evaluated in order and the **first match wins**. Always finish with an explicit fallback.

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"       # merge request pipelines
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH           # main
    - if: $CI_COMMIT_TAG                                    # releases

deploy-dev:
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: on_success
    - when: never
```

- `workflow:rules` defined once prevents the classic duplicate "branch pipeline plus merge request pipeline" for the same commit.
- `rules:changes` limits jobs to changed paths: useful in monorepos.
- `rules:exists`, `rules:variables`, `when: manual` and `allow_failure` cover manual gates.

## 4.4 Reuse: `extends`, anchors, `!reference`, `include`, components

```yaml
.dotnet:
  image: registry.acme.internal/base/dotnet-sdk:10.0
  variables: { NUGET_PACKAGES: "$CI_PROJECT_DIR/.nuget" }
  cache:
    key: { files: [ "**/packages.lock.json" ] }
    paths: [ .nuget ]

.setup:
  script:
    - dotnet --info

build:
  extends: .dotnet
  stage: build
  script:
    - !reference [.setup, script]            # reuse a snippet from another job
    - dotnet restore --locked-mode
    - dotnet build -c Release --no-restore
```

Sharing across repositories, in order of maturity:

1. **`include:project`** with a pinned `ref` (tag). Simple; shared template repo.
2. **CI/CD components**: versioned, parameterised (`inputs`), discoverable.
3. YAML anchors within a single file.

```yaml
include:
  - component: $CI_SERVER_FQDN/acme/platform/pipeline-templates/dotnet-build@1.4.0
    inputs:
      app_path: src/Orders.Api
      sonar_gate: true
```

### Operating a shared template safely

- **Pin versions.** Do not include templates at `ref: main`: one bad commit breaks every repo at once. Use tags or component versions; have one canary repo track `main`.
- **Lint changes** with the CI Lint tool (and `glab ci lint`) in the template repo's own pipeline.
- **Document the variable contract** (for example `APP_NAME`, `APP_PATH`, `TEST_COMMAND`): per-service CI files become 10-20 lines because the template does the work.
- Allow services to override by **job name** (`build: variables: ...`), which is the supported extension point.

## 4.5 Variables, secrets and OIDC

- **Predefined variables**: `CI_COMMIT_SHA`, `CI_COMMIT_SHORT_SHA`, `CI_PIPELINE_SOURCE`, `CI_DEFAULT_BRANCH`, `CI_MERGE_REQUEST_*`, and many more.
- Store secrets as **masked and protected** variables, or use file-type variables. Never `echo` them.
- **ID tokens (OIDC).** `id_tokens` gives each job a signed JWT with its own audience (`aud`). Claims include `project_id`, `namespace_id`, `ref_protected`, `pipeline_id` and `sha`; tokens are signed RS256 and expire at the job timeout (5 minutes if none set). Vault verifies the token, checks bound claims and returns a Vault token. Prefer stable claims (`project_id`) over names in trust policies.

```yaml
fetch-registry-credentials:
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://vault.acme.internal
  script:
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login role=ci-orders jwt=$VAULT_ID_TOKEN)
    - vault kv get -field=token kv/ci/registry-push
```

## 4.6 A complete reference pipeline for a .NET service

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml

stages: [ verify, build, scan, package, deploy ]

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

variables:
  DOTNET_CLI_TELEMETRY_OPTOUT: "1"
  DOTNET_SDK_IMAGE: registry.acme.internal/base/dotnet-sdk:10.0
  APP_NAME: orders-api
  APP_PATH: src/Orders.Api
  IMAGE: registry.acme.internal/orders/$APP_NAME

.dotnet:
  image: $DOTNET_SDK_IMAGE
  cache:
    key: { files: [ "**/packages.lock.json" ] }
    paths: [ .nuget ]
  variables: { NUGET_PACKAGES: "$CI_PROJECT_DIR/.nuget" }

format:
  extends: .dotnet
  stage: verify
  script: [ "dotnet format --verify-no-changes" ]

build-test:
  extends: .dotnet
  stage: build
  services:
    - { name: "postgres:16", alias: db }
  variables:
    POSTGRES_PASSWORD: "ci-only-throwaway"       # throwaway database used only in this job
  script:
    - dotnet restore --locked-mode
    - dotnet build -c Release --no-restore
    - dotnet test -c Release --no-build --logger "junit;LogFileName=unit_tests.xml" --collect:"XPlat Code Coverage"
    - dotnet publish $APP_PATH -c Release -o publish --no-build
  coverage: '/Total\s+\|\s+(\d+(?:\.\d+)?)%/'
  artifacts:
    when: always
    paths: [ publish ]
    reports: { junit: "**/unit_tests.xml" }
    expire_in: 1 week

image-build:
  stage: package
  needs: [ build-test ]
  image: quay.io/buildah/stable
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - buildah bud --build-arg version=$CI_COMMIT_SHORT_SHA -t $IMAGE:$CI_COMMIT_SHORT_SHA .
    - buildah push --creds "$REGISTRY_USER:$REGISTRY_TOKEN" $IMAGE:$CI_COMMIT_SHORT_SHA

bump-gitops-dev:
  stage: deploy
  needs: [ image-build ]
  image: registry.acme.internal/tools/ci-utils:1.0      # contains git and yq
  resource_group: gitops-dev
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - git clone "https://oauth2:${GITOPS_TOKEN}@gitlab.acme.internal/acme/platform/gitops.git"
    - cd gitops/envs/dev/$APP_NAME
    - yq -i '.image.tag = strenv(CI_COMMIT_SHORT_SHA)' values.yaml
    - git -c user.name=ci -c user.email=ci@acme.internal commit -am "$APP_NAME -> $CI_COMMIT_SHORT_SHA"
    - git push
```

Notes:

- `REGISTRY_USER`, `REGISTRY_TOKEN` and `GITOPS_TOKEN` should be masked, protected variables, or better, short-lived credentials fetched from Vault with an ID token.
- Add a container-scanning job (Trivy or GitLab Container Scanning) after `image-build` and fail on fixable critical findings.
- The coverage regular expression depends on your coverage tool's console output; adjust it.

## 4.7 Dynamic and child pipelines

Use `trigger:include` to run a child pipeline from a generated YAML artifact (for example one build job per changed project), or `trigger:project` to start a pipeline in another repository (for example the GitOps repo's validation pipeline).

```yaml
generate:
  stage: build
  script: ./scripts/gen-child-pipeline.sh > child.yml
  artifacts: { paths: [ child.yml ] }

run-children:
  stage: package
  needs: [ generate ]
  trigger:
    include: [ { artifact: child.yml, job: generate } ]
    strategy: depend
```

## 4.8 Versioning and releases

- Tag images with the **commit SHA** (traceability) and additionally a **SemVer** on release tags.
- Derive the assembly version from the tag in CI (`-p:Version=`), or use a tool such as GitVersion.
- Create a GitLab Release (the `release` keyword) from tag pipelines, with generated notes.

## 4.9 Pipeline performance and cost

| Technique | Effect |
|---|---|
| `needs` DAG | Removes stage-barrier waiting |
| NuGet cache keyed on lock file | Skips restores |
| `interruptible: true` on build/test | Newer pushes cancel stale pipelines |
| Split test projects with `parallel` | Faster feedback |
| Docker layer cache in registry (`--cache-from`) | Faster image builds |
| Small, pre-baked job images in your registry | Avoids per-job installs |

## 4.10 Debugging pipelines

| Symptom | Likely cause | First action |
|---|---|---|
| Pipeline not created | No `workflow:rules` match, or an `include` failed to resolve | Pipeline editor / CI Lint; confirm include `ref` exists |
| Job stuck "pending" | No runner with matching tags, or runner capacity | Check runner status and tags |
| `x509: certificate signed by unknown authority` | Internal CA missing from job image | Bake CA into base images |
| Cache never hits | Key contains changing values; protected vs unprotected branch isolation | Key on lock file; check `cache:policy` |
| Every repo breaks at once | Shared template changed at `ref: main` | Pin versions; revert the template commit |

## Done when

- [ ] A merge request runs format, build, test and scans, and cannot merge on red.
- [ ] Merge to `main` produces an image tagged with the commit SHA in your registry.
- [ ] The pipeline commits the new tag to the GitOps repo.
- [ ] The shared template is versioned (tag or component) and linted.
