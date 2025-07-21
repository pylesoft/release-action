# 🚀 Pylesoft Release Action

A simple, reusable GitHub Action for creating releases with automatic branch management and precise changelog control.

## ✨ Features

-   🏷️ **Create releases** with automatic tag creation
-   📝 **Precise changelogs** - specify previous version like GitHub UI
-   🌿 **Automatic branch creation** (e.g., `2025.29.0` → `2025.29.x` for hotfixes)
-   📋 **Draft mode** - review before publishing
-   🔄 **Conflict handling** - automatically replaces existing releases/tags
-   🎯 **Zero configuration** - uses repository context automatically

## 🎮 Quick Start

### 1. Add to your repository

Create `.github/workflows/release.yml`:

```yaml
name: 🚀 Release

on:
    workflow_dispatch:
        inputs:
            version:
                description: "Version to release (e.g., 2025.29.0)"
                required: true
                type: string
            previous_version:
                description: "Previous version for changelog (e.g., 2025.28.0) - optional"
                required: false
                type: string
            draft:
                description: "Create as draft"
                required: false
                type: boolean
                default: false

jobs:
    release:
        runs-on: ubuntu-latest
        permissions:
            contents: write

        steps:
            - uses: actions/checkout@v4
              with:
                  fetch-depth: 0

            - name: Execute release
              uses: pylesoft/release-action@v1
              with:
                  version: ${{ inputs.version }}
                  previous-version: ${{ inputs.previous_version }}
                  draft: ${{ inputs.draft }}
```

### 2. Use it

1. Go to **Actions** → **"Release"**
2. Click **"Run workflow"**
3. Fill in:
    - **Version**: `2025.29.0`
    - **Previous version**: `2025.28.0` (optional)
    - **Draft**: ☐ (unchecked = publish immediately)
4. Click **"Run workflow"**

That's it! 🎉

## 📋 Inputs

| Input              | Description                                              | Required | Default            |
| ------------------ | -------------------------------------------------------- | -------- | ------------------ |
| `version`          | Release version (e.g., `2025.29.0`)                      | ✅ Yes   |                    |
| `previous-version` | Previous version for changelog range (e.g., `2025.28.0`) | ❌ No    | `''` (auto-detect) |
| `draft`            | Create as draft release                                  | ❌ No    | `false`            |

## 📤 Outputs

This action doesn't produce outputs, but creates:

-   🏷️ **Git tag** with the specified version
-   📝 **GitHub release** (draft or published)
-   🌿 **Release branch** for hotfixes (e.g., `2025.29.x`)

## 💡 Usage Examples

### Basic Release

```yaml
- uses: pylesoft/release-action@v1
  with:
      version: "2025.29.0"
```

### Precise Changelog Control

```yaml
- uses: pylesoft/release-action@v1
  with:
      version: "2025.29.0"
      previous-version: "2025.28.0" # Exact range for release notes
      draft: false
```

### Draft Release for Review

```yaml
- uses: pylesoft/release-action@v1
  with:
      version: "2025.29.0"
      previous-version: "2025.28.0"
      draft: true # Create as draft, review later
```

### Auto-Detect Previous Version

```yaml
- uses: pylesoft/release-action@v1
  with:
      version: "2025.29.0"
      # previous-version omitted - GitHub auto-detects
      draft: false
```

## 🎯 Real-World Scenarios

### Regular Release

```
Version: 2025.30.0
Previous: 2025.29.0
Draft: false
→ Published release with changelog from 2025.29.0 to 2025.30.0
→ Creates branch: 2025.30.x
```

### Emergency Hotfix

```
Version: 2025.29.1
Previous: 2025.29.0
Draft: false
→ Hotfix release with only emergency commits
→ Creates branch: 2025.29.x (updated)
```

### Major Release (Include More Changes)

```
Version: 2025.30.0
Previous: 2025.27.0  # Skip 2025.28.0, 2025.29.0
Draft: false
→ Bigger changelog with more features
→ Creates branch: 2025.30.x
```

### Draft for Review

```
Version: 2025.31.0
Previous: 2025.30.0
Draft: true
→ Draft release (review and publish later)
→ Creates branch: 2025.31.x
```

## 🏗️ Advanced Usage

### With Version File Updates

For repositories that need version files updated (like `package.json`, `box.json`, etc.):

```yaml
name: 🚀 Release with Version Updates

on:
    workflow_dispatch:
        inputs:
            version:
                description: "Version to release (e.g., 2025.29.0)"
                required: true
                type: string
            previous_version:
                description: "Previous version for changelog (optional)"
                required: false
                type: string
            draft:
                description: "Create as draft"
                required: false
                type: boolean
                default: false

jobs:
    release:
        runs-on: ubuntu-latest
        permissions:
            contents: write

        steps:
            - uses: actions/checkout@v4
              with:
                  fetch-depth: 0

            - name: Update version files
              run: |
                  VERSION="${{ inputs.version }}"

                  # Update package.json
                  if [[ -f "package.json" ]]; then
                    jq --arg version "$VERSION" '.version = $version' package.json > tmp.$$ && mv tmp.$$ package.json
                  fi

                  # Update box.json (CommandBox)
                  if [[ -f "box.json" ]]; then
                    jq --arg version "$VERSION" '.version = $version' box.json > tmp.$$ && mv tmp.$$ box.json
                  fi

                  # Update ModuleConfig.cfc (ColdBox)
                  if [[ -f "ModuleConfig.cfc" ]]; then
                    sed -i.bak "s/this\.version[[:space:]]*=[[:space:]]*\"[^\"]*\"/this.version = \"$VERSION\"/g" ModuleConfig.cfc
                    rm -f ModuleConfig.cfc.bak
                  fi

                  # Commit changes
                  git config user.name "Release Bot"
                  git config user.email "release-bot@pylesoft.com"

                  if ! git diff --quiet; then
                    git add -A
                    git commit -m "Update version to $VERSION"
                    git push origin ${{ github.ref_name }}
                  fi

            - name: Execute release
              uses: pylesoft/release-action@v1
              with:
                  version: ${{ inputs.version }}
                  previous-version: ${{ inputs.previous_version }}
                  draft: ${{ inputs.draft }}
```

### Automated Hotfix Workflow

```yaml
name: 🚨 Hotfix

on:
    workflow_dispatch:
        inputs:
            base_version:
                description: "Base version (e.g., 2025.29.0)"
                required: true
                type: string

jobs:
    hotfix:
        runs-on: ubuntu-latest
        permissions:
            contents: write

        steps:
            - uses: actions/checkout@v4
              with:
                  fetch-depth: 0

            - name: Calculate hotfix version
              id: version
              run: |
                  BASE="${{ inputs.base_version }}"

                  if [[ "$BASE" =~ ^([0-9]+\.[0-9]+)\.([0-9]+)$ ]]; then
                    PREFIX="${BASH_REMATCH[1]}"
                    PATCH=$((${BASH_REMATCH[2]} + 1))
                    HOTFIX_VERSION="${PREFIX}.${PATCH}"
                  else
                    echo "❌ Invalid version format: $BASE"
                    exit 1
                  fi

                  echo "version=$HOTFIX_VERSION" >> $GITHUB_OUTPUT

            - name: Execute hotfix release
              uses: pylesoft/release-action@v1
              with:
                  version: ${{ steps.version.outputs.version }}
                  previous-version: ${{ inputs.base_version }}
                  draft: false
```

## 🔧 How It Works

### Changelog Generation

#### When you specify `previous-version`:

```bash
gh release create "2025.29.0" --notes-start-tag "2025.28.0"
```

→ Includes commits from `2025.28.0` to `2025.29.0`

#### When you omit `previous-version`:

```bash
gh release create "2025.29.0" --generate-notes
```

→ GitHub automatically detects the previous release

### Branch Creation

The action automatically creates a release branch for hotfixes:

-   `2025.29.0` → `2025.29.x`
-   `2025.30.12` → `2025.30.x`
-   `1.5.3` → `1.5.x`

If the branch already exists, it updates it to point to the new release.

### Conflict Resolution

If a release or tag already exists:

1. Deletes the existing release
2. Deletes the existing tag (locally and remotely)
3. Creates the new tag and release
4. Updates the release branch

## ✅ Benefits

### 🎯 **Simple & Reusable**

-   One action, works in any repository
-   No configuration needed per repo
-   Uses GitHub's native functionality

### 📝 **Professional Release Notes**

-   Same control as GitHub UI
-   Precise commit range selection
-   Auto-generated or custom range

### 🌿 **Automatic Branch Management**

-   Creates hotfix branches automatically
-   Follows semantic versioning patterns
-   Handles conflicts gracefully

### 🔒 **Secure by Default**

-   Uses `github.token` automatically
-   No additional permissions needed
-   Runs in isolated environment

### 🚀 **Zero Infrastructure**

-   No servers or dependencies
-   Runs on GitHub's infrastructure
-   Update once, works everywhere

## 📋 Requirements

-   Repository with `contents: write` permission
-   Git history with commits to generate release notes from
-   Valid version format (recommended: `YYYY.X.X` or `X.Y.Z`)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with a sample repository
5. Submit a pull request

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details.

## 🔗 Links

-   [GitHub Repository](https://github.com/pylesoft/release-action)
-   [GitHub Actions Documentation](https://docs.github.com/en/actions)
-   [Release Notes Best Practices](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes)
