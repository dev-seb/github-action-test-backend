# github-action-test-backend

## Release Workflow

This document describes the automated release workflow starting from the GitHub Actions workflow Start a new release from develop branch (Release start).

Notes:

- CI-related steps named `Fake CI` are ignored in this flow.
- The release process is manually triggered by a user.
- A manual approval + merge commit is required for the release PR after it is created and after the release automerge workflow.

### Overview Diagram

```mermaid
flowchart TD
	 U([User]) --> A[Start a new release from develop branch<br/>Release start]
	 A --> B[Prepare release changelog and version bump<br/>Release]
	 B --> C[Automerge release changelog and version bump PR<br/>Release automerge]
	 C --> D[Create release PR<br/>Release]
	 D --> E{User approves and performs<br/>merge commit of release PR}
	 E --> F[Create merge back PR<br/>Merge back]
```

### Sequence Diagram

```mermaid
sequenceDiagram
	 autonumber
	 participant U as User
	 participant PR as Pull requests
	 participant RS as Start a new release from develop branch (Release start)
	 participant R as Release workflow
	 participant RA as Release automerge workflow
	 participant MB as Merge back workflow

	 U->>RS: Manually trigger workflow from Actions tab
	 RS->>R: Start a new release from develop branch (Release start)
	 R->>R: Prepare release changelog and version bump (Release)
	 R->>RA: Trigger automerge release changelog and version bump PR
	 RA->>RA: Automerge release changelog and version bump PR (Release automerge)
	 RA->>R: Automerge sequence completed
	 R->>PR: Create release PR (Release)
	 U->>PR: Approve and merge release PR (manual merge commit)
	 PR->>MB: Trigger merge back process
	 MB->>MB: Create merge back PR (Merge back)
```

### Branch Flow Diagram

```mermaid
gitGraph
	 commit id: "main baseline"
	 branch develop
	 checkout develop
	 commit id: "start release"
	 branch release
	 checkout release
	 commit id: "release prep"
	 branch release-changelog-and-bump
	 checkout release-changelog-and-bump
	 commit id: "changelog + version bump"
	 checkout release
	 merge release-changelog-and-bump
	 commit id: "release PR created"
	 checkout main
	 merge release
	 commit id: "release merged to main"
	 checkout develop
	 merge main
	 commit id: "merge back PR"
```

## Step-by-Step Release Process

1. **Start a new release from develop branch** (`Release start`)
   - Entry point of the release process, manually triggered by a user from the GitHub Actions tab.
   - Kicks off the main release workflow on top of `develop`.

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
