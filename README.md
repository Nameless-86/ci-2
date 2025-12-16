# CI-2 Repository - Workflow Explanation

## Overview

This repository (`ci-2`) depends on `ci-1` as a Go module. When `ci-1` changes (specifically when its `go.sum` changes), this repository automatically:
1. Updates its dependency on `ci-1` to the latest version
2. Builds a binary from the updated code
3. Commits the binary back to this repository (without committing `go.mod` or `go.sum`)

## The Big Picture

This repository is the "consumer" in the relationship:
- **Listens** for events from `ci-1` via `repository_dispatch`
- **Updates** its dependency when notified
- **Builds** a new binary with the updated dependency
- **Commits** the binary back to the repository

## What is repository_dispatch?

`repository_dispatch` is a GitHub feature that allows one repository to send a custom event to another repository. In our setup:

- **Sender (ci-1)**: Sends an event with `event_type: "go_sum_changed-at-ci-1"`
- **Receiver (ci-2)**: This workflow listens for that exact event type and runs when it receives it

This creates a cross-repository trigger - when `ci-1` changes, `ci-2` automatically responds.

## Workflow File: `.github/workflows/build-on-dispatch.yaml`

Let's break down this file line by line:

### Workflow Name
```yaml
name: Update from ci-1 and build
```
- Human-readable name shown in GitHub Actions UI

### Trigger Condition
```yaml
on:
  repository_dispatch:
    types: [go_sum_changed-at-ci-1]
```

This is the key part - it defines when the workflow runs:

- **`repository_dispatch:`** - Listens for repository_dispatch events from other repositories
- **`types: [go_sum_changed-at-ci-1]`** - Only triggers when the event type matches exactly
  - This must match the `event_type` sent by `ci-1` in its API call
  - If the types don't match, the workflow won't run

**Important**: This workflow file must be on the `main` branch (or default branch) for `repository_dispatch` to work. GitHub only checks workflows on the default branch for incoming events.

### Permissions
```yaml
permissions:
  contents: write
```
- **`contents: write`** - Needed because we'll be committing the binary back to the repository
- Without this, the workflow can't push changes

### Jobs Section
```yaml
jobs:
  update-and-build:
    runs-on: ubuntu-latest
```

- **`update-and-build:`** - Name of the job
- **`runs-on: ubuntu-latest`** - Runs on a fresh Ubuntu Linux virtual machine

### Step 1: Checkout CI-2
```yaml
      - name: Step 1 – checkout ci-2
        uses: actions/checkout@v4
```

- Downloads the `ci-2` repository code to the runner
- Makes all files available in the workspace
- This is needed before we can run `go` commands or commit changes

### Step 2: Setup Go
```yaml
      - name: Step 2 – setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.23.1"
```

- **`uses: actions/setup-go@v5`** - Pre-built action that installs Go
- **`with:`** - Parameters for the action
- **`go-version: "1.23.1"`** - Installs Go version 1.23.1
  - Note: If `ci-1` requires a newer Go version, Go will auto-upgrade during `go get`

### Step 3: Log Start
```yaml
      - name: Step 3 – log start
        run: echo "Starting dependency update from ci-1 and build in ci-2"
```

- Simple logging step
- Prints a message to the workflow logs
- Helps with debugging and understanding workflow progress

### Step 4: Update Dependency
```yaml
      - name: Step 4 – go get -u ci-1 and tidy
        run: |
          echo "Running: go get -u github.com/Nameless-86/ci-1@latest"
          go get -u github.com/Nameless-86/ci-1@latest

          echo "Running: go mod tidy"
          go mod tidy
```

This step updates the dependency on `ci-1`:

- **`go get -u github.com/Nameless-86/ci-1@latest`**
  - `go get` - Downloads and adds a dependency
  - `-u` - Update flag (updates to latest version)
  - `github.com/Nameless-86/ci-1@latest` - The module path and version
    - `@latest` means get the latest commit from the default branch
  - This modifies `go.mod` and `go.sum` files

- **`go mod tidy`**
  - Cleans up `go.mod` and `go.sum`
  - Removes unused dependencies
  - Ensures files are in correct format

**Important**: At this point, `go.mod` and `go.sum` have been updated with the new dependency version. However, we don't want to commit these files - we only want to commit the binary.

### Step 5: Build Binary
```yaml
      - name: Step 5 – build binary
        run: |
          echo "Building binary to ./bin/ci2-app"
          mkdir -p bin
          GOFLAGS="-mod=readonly" go build -o bin/ci2-app ./...
```

- **`mkdir -p bin`** - Creates a `bin` directory (the `-p` flag means "don't error if it already exists")
- **`GOFLAGS="-mod=readonly"`** - Prevents Go from modifying `go.mod`/`go.sum` during build
- **`go build -o bin/ci2-app ./...`**
  - `-o bin/ci2-app` - Output filename and location
  - `./...` - Builds all packages in current directory and subdirectories
- The binary is now in `bin/ci2-app`

### Step 6: Upload Artifact
```yaml
      - name: Step 6 – optionally upload binary artifact
        uses: actions/upload-artifact@v4
        with:
          name: ci2-app
          path: bin/ci2-app
```

- **`uses: actions/upload-artifact@v4`** - Pre-built action for saving files
- **`name: ci2-app`** - Name of the artifact (shows in GitHub UI)
- **`path: bin/ci2-app`** - File to upload
- This makes the binary downloadable from the workflow run page
- Useful for debugging or manual downloads
- Artifacts are temporary (expire after some time)

### Step 7: Commit Binary (Without go.mod/go.sum)
```yaml
      - name: Step 7 – commit binary only (no go.mod / go.sum)
        env:
          CI2_PUSH_PAT: ${{ secrets.CI2_PUSH_PAT }}
        run: |
          echo "Preparing commit with binary only (excluding go.mod/go.sum)"

          # Move go.mod and go.sum away so they are NOT included in the commit
          mkdir -p .gometa
          if [ -f go.mod ]; then mv go.mod .gometa/go.mod; fi
          if [ -f go.sum ]; then mv go.sum .gometa/go.sum; fi

          git config user.name "ci-bot"
          git config user.email "ci-bot@example.com"

          git add bin/ci2-app
          git status

          git commit -m "Update ci-2 binary from ci-1 change: ${GITHUB_SHA}" || echo "No binary changes to commit"

          echo "Pushing commit with binary only to main"
          git push https://$CI2_PUSH_PAT@github.com/Nameless-86/ci-2.git HEAD:main

          # Restore go.mod and go.sum in the workspace (still uncommitted)
          if [ -f .gometa/go.mod ]; then mv .gometa/go.mod go.mod; fi
          if [ -f .gometa/go.sum ]; then mv .gometa/go.sum go.sum; fi

          echo "Finished: binary committed, go.mod/go.sum left unchanged in repo history"
```

This is the most complex step. Let's break it down:

**Environment Variable:**
- **`CI2_PUSH_PAT: ${{ secrets.CI2_PUSH_PAT }}`** - Loads a PAT for pushing to this repository
  - We need a PAT because the default `GITHUB_TOKEN` might not have write permissions
  - Or we could use `GITHUB_TOKEN` if permissions are set correctly

**Moving go.mod and go.sum:**
```bash
mkdir -p .gometa
if [ -f go.mod ]; then mv go.mod .gometa/go.mod; fi
if [ -f go.sum ]; then mv go.sum .gometa/go.sum; fi
```
- Creates a temporary directory `.gometa`
- Moves `go.mod` and `go.sum` out of the way
- `if [ -f ... ]` checks if file exists before moving
- This ensures these files won't be included in the git commit

**Git Configuration:**
```bash
git config user.name "ci-bot"
git config user.email "ci-bot@example.com"
```
- Sets the git user for commits
- Required before making commits

**Staging and Committing:**
```bash
git add bin/ci2-app
git status
git commit -m "Update ci-2 binary from ci-1 change: ${GITHUB_SHA}" || echo "No binary changes to commit"
```
- **`git add bin/ci2-app`** - Stages only the binary file
- **`git status`** - Shows what will be committed (for logging)
- **`git commit`** - Creates a commit
  - `-m "..."` - Commit message
  - `${GITHUB_SHA}` - GitHub Actions variable containing the commit SHA that triggered the original workflow
  - `|| echo "..."` - If commit fails (e.g., no changes), just print a message instead of failing

**Pushing:**
```bash
git push https://$CI2_PUSH_PAT@github.com/Nameless-86/ci-2.git HEAD:main
```
- Pushes the commit to the `main` branch
- Uses PAT for authentication (embedded in URL)
- `HEAD:main` means push current HEAD to main branch

**Restoring Files:**
```bash
if [ -f .gometa/go.mod ]; then mv .gometa/go.mod go.mod; fi
if [ -f .gometa/go.sum ]; then mv .gometa/go.sum go.sum; fi
```
- Moves `go.mod` and `go.sum` back to their original locations
- They're restored in the workspace but were never committed
- This keeps the workspace clean for any subsequent steps

## Why Not Commit go.mod/go.sum?

The workflow intentionally avoids committing `go.mod` and `go.sum` because:
- These files change every time we update the dependency
- Committing them would create a lot of commit noise
- The binary is the actual deliverable
- The dependency versions are already locked in `go.sum` (even if not committed, they're in the repo from previous commits)

However, note that this means:
- The `go.mod` and `go.sum` in the repository might be out of sync with the binary
- If someone clones the repo, they'll need to run `go mod download` to get dependencies
- The binary was built with the updated dependencies, but the repo doesn't reflect that

## Required Secrets

You need to set up this secret in `ci-2` repository settings:

- **`CI2_PUSH_PAT`** - A Personal Access Token with `repo` scope for pushing to `ci-2`
  - Can be the same token as `CI2_PAT` in `ci-1`, or a different one
  - Must have write access to `ci-2` repository

Alternatively, you can use the default `GITHUB_TOKEN` if you set permissions correctly:
```yaml
permissions:
  contents: write
```
And change the push command to:
```bash
git push origin main
```
(But using a PAT is more explicit and reliable)

## Workflow Execution Flow

1. `ci-1` sends a `repository_dispatch` event with type `go_sum_changed-at-ci-1`
2. GitHub receives the event and checks `ci-2` for workflows listening to that event type
3. This workflow starts on a fresh Ubuntu runner
4. Step 1: Checks out `ci-2` code
5. Step 2: Installs Go 1.23.1
6. Step 3: Logs start message
7. Step 4: Updates dependency on `ci-1` to latest version (modifies `go.mod` and `go.sum`)
8. Step 5: Builds binary with updated dependency
9. Step 6: Uploads binary as downloadable artifact
10. Step 7: Moves `go.mod`/`go.sum` out of way, commits only the binary, pushes to `main`, restores files

## Testing the Workflow

You can manually trigger this workflow:
1. Go to `ci-2` repository → Actions tab
2. Click on "Update from ci-1 and build"
3. Click "Run workflow" button (if `workflow_dispatch` is added to triggers)

Or trigger it from `ci-1`:
1. Manually run the `ci-1` workflow (which sends the repository_dispatch)
2. The `ci-2` workflow should automatically start

## Troubleshooting

**Workflow doesn't run:**
- Check that the `event_type` in `ci-1` matches the `types` in `ci-2` workflow
- Ensure the workflow file is on the `main` branch (default branch)
- Check that `ci-1` successfully sent the repository_dispatch (check `ci-1` workflow logs)

**Build fails:**
- Check that `ci-1` module path is correct (`github.com/Nameless-86/ci-1`)
- Ensure `ci-1` is publicly accessible or the PAT has access
- Check Go version compatibility

**Commit fails:**
- Verify `CI2_PUSH_PAT` secret exists and has correct permissions
- Check that `contents: write` permission is set
- Ensure the PAT hasn't expired
