# sedona-sync-action

This repository publishes Docker images and Maven snapshots from [Apache Sedona](https://github.com/apache/sedona).

## Docker images

[`Sync Sedona docker`](.github/workflows/sync-docker.yml) runs daily and manually. It builds from Sedona's default branch and publishes the `latest` tag to [apache/sedona on Docker Hub](https://hub.docker.com/r/apache/sedona).

## Maven snapshots

[`Publish Sedona snapshots`](.github/workflows/publish-snapshot.yml), adapted from [apache/sedona#3406](https://github.com/apache/sedona/pull/3406), checks `apache/sedona`'s `master` every 15 minutes and can also be started manually. It publishes only when the upstream commit differs from the last successfully published commit. All variants build from the same resolved commit, which is recorded in the run summary:

| Spark | Scala | JDK |
| --- | --- | --- |
| 3.5 | 2.12 | 11 |
| 3.5 | 2.13 | 11 |
| 4.0 | 2.13 | 17 |
| 4.1 | 2.13 | 17 |

Artifacts are deployed to the [ASF snapshot repository](https://repository.apache.org/content/repositories/snapshots). The workflow refuses versions that do not end in `-SNAPSHOT`. Active publishing runs are not cancelled by newer runs. The shared Maven modules are deployed by the Spark 3.5 / Scala 2.12 build; the other builds deploy their variant-specific Spark artifacts.

After all four builds succeed, the workflow records the upstream SHA in a GitHub deployment with the task `record-snapshot`. Subsequent checks skip that SHA. A failed or cancelled build does not advance the checkpoint, so the next check retries if upstream has not changed, or publishes the newer commit if it has. A GitHub API failure stops the check instead of starting an unchecked publish. The checkpoint uses the built-in `GITHUB_TOKEN`; no additional PAT or upstream workflow is needed. Only the final job has `deployments: write`, and it has no Nexus secrets or source checkout.

Checks are scheduled at minutes 7, 22, 37, and 52 of every hour (UTC). GitHub can delay scheduled runs, and multiple upstream commits between checks are combined into one snapshot of the latest commit. This is polling, not a guarantee of one snapshot per upstream commit. See [GitHub's schedule documentation](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule).

### Credential setup

Before the first publish:

1. In this repository's **Settings → Environments**, create `nexus-snapshots`.
2. Under **Deployment branches and tags**, choose **Selected branches and tags**, and add a **Branch** rule for `main`.
3. Add these **environment secrets**:
   - `NEXUS_USER`: your ASF Nexus username.
   - `NEXUS_PW`: your ASF Nexus password or supported Nexus credential, with permission to deploy Sedona snapshots.

A GitHub personal access token does not authenticate to ASF Nexus. The Nexus credentials stay in this repository; they are not needed in `apache/sedona`. Keep them as environment secrets so jobs must use the protected `nexus-snapshots` environment to access them. See [GitHub's environment setup guide](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments) and [ASF's Maven publishing guide](https://infra.apache.org/publishing-maven-artifacts.html).

The workflow runs only in `jiayuasu/sedona-sync-action` on `main`. It has no pull-request trigger, checks out upstream code without persisted GitHub credentials, and exposes Nexus credentials only to the deploy step. Missing credentials cause a clear failure before deployment.

### Manual run

After the workflow is merged into `main` and credentials are configured, select **Actions → Publish Sedona snapshots → Run workflow** with the `main` branch, or run:

```bash
gh workflow run publish-snapshot.yml --repo jiayuasu/sedona-sync-action --ref main
```

Manual runs use the same change check and skip an already-published commit.
