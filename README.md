<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# 🚀 Nexus Staging Action

Composite GitHub Action to manage the Sonatype Nexus 2 staging repository
lifecycle via REST API. Pure bash+curl implementation — no lftools or Python
dependency required.

## Features

- **Stage**: Create staging repo, upload Maven artifacts, close for validation
- **Close**: Close an existing staging repository
- **Promote**: Release a closed staging repository to the releases repository
- **Drop**: Drop/delete a staging repository (cleanup on failure)
- Compatible with Nexus 2 staging API
- Writes `staging-repo.txt` in JJB-compatible format
- Generates GitHub Actions step summary

<!-- markdownlint-disable MD013 MD060 -->

## Nexus 2 REST API Reference

| Operation | Method | Endpoint                                                       |
| --------- | ------ | -------------------------------------------------------------- |
| Create    | POST   | `/service/local/staging/profiles/{profile-id}/start`           |
| Upload    | PUT    | `/service/local/staging/deployByRepositoryId/{repo-id}/{path}` |
| Close     | POST   | `/service/local/staging/profiles/{profile-id}/finish`          |
| Promote   | POST   | `/service/local/staging/profiles/{profile-id}/promote`         |
| Drop      | POST   | `/service/local/staging/profiles/{profile-id}/drop`            |

<!-- markdownlint-enable MD013 MD060 -->

All API payloads use XML:

```xml
<promoteRequest><data>
  <description>text</description>
  <stagedRepositoryId>repo-id</stagedRepositoryId>
</data></promoteRequest>
```

## Usage

### Stage Mode (Create + Upload + Close)

```yaml
- name: 'Stage Maven artifacts to Nexus'
  id: nexus-stage
  uses: askb/nexus-staging-action@main
  with:
    nexus-server: 'https://nexus.opendaylight.org'
    nexus-username: ${{ secrets.NEXUS_USERNAME }}
    nexus-password: ${{ secrets.NEXUS_PASSWORD }}
    staging-profile-id: ${{ vars.STAGING_PROFILE_ID }}
    mode: 'stage'
    m2repo-path: 'm2repo'
    description: 'CI build ${{ github.run_id }}'
```

### Promote Mode (Release)

```yaml
- name: 'Promote staging repository'
  uses: askb/nexus-staging-action@main
  with:
    nexus-server: 'https://nexus.opendaylight.org'
    nexus-username: ${{ secrets.NEXUS_USERNAME }}
    nexus-password: ${{ secrets.NEXUS_PASSWORD }}
    staging-profile-id: ${{ vars.STAGING_PROFILE_ID }}
    mode: 'promote'
    staging-repo-id: ${{ needs.stage.outputs.staging-repo-id }}
```

### Close Mode

```yaml
- name: 'Close staging repository'
  uses: askb/nexus-staging-action@main
  with:
    nexus-server: 'https://nexus.opendaylight.org'
    nexus-username: ${{ secrets.NEXUS_USERNAME }}
    nexus-password: ${{ secrets.NEXUS_PASSWORD }}
    staging-profile-id: ${{ vars.STAGING_PROFILE_ID }}
    mode: 'close'
    staging-repo-id: 'example-1234'
```

### Drop Mode (Cleanup)

```yaml
- name: 'Drop staging repository'
  if: failure()
  uses: askb/nexus-staging-action@main
  with:
    nexus-server: 'https://nexus.opendaylight.org'
    nexus-username: ${{ secrets.NEXUS_USERNAME }}
    nexus-password: ${{ secrets.NEXUS_PASSWORD }}
    staging-profile-id: ${{ vars.STAGING_PROFILE_ID }}
    mode: 'drop'
    staging-repo-id: ${{ steps.nexus-stage.outputs.staging-repo-id }}
```

### Full Pipeline Example

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 'Build with Maven'
        run: mvn deploy -DaltDeploymentRepository=local::file:m2repo

  stage:
    needs: build
    runs-on: ubuntu-latest
    outputs:
      staging-repo-id: ${{ steps.stage.outputs.staging-repo-id }}
    steps:
      - name: 'Stage to Nexus'
        id: stage
        uses: askb/nexus-staging-action@main
        with:
          nexus-server: ${{ vars.NEXUS_SERVER }}
          nexus-username: ${{ secrets.NEXUS_USERNAME }}
          nexus-password: ${{ secrets.NEXUS_PASSWORD }}
          staging-profile-id: ${{ vars.STAGING_PROFILE_ID }}
          mode: 'stage'

  release:
    needs: stage
    runs-on: ubuntu-latest
    steps:
      - name: 'Promote staging repo'
        uses: askb/nexus-staging-action@main
        with:
          nexus-server: ${{ vars.NEXUS_SERVER }}
          nexus-username: ${{ secrets.NEXUS_USERNAME }}
          nexus-password: ${{ secrets.NEXUS_PASSWORD }}
          staging-profile-id: ${{ vars.STAGING_PROFILE_ID }}
          mode: 'promote'
          staging-repo-id: ${{ needs.stage.outputs.staging-repo-id }}
```

## Inputs

<!-- markdownlint-disable MD013 MD060 -->

| Input                | Description                                                     | Required | Default                  |
| -------------------- | --------------------------------------------------------------- | -------- | ------------------------ |
| `nexus-server`       | Nexus server URL (e.g., `https://nexus.opendaylight.org`)       | ✅        | —                        |
| `nexus-username`     | Nexus username for authentication                               | ✅        | —                        |
| `nexus-password`     | Nexus password for authentication                               | ✅        | —                        |
| `staging-profile-id` | Nexus staging profile ID (per-project)                          | ✅        | —                        |
| `mode`               | Operation mode: `stage`, `close`, `promote`, `drop`             | ✅        | `stage`                  |
| `m2repo-path`        | Path to local Maven repo directory (for `stage` mode)           | ❌        | `m2repo`                 |
| `staging-repo-id`    | Existing staging repo ID (for `close`/`promote`/`drop`)         | ❌        | —                        |
| `description`        | Description for the staging repository                          | ❌        | `GitHub Actions staging` |

<!-- markdownlint-enable MD013 MD060 -->

## Outputs

<!-- markdownlint-disable MD013 MD060 -->

| Output             | Description                                  |
| ------------------ | -------------------------------------------- |
| `staging-repo-id`  | Staging repository ID (e.g., `example-1234`) |
| `staging-repo-url` | Staging repository URL                       |

<!-- markdownlint-enable MD013 MD060 -->

## How It Works

### Stage Mode

1. **Create** — POST to `/staging/profiles/{id}/start` to open a new
   staging repository
2. **Upload** — PUT each file from `m2repo-path` to
   `/staging/deployByRepositoryId/{repo-id}/{relative-path}`
3. **Close** — POST to `/staging/profiles/{id}/finish` to close the
   repository and trigger Nexus validation rules
4. Writes `archives/staging-repo.txt` in JJB-compatible format:
   `{repo-id} {repo-url}`

### Promote Mode

1. **Verify** — GET `/staging/repository/{repo-id}` to confirm the
   staging repository exists
2. **Promote** — POST to `/staging/profiles/{id}/promote` to release
   artifacts to the releases repository

### Close Operation

- POST to `/staging/profiles/{id}/finish` to close an
  opened staging repository

### Drop Operation

- POST to `/staging/profiles/{id}/drop` to delete the staging
  repository (useful for cleanup on failure)

## Comparison with lftools

<!-- markdownlint-disable MD013 MD060 -->

| Feature        | lftools                        | nexus-staging-action  |
| -------------- | ------------------------------ | --------------------- |
| Runtime        | Python + pip                   | bash + curl           |
| Install        | `pip install lftools`          | None (built-in)       |
| Stage          | `lftools deploy nexus-stage`   | `mode: stage`         |
| Release        | `lftools nexus release`        | `mode: promote`       |
| Drop           | Manual API call                | `mode: drop`          |
| CI Integration | Script wrapper                 | Native GitHub Action  |
| Output format  | stdout parsing                 | GitHub Action outputs |

<!-- markdownlint-enable MD013 MD060 -->

## Requirements

- Sonatype Nexus 2 server with staging profiles configured
- Nexus user account with staging permissions
- Staging profile ID for the target project
- `curl` available on the runner (default on all GitHub runners)

## License

[Apache-2.0](LICENSES/Apache-2.0.txt)
