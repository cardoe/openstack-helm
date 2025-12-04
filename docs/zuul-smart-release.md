# Smart Chart Release Process

## Overview

The Zuul release process has been optimized to only build and publish Helm charts that have actually changed, rather than rebuilding all charts on every merge to master. This significantly reduces build times and avoids unnecessary version bumps for unchanged charts.

This implementation uses the official [Helm Chart Testing (ct)](https://github.com/helm/chart-testing) tool, which is the community-standard tool for detecting changed charts.

## How It Works

### 1. Change Detection

When a commit is merged to master, the `ct list-changed` command from the chart-testing tool analyzes the git diff to determine which chart directories have been modified.

The tool considers:
- Direct changes to chart files (e.g., `nova/templates/*.yaml`)
- Changes to chart metadata (`Chart.yaml`)
- Changes to chart values (`values.yaml`)
- Changes to chart dependencies

Chart-testing is already used in the GitHub Actions workflow (`.github/workflows/lint.yaml`) for linting, ensuring consistency across CI systems.

### 2. Special Cases

**No Changes Detected**: If no charts are detected as changed (e.g., only documentation changes), all charts are still built as a safety fallback to ensure nothing is missed.

### 3. Build Process

The `playbooks/build-chart.yaml` playbook:
1. Installs the chart-testing (ct) tool if not already present
2. Runs `ct list-changed` to get the list of changed charts
3. Iterates through the changed charts
4. Runs `make <chart-name>` for each changed chart
5. Generates changelogs and packages only the changed charts
6. Falls back to `make all` if no changes are detected

### 4. Publishing

The `playbooks/publish/post.yaml` playbook:
1. Downloads the current Helm repository index from tarballs.opendev.org
2. Merges the newly built charts into the existing index
3. Only publishes the changed chart packages (*.tgz files)
4. Updates the repository index with the new chart versions

## Benefits

1. **Faster CI/CD**: Only changed charts are built, reducing build time proportionally
2. **Cleaner Version History**: Charts only get new versions when they actually change
3. **Reduced Storage**: Fewer unnecessary chart versions are published
4. **Better Traceability**: Users can see exactly which charts were updated in each release

## Example

If a developer only modifies the `nova` chart:
- Before: All ~80 charts are rebuilt and published
- After: Only the `nova` chart is rebuilt and published

## Debugging

To manually check which charts would be built, use the chart-testing tool:

```bash
# Install chart-testing (if not already installed)
brew install chart-testing  # macOS
# or download from https://github.com/helm/chart-testing/releases

# Compare against a specific branch
ct list-changed --target-branch origin/master --chart-dirs=.

# Compare against main branch
ct list-changed --target-branch origin/main --chart-dirs=.
```

The `ct list-changed` command outputs changed chart directories (one per line).

### Chart Testing Configuration

Chart-testing can be configured using a `ct.yaml` file in the repository root. See the [chart-testing documentation](https://github.com/helm/chart-testing/blob/main/doc/ct_list-changed.md) for more options.
