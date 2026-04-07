# MuleSoft CloudHub 2.0 CI/CD Setup

This document captures the clean setup for this project's GitHub Actions CI/CD pipeline based on the final working configuration in this repository.

## Goal

Build a Mule 4 application in GitHub Actions, publish the artifact to Anypoint Exchange, and deploy it to CloudHub 2.0 using a Connected App.

## Final Pipeline Behavior

The pipeline currently:

1. Triggers only when a pull request into `main` is merged.
2. Builds the Mule application with Maven.
3. Computes the first 5 characters of the commit SHA during the workflow.
4. Skips MUnit tests in CI.
5. Publishes the generated artifact to Anypoint Exchange.
6. Deploys the artifact to CloudHub 2.0.
7. Sends the short commit SHA to CloudHub as an application property for traceability.
8. Sends Anypoint platform autodiscovery properties required by API Manager tracking.

## Why MUnit Is Skipped

MUnit for Mule EE applications requires access to MuleSoft enterprise runtime artifacts. In this project, CI failed during MUnit startup because the GitHub runner did not have access to the required EE repository artifacts such as `com.mulesoft.licm:licm`.

Because this project is currently running on a trial-style setup without enterprise repository access, the CI pipeline skips MUnit using `-DskipMunitTests`.

If enterprise repository access is later enabled, MUnit can be reintroduced.

## Prerequisites

Before setting up the pipeline, make sure the following are available:

1. A MuleSoft Connected App with permissions to:
   - publish assets to Exchange
   - deploy applications to Runtime Manager / CloudHub 2.0
2. A CloudHub 2.0 target that exists in your Anypoint environment.
3. The correct CloudHub 2.0 target for your account.

For this repository and trial account, the valid target is:

- `Cloudhub-US-East-2`

This matches the visible shared space region in Runtime Manager:

- `US East (Ohio)`

## GitHub Secrets

Create these repository secrets in GitHub:

1. `ANYPOINT_CLIENT_ID`
2. `ANYPOINT_CLIENT_SECRET`
3. `ANYPOINT_ENV_NAME`
4. `ANYPOINT_ORG_ID`
5. `ANYPOINT_PLATFORM_CLIENT_ID`
6. `ANYPOINT_PLATFORM_CLIENT_SECRET`
7. `MULE_SECURE_KEY`

## Optional GitHub Variables

These variables are optional because the workflow already provides defaults:

1. `ANYPOINT_BUSINESS_GROUP_ID`
2. `CLOUDHUB_TARGET`
3. `CLOUDHUB_APP_NAME`
4. `CLOUDHUB_REPLICAS`
5. `CLOUDHUB_VCORES`

Recommended values for this repo:

1. `CLOUDHUB_TARGET=Cloudhub-US-East-2`
2. `CLOUDHUB_APP_NAME=sapi-salesforce-implementation`
3. `CLOUDHUB_REPLICAS=1`
4. `CLOUDHUB_VCORES=0.1`

## Workflow File

The workflow file is:

- `.github/workflows/deploy-cloudhub.yml`

The workflow is triggered only on merged PRs:

```yaml
on:
  pull_request:
    branches:
      - main
    types:
      - closed
```

The build job runs only when the PR was actually merged:

```yaml
if: github.event.pull_request.merged == true
```

## Maven Settings in CI

The workflow generates a temporary Maven `settings.xml` during the run so Maven can authenticate to Anypoint Exchange.

It uses:

- username: `~~~Client~~~`
- password: `${ANYPOINT_CLIENT_ID}~?~${ANYPOINT_CLIENT_SECRET}`

No committed `.maven` folder is required for the current pipeline.

## Commit Traceability

CloudHub 2.0 does not reliably preserve the uploaded JAR file name in the Runtime Manager UI. Even if the workflow uploads a renamed file, the UI can still display the canonical application filename.

Because of that, the pipeline now uses a more reliable approach:

1. package the normal Mule JAR
2. compute the first 5 characters of the commit SHA
3. send that value to CloudHub as an application property

The property name is:

- `commit.shortSha`

Example value:

- `3e845`

This makes it easier to trace a deployment back to the commit that produced it without depending on the CloudHub UI file label.

## pom.xml Configuration

The main settings in `pom.xml` are:

1. `mule-maven-plugin` version `4.8.0`
2. `app.runtime` set to `4.11.0`
3. `cloudhub.target` set to `Cloudhub-US-East-2`
4. `releaseChannel` set to `EDGE`
5. `javaVersion` set to `17`
6. Connected App authentication under `cloudhub2Deployment`
7. `env=dev` passed as a CloudHub application property
8. `commit.shortSha` passed as a CloudHub application property
9. `anypoint.platform.base_uri` passed as a CloudHub application property
10. `anypoint.platform.config.analytics.agent.enabled` passed as a CloudHub application property
11. `anypoint.platform.analytics_base_url` passed as a CloudHub application property
12. `MULE_SECURE_KEY` passed as a CloudHub secure property
13. `anypoint.platform.client_id` and `anypoint.platform.client_secret` passed as CloudHub secure properties

Important note:

- `region` is not a valid `cloudhub2Deployment` parameter for Mule Maven Plugin `4.8.0`
- deployment region is effectively determined by the selected CloudHub target and Anypoint-side account configuration

## Secure Property Configuration

`MULE_SECURE_KEY` is configured in two places:

1. In `pom.xml` under `secureProperties`
2. In `mule-artifact.json` under `secureProperties`

The same secure-property pattern is also used for:

1. `anypoint.platform.client_id`
2. `anypoint.platform.client_secret`

This ensures the decryption key and autodiscovery client credentials are treated as secure properties in CloudHub.

## Application Property for `env`

The deployment is configured to send the following CloudHub application property:

- `env=dev`

This is defined in `pom.xml` under the CloudHub 2.0 deployment configuration:

```xml
<properties>
  <env>${app.env}</env>
</properties>
```

with:

```xml
<app.env>dev</app.env>
```

Important note:

- `global.xml` in this project still hardcodes `env` to `local`
- the deploy property is still sent to CloudHub so it is visible in the application properties
- if you later want runtime profile switching to be fully driven by deployment, `global.xml` would need to stop hardcoding `env=local`

## Application Property for Commit SHA

The deployment is also configured to send the short Git commit SHA as a CloudHub application property:

- `commit.shortSha`

This is defined in `pom.xml` under the CloudHub 2.0 deployment configuration:

```xml
<properties>
  <env>${app.env}</env>
  <commit.shortSha>${app.commit.shortSha}</commit.shortSha>
</properties>
```

with:

```xml
<app.commit.shortSha>local</app.commit.shortSha>
```

During CI/CD deployment, the workflow overrides this with:

```bash
-Dapp.commit.shortSha="${SHORT_SHA}"
```

The `local` default simply keeps the Maven configuration valid for local packaging when no CI commit SHA is being passed.

## Autodiscovery Platform Properties

To support API Manager autodiscovery and tracking, the deployment also sends these Anypoint platform properties:

Application property:

- `anypoint.platform.base_uri=https://anypoint.mulesoft.com/`
- `anypoint.platform.config.analytics.agent.enabled=true`
- `anypoint.platform.analytics_base_url=https://analytics-ingest.anypoint.mulesoft.com/`

Secure properties:

- `anypoint.platform.client_id`
- `anypoint.platform.client_secret`

This is defined in `pom.xml` under the CloudHub 2.0 deployment configuration:

```xml
<properties>
  <env>${app.env}</env>
  <commit.shortSha>${app.commit.shortSha}</commit.shortSha>
  <anypoint.platform.base_uri>${anypoint.platform.base_uri}</anypoint.platform.base_uri>
  <anypoint.platform.config.analytics.agent.enabled>${anypoint.platform.config.analytics.agent.enabled}</anypoint.platform.config.analytics.agent.enabled>
  <anypoint.platform.analytics_base_url>${anypoint.platform.analytics_base_url}</anypoint.platform.analytics_base_url>
</properties>

<secureProperties>
  <MULE_SECURE_KEY>${MULE_SECURE_KEY}</MULE_SECURE_KEY>
  <anypoint.platform.client_id>${anypoint.platform.client_id}</anypoint.platform.client_id>
  <anypoint.platform.client_secret>${anypoint.platform.client_secret}</anypoint.platform.client_secret>
</secureProperties>
```

During CI/CD deployment, the workflow passes:

```bash
-Danypoint.platform.client_id="${ANYPOINT_PLATFORM_CLIENT_ID}"
-Danypoint.platform.client_secret="${ANYPOINT_PLATFORM_CLIENT_SECRET}"
```

These values should come from GitHub Actions secrets.

## Current Versioning Strategy

The project version is currently:

- `1.0.1-SNAPSHOT`

This is important for Exchange publishing.

### Why SNAPSHOT is used

Exchange does not auto-increment versions for you.

If you publish a fixed version like `1.0.0` and that version already exists in a published state, Exchange rejects the publish with an error similar to:

- `An asset already exists with this version and published lifecycle state.`

Using a `-SNAPSHOT` version allows repeated CI publishes during ongoing development.

### Recommended versioning workflow

1. Work on `1.0.1-SNAPSHOT`
2. Let CI publish and deploy repeatedly
3. When you want a real release, change to `1.0.1`
4. After that, move to `1.0.2-SNAPSHOT`

## Build and Deploy Commands Used by CI

### Package

```bash
mvn --batch-mode --errors clean package \
  --settings "${HOME}/.m2/settings.xml" \
  -DskipMunitTests \
  -Denv=dev \
  -DMULE_SECURE_KEY="${MULE_SECURE_KEY}" \
  -Danypoint.org.id="${ANYPOINT_ORG_ID}"
```

### Publish to Exchange

```bash
mvn --batch-mode --errors deploy \
  --settings "${HOME}/.m2/settings.xml" \
  -DskipMunitTests \
  -Dmule.artifact="$(ls target/*.jar | head -1)" \
  -Denv=dev \
  -DMULE_SECURE_KEY="${MULE_SECURE_KEY}" \
  -Danypoint.org.id="${ANYPOINT_ORG_ID}"
```

### Deploy to CloudHub 2.0

```bash
mvn --batch-mode --errors mule:deploy \
  --settings "${HOME}/.m2/settings.xml" \
  -DskipMunitTests \
  -Dmule.artifact="$(ls target/*.jar | head -1)" \
  -Denv=dev \
  -Dapp.commit.shortSha="${SHORT_SHA}" \
  -Danypoint.platform.client_id="${ANYPOINT_PLATFORM_CLIENT_ID}" \
  -Danypoint.platform.client_secret="${ANYPOINT_PLATFORM_CLIENT_SECRET}" \
  -DMULE_SECURE_KEY="${MULE_SECURE_KEY}" \
  -DconnectedAppClientId="${ANYPOINT_CLIENT_ID}" \
  -DconnectedAppClientSecret="${ANYPOINT_CLIENT_SECRET}" \
  -Danypoint.org.id="${ANYPOINT_ORG_ID}" \
  -Danypoint.business.group.id="${CLOUDHUB_BUSINESS_GROUP_ID}" \
  -Denvironment="${ANYPOINT_ENV_NAME}" \
  -Dcloudhub.target="${CLOUDHUB_TARGET}" \
  -Dapp.name="${CLOUDHUB_APP_NAME}" \
  -Dcloudhub.replicas="${CLOUDHUB_REPLICAS}" \
  -Dcloudhub.vcores="${CLOUDHUB_VCORES}"
```

## From-Scratch Setup Checklist

Use this checklist if you want to rebuild the pipeline from scratch:

1. Create the Connected App in Anypoint Platform.
2. Grant it Exchange and Runtime Manager permissions required for publish and deploy.
3. Add the required GitHub repository secrets.
4. Set the CloudHub target to the actual region available in your account.
5. Keep the application version on a `-SNAPSHOT` during active CI/CD development.
6. Configure `mule-maven-plugin` for:
   - Connected App auth
   - CloudHub 2.0 target
   - application properties such as `env=dev`
   - Java 17
   - release channel
   - secure properties
7. Add `MULE_SECURE_KEY` to `mule-artifact.json` as a secure property.
8. Use a workflow that:
   - builds the artifact
   - computes a short commit SHA for traceability
   - uploads the built JAR
   - publishes to Exchange
   - deploys to CloudHub 2.0
   - passes `commit.shortSha` as an app property during deploy
   - passes Anypoint autodiscovery platform properties during deploy
9. Skip MUnit in CI unless enterprise repository access is available.

## Common Failures and Fixes

### 1. MUnit fails in CI with enterprise dependency errors

Symptoms:

- `Cannot create embedded container`
- `com.mulesoft.licm:licm`
- `401 Unauthorized`

Meaning:

- the CI runner does not have access to MuleSoft EE runtime artifacts

Fix:

- skip MUnit in CI or obtain enterprise repository access

### 2. Exchange publish fails because version already exists

Symptoms:

- `An asset already exists with this version and published lifecycle state.`

Meaning:

- a fixed version like `1.0.0` was already published

Fix:

- use a new release version or move back to a `-SNAPSHOT` version

### 3. CloudHub deploy fails because target region does not match account region

Symptoms:

- target in one region
- default region in another region

Fix:

- use the CloudHub target that actually exists in your account
- for this trial account, use `Cloudhub-US-East-2`

## Files Involved

1. `.github/workflows/deploy-cloudhub.yml`
2. `pom.xml`
3. `mule-artifact.json`
4. `src/main/resources/log4j2.xml`

## Final Notes

This setup is optimized for:

1. trial or sandbox-style CloudHub 2.0 deployments
2. Connected App authentication
3. repeatable CI publishes using snapshot versions
4. secure handling of `MULE_SECURE_KEY`

If enterprise repository access is later enabled, the next improvement would be reintroducing MUnit into the pipeline.
