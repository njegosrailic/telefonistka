## Installation

The current version of the docs doesn't cover the details of running the Telefonistka instance beyond listing its [configuration options](#server-configuration) and noting that its `/webhook` endpoint needs to accessable from Github(Cloud or private instance)

The Github side of the configuration could be done via a creation of an GitHub Application(recommended) or by configuring a webhook + github service account permission for each relevant repo.

### GitHub Application

* Create the application.
  * Go to GitHub [apps page](https://github.com/settings/apps) under "Developer settings" and create a new app.
  * Set the Webhooks URL to point to your running Telefonistka instance(remember the `/webhook` URL path), use HTTPS and set `Webhook secret` (pass  to instace via `GITHUB_WEBHOOK_SECRET` env var)
  * Provide the new app with read&write `Repository permissions` for `Commit statuses`, `Contents`, `Issues` and `Pull requests`.
  * Subscribe to `Issues` and `Pull request` events
  * Generate a `Private key` and provide it you your instance with the  `GITHUB_APP_PRIVATE_KEY_PATH` env variable.
  * Grab the `App ID` provide it you your instance with the `GITHUB_APP_ID` env variable.
* For each relevant repo:
  * Add repo to application configuration.
  * Add `telefonistka.yaml` to repo root.

### Webhook + service account

* Create a github service account(basically a regualr account) and generate an API Token, provide token via `GITHUB_OAUTH_TOKEN` Env var
* For each relevant repo:
  * Add the Telefonistka API endpoints to the webhooks under the repo settings page.
  * Ensure the service account has the relevant permission on the repo.
  * Add `telefonistka.yaml` to repo root.

## Server Configuration

Environment variables for the webhook process:

`APPROVER_GITHUB_OAUTH_TOKEN` GitHub oAuth token for automatically approving promotion PRs

`GITHUB_OAUTH_TOKEN` GitHub main oAuth token for all other GH operations

`GITHUB_HOST` URL for github API, needed for Github Enterprise Server, should include http scheme but no `/api/v3` path, e.g. :`https://my-gh-host.com/`

`GITHUB_WEBHOOK_SECRET` secret used to sign webhook payload to be validated by the WH server, must match the sting in repo settings/hooks page

`GITHUB_APP_PRIVATE_KEY_PATH`  Private key for Github applications style of deployments, in PEM format

`GITHUB_APP_ID` Application ID for Github applications style of deployments, available in the Github Application setting page.

Behavior of the bot is configured by YAML files **in the target repo**:

## Repo Configuration

Pulled from `telefonistka.yaml` file in the repo root directory(default branch)

Configuration keys:  

|key|desc|
|---|---|
|`promotionPaths`| Array of maps, each map describes a promotion flow|  
|`promotionPaths[0].sourcePath`| directory that holds components(subdirectories) to be synced, can include a regex.|
|`promotionPaths[0].conditions` | conditions for triggering a specific promotion flows. Flows are evatluated in order, first one to match is triggered.|
|`promotionPaths[0].conditions.prHasLabels` | Array of PR labels, if the triggering PR has any of these lables the condition is considered fulfilled. Currently it's the only supported condition type|
|`promotionPaths[0].targetPaths`|  Array of arrays(!!!) of target paths tied to the source path mentioned above, each top level element represent a PR that will be opened, so multiple target can be synced in a single PR|  
|`dryRunMode`| if true, the bot will just comment the planned promotion on the merged PR|
|`autoApprovePromotionPrs`| if true the bot will auto-approve all promotion PRs, with the assumption the original PR was peer reviewed and is promoted verbatim. Required additional GH token via APPROVER_GITHUB_OAUTH_TOKEN env variable|
|`toggleCommitStatus`| Map of strings, allow (non-repo-admin) users to change the [Github commit status](https://docs.github.com/en/rest/commits/statuses) state(from failure to success and back). This can be used to continue promotion of a change that doesn't pass repo checks. the keys are strings commented in the PRs, values are [Github commit status context](https://docs.github.com/en/rest/commits/statuses?apiVersion=2022-11-28#create-a-commit-status) to be overridden|

Example:

```yaml
promotionPaths:
  - sourcePath: "workspace/"
    targetPaths:
      - 
        - "clusters/dev/us-east4/c2"
        - "clusters/lab/europe-west4/c1"
        - "clusters/staging/us-central1/c1"
        - "clusters/staging/us-central1/c2"
        - "clusters/staging/europe-west4/c1"
  - sourcePath: "clusters/staging/[^/]*/[^/]*" # This will start a promotion to prod from any "staging" path
    conditions:
      prHasLabels:
        - "quick_promotion" # This flow will run only if PR has "quick_promotion" label, see targetPaths below
    targetPaths:
      -
        - "clusters/prod/us-west1/c2" # First PR for only a single cluster
      -
        - "clusters/prod/europe-west3/c2" # 2nd PR will sync all 4 remaining clusters
        - "clusters/prod/europe-west4/c2"
        - "clusters/prod/us-central1/c2"
        - "clusters/prod/us-east4/c2"
  - sourcePath: "clusters/staging/[^/]*/[^/]*" # This flow will run on PR without "quick_promotion" label
    targetPaths:
      -
        - "clusters/prod/us-west1/c2" # Each cluster will have its own promotion PR
      -
        - "clusters/prod/europe-west3/c2"
      -
        - "clusters/prod/europe-west4/c2"
      -
        - "clusters/prod/us-central1/c2"
      -
        - "clusters/prod/us-east4/c2"
dryRunMode: true
autoApprovePromotionPrs: true
toggleCommitStatus:
  override-terrafrom-pipeline: "github-action-terraform"
```

## Component Configuration

This optional in-component configuration file allows overriding the general promotion configuration for a specific component.  
File location is `COMPONENT_PATH/telefonistka.yaml` (no leading dot in file name), so it could be:  
`workspace/reloader/telefonistka.yaml` or `env/prod/us-central1/c2/wf-kube-proxy-metrics-proxy/telefonistka.yaml`  
it includes only two optional configuration keys, `promotionTargetBlockList` and `promotionTargetAllowList`.  
Both are matched against the target component path using Golang regex engine.

If a target path matches an entry in `promotionTargetBlockList` it will not be promoted(regardless of `promotionTargetAllowList`).

If  `promotionTargetAllowList` exist(non empty), only target paths that matches it will be promoted to(but the previous statement about `promotionTargetBlockList` still applies).

```yaml
promotionTargetBlockList:
  - env/staging/europe-west4/c1.*
  - env/prod/us-central1/c3/
promotionTargetAllowList:
  - env/prod/.*
  - env/(dev|lab)/.*
```