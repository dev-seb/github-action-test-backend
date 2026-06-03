# github-action-test-backend

## Release Workflow

This document describes the automated release workflow starting from the GitHub Actions workflow Start a new release from develop branch (Release start).

Notes:

- CI-related steps named `Fake CI` are ignored in this flow.
- The release process is manually triggered by a user.
- A manual approval + merge commit is required for the release PR after it is created and after the release automerge workflow.

### Sequence Diagram


Frontend application:
```mermaid
---
title: Frontend release sequence
config:
  mirrorActors: false
---
sequenceDiagram
    actor User
    participant Live as efficientship-live
    participant GA_Live as GitHub Actions (live)
    participant AWS

    %% Start release
    User->>GA_Live: Trigger release workflow (workflow_dispatch)
    GA_Live->>Live: Create PR (develop → main)

    %% First PR
    User->>Live: Review & approve PR
    User->>Live: Merge PR

    %% Versioning PR
    Live->>GA_Live: Trigger workflow (on merge)
    GA_Live->>Live: Create PR (changelog + version bump)

    User->>Live: Review & approve PR
    User->>Live: Merge PR

    %% Release
    Live->>GA_Live: Trigger workflow (on merge)
    GA_Live->>Live: Create GitHub Release + Tag

    %% Build & deploy
    GA_Live->>AWS: Deploy to S3

    %% Merge back PR
    GA_Live->>Live: Create merge back PR (main → develop)

    User->>Live: Review & approve PR
    User->>Live: Merge PR
```

Backend application:
```mermaid
---
title: Backend release sequence
config:
  mirrorActors: false
---
sequenceDiagram
    actor User
    participant Backend as efficientship-backend
    participant GA_Backend as GitHub Actions (backend)
    participant Infra as efficientship-infra
    participant GA_Infra as GitHub Actions (infra)
    participant AWS

    %% Start release
    User->>GA_Backend: Trigger release workflow (workflow_dispatch)
    GA_Backend->>Backend: Create PR (develop → main)

    %% First PR
    User->>Backend: Review & approve PR
    User->>Backend: Merge PR

    %% Versioning PR
    Backend->>GA_Backend: Trigger workflow (on merge)
    GA_Backend->>Backend: Create PR (changelog + version bump)

    User->>Backend: Review & approve PR
    User->>Backend: Merge PR

    %% Release
    Backend->>GA_Backend: Trigger workflow (on merge)
    GA_Backend->>Backend: Create GitHub Release + Tag

    %% Build & deploy backend
    GA_Backend->>AWS: Build Docker image
    GA_Backend->>AWS: Push to ECR

    %% Infra update
    GA_Backend->>Infra: Create PR (version bump)

    User->>Infra: Review & approve PR
    User->>Infra: Merge PR

    %% Deploy infra
    Infra->>GA_Infra: Trigger workflow (on merge)
    GA_Infra->>AWS: Deploy to ECS

    %% Merge back PR
    GA_Backend->>Backend: Create merge back PR (main → develop)

    User->>Backend: Review & approve PR
    User->>Backend: Merge PR
```

### Sequence Diagram

Release:

```mermaid
---
Release process
---
gitGraph
commit id: "init"

branch "develop"
checkout "develop"
commit id: "init develop"

commit id: "new PR branch"

branch "feature/new-feature"
checkout "feature/new-feature"
commit id: "feat: commit 1"
commit id: "feat: commit 2"

checkout "develop"
merge "feature/new-feature" id: "squash"

checkout "main"
merge "develop" id: "merge"

branch "release-please--main"
checkout "release-please--main"
commit id: "chore(release): 1.0.0"

checkout "main"
merge "release-please--main" type: HIGHLIGHT tag: "1.0.0"

checkout "develop"
merge "main" id: "merge back"
```

Hotfix:

```mermaid
---
Release process
---
gitGraph
commit id: "init"

branch "develop"
checkout "develop"
commit id: "init develop"

checkout "main"
commit id: "new PR branch"

branch "fix/hotfix"
checkout "fix/hotfix"
commit id: "fix: commit 1"
commit id: "fix: commit 2"

checkout "main"
merge "fix/hotfix" id: "squash"

branch "release-please--main"
checkout "release-please--main"
commit id: "chore(release): 1.0.1"

checkout "main"
merge "release-please--main" type: HIGHLIGHT tag: "1.0.1"

checkout "develop"
merge "main" id: "merge back"
```

## Step-by-Step Release Process

1. **Start a new release from develop branch** (`Release start`)
   - Entry point of the release process, manually triggered by a user from the GitHub Actions tab.
   - Kicks off the main release workflow on top of `develop`.

5. **Manual approval and merge commit of the release PR** (User action)
   - This manual step occurs after release PR creation and after the `Release automerge` workflow.
   - A user reviews, approves, and performs the merge commit on the release PR.

2. **Prepare release changelog and version bump** (`Release`)
   - Automatically computes the next version and prepares changelog/version updates.
   - Produces the changes that will be proposed through a release PR.

3. **Automerge release changelog and version bump PR** (`Release automerge`)
   - Runs automatically after release preparation.
   - Ensures release metadata updates are fully integrated.

4. **Create release PR** (`Release`)
   - Automatically opens the release pull request containing changelog and version bump.
   - This step happens after the `Release automerge` workflow.

5. **Manual approval and merge commit of the release PR** (User action)
   - This manual step occurs after release PR creation and after the `Release automerge` workflow.
   - A user reviews, approves, and performs the merge commit on the release PR.

6. **Create merge back PR** (`Merge back`)
   - Automatically opens a PR to merge changes from `main` back into `develop`.
   - Keeps `develop` aligned with release changes already on `main`.
