# Sentry Release GitHub Action

**NOTE**: Currently only available for Linux runners. See [this issue](https://github.com/getsentry/action-release/issues/58) for more details.

Automatically create a Sentry release in a workflow.

A release is a version of your code that can be deployed to an environment. When you give Sentry information about your releases, you unlock a number of new features:

- Determine the issues and regressions introduced in a new release
- Predict which commit caused an issue and who is likely responsible
- Resolve issues by including the issue number in your commit message
- Receive email notifications when your code gets deployed

Additionally, releases are used for applying source maps to minified JavaScript to view original, untransformed source code. You can learn more about releases in the [releases documentation](https://docs.sentry.io/workflow/releases).

## Prerequisites

### Create a Sentry Internal Integration

NOTE: You have to be an admin in your Sentry org to create this.

For this action to communicate securely with Sentry, you'll need to create a new internal integration. In Sentry, navigate to: _Settings > Developer Settings > Custom Integrations > Create New Integration > Internal Integration_.

Give your new integration a name (for example, "GitHub Action Release Integration”) and specify the necessary permissions. In this case, we need Admin access for “Release” and Read access for “Organization”.

![View of internal integration permissions.](images/internal-integration-permissions.png)

Click “Save” at the bottom of the page, then go back into your newly created integration and click "New Token". Grab this newly generated token and use it as your `SENTRY_AUTH_TOKEN`. We recommend you store this as an [encrypted secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions).

## Usage

Adding the following to your workflow will create a new Sentry release and tell Sentry that you are deploying to the `production` environment.
  
```yaml
- uses: actions/checkout@v3
  with:
    fetch-depth: 0

- name: Create Sentry release
  uses: getsentry/action-release@v1
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
    SENTRY_ORG: ${{ secrets.SENTRY_ORG }}
    SENTRY_PROJECT: ${{ secrets.SENTRY_PROJECT }}
    # SENTRY_URL: https://sentry.io/
  with:
    environment: production
```

### Inputs

#### Environment Variables

|name|description|default|
|---|---|---|
|`SENTRY_AUTH_TOKEN`|**[Required]** Authentication token for Sentry. See [installation](#create-a-sentry-internal-integration).|-|
|`SENTRY_ORG`|**[Required]** The slug of the organization name in Sentry.|-|
|`SENTRY_PROJECT`|The slug of the project name in Sentry. One of `SENTRY_PROJECT` or `projects` is required.|-|
|`SENTRY_URL`|The URL used to connect to Sentry. (Only required for [Self-Hosted Sentry](https://develop.sentry.dev/self-hosted/))|`https://sentry.io/`|

#### Parameters

|name|description|default|
|---|---|---|
|`environment`|Set the environment for this release. E.g. "production" or "staging". Omit to skip adding deploy to release.|-|
|`finalize`|When false, omit marking the release as finalized and released.|`true`|
|`ignore_missing`|When the flag is set and the previous release commit was not found in the repository, will create a release with the default commits count instead of failing the command.|`false`|
|`ignore_empty`|When the flag is set, command will not fail and just exit silently if no new commits for a given release have been found.|`false`|
|`sourcemaps`|Space-separated list of paths to JavaScript sourcemaps. Omit to skip uploading sourcemaps.|-|
|`dist`|Unique identifier for the distribution, used to further segment your release. Usually your build number.|-|
|`started_at`|Unix timestamp of the release start date. Omit for current time.|-|
|`version`|Identifier that uniquely identifies the releases. _Note: the `refs/tags/` prefix is automatically stripped when `version` is `github.ref`._|<code>${{&nbsp;github.sha&nbsp;}}</code>|
|`version_prefix`|Value prepended to auto-generated version. For example "v".|-|
|`set_commits`|Specify whether to set commits for the release. Either "auto" or "skip".|"auto"|
|`projects`|Space-separated list of paths of projects. When omitted, falls back to the environment variable `SENTRY_PROJECT` to determine the project.|-|
|`url_prefix`|Adds a prefix to source map urls after stripping them.|-|
|`strip_common_prefix`|Will remove a common prefix from uploaded filenames. Useful for removing a path that is build-machine-specific.|`false`|
|`working_directory`|Directory to collect sentry release information from. Useful when collecting information from a non-standard checkout directory.|-|
|`disable_telemetry`|The action sends telemetry data and crash reports to Sentry. This helps us improve the action. You can turn this off by setting this flag.|`false`|

### Outputs

|name|description|examples|
|---|---|---|
|`version`|Identifier that uniquely identifies the releases.|`734713bc047d87bf7eac9674765ae793478c50d3`, if using the default <code>${{&nbsp;github.sha&nbsp;}}</code>|


### Examples

- Create a new Sentry release for the `production` environment and upload JavaScript source maps from the `./lib` directory.

    ```yaml
    - uses: getsentry/action-release@v1
      with:
        environment: 'production'
        sourcemaps: './lib'
    ```

- Create a new Sentry release for the `production` environment of your project at version `v1.0.1`.

    ```yaml
    - uses: getsentry/action-release@v1
      with:
        environment: 'production'
        version: 'v1.0.1'
    ```

- Create a new Sentry release for [Self-Hosted Sentry](https://develop.sentry.dev/self-hosted/)

    ```yaml
    - uses: getsentry/action-release@v1
      env:
        SENTRY_URL: https://sentry.example.com/
    ```

## Releases

The `build.yml` workflow will build a Docker image every time a pull request merges to `master` and upload it to [the GitHub registry](https://github.com/orgs/getsentry/packages?repo_name=action-release), thus, effectively being live for everyone even if we do not bump the version.

NOTE: Unfortunately, we only use the `latest` tag for the Docker image, thus, making use of a version with the action innefective (e.g. `v1` vs `v1.3.0`). See #129 on how to fix this.

NOTE: Right now, our Docker image publishing is decoupled from `tag` creation in the repository. We should only publish a specific Docker tag when we create a tag (you can make GitHub workflows listen to this). See #102 for details. Once this is fixed merges to `master` will not make the Docker image live and the following paragraph will be legit.

When you are ready to make a release, open a [new release checklist issue](https://github.com/getsentry/action-release/issues/new?assignees=&labels=&template=release-checklist.md&title=New+release+checklist+for+%5Bversion+number%5D) and follow the steps in there.

The Docker build is [multi-staged](https://github.com/getsentry/action-release/blob/master/Dockerfile) in order to make the final image used by the action as small as possible to reduce network transfer (use `docker images` to see the sizes of the images).

### End to end testing on GitHub's CI

The first job in `test.yml` has instructions on how to tweak a job in order to execute your changes as part of the PR.

NOTE: Contributors will need to create an internal integration in their Sentry org and need to be an admin. See `Prerequisites` section above.

Members of this repo will not have to set anything up since [the integration](https://sentry-ecosystem.sentry.io/settings/developer-settings/end-to-end-action-release-integration-416eb2/) is already set-up. Just open the PR and you will see [a release created](https://sentry-ecosystem.sentry.io/releases/?project=4505075304693760) for your PR.

## Development

If your change impacts the options used for the action, you need to update the README.md with the new options.

### Unit tests

You can run the unit tests with `yarn test`.

## Contributing

See the [Contributing Guide](https://github.com/getsentry/action-release/blob/master/CONTRIBUTING).

## License

See the [License File](https://github.com/getsentry/action-release/blob/master/LICENSE).

## Troubleshooting

Suggestions and issues can be posted on the repository's
[issues page](https://github.com/getsentry/action-release/issues).

- Forgetting to include the required environment variables
  (`SENTRY_AUTH_TOKEN`, `SENTRY_ORG`, and `SENTRY_PROJECT`), yields an error that looks like:

    ```text
    Environment variable SENTRY_ORG is missing an organization slug
    ```

- Building and running this action locally on an unsupported environment yields an error that looks like:

    ```text
    Syntax error: end of file unexpected (expecting ")")
    ```

- When adding the action, make sure to first checkout your repo with `actions/checkout@v3`.
Otherwise it could fail at the `propose-version` step with the message:

    ```text
    error: Could not automatically determine release name
    ```

- In `actions/checkout@v3` the default fetch depth is 1. If you're getting the error message:

    ```text
    error: Could not find the SHA of the previous release in the git history. Increase your git clone depth.
    ```

    you can fetch all history for all branches and tags by setting the `fetch-depth` to zero like so:

    ```text
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    ```
