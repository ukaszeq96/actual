# Actual Budget with XLSX import in Komodo

This build applies the XLSX import changes from [Actual Budget PR #9412](https://github.com/actualbudget/actual/pull/9412) to the release behind `ghcr.io/actualbudget/actual:latest`. It reads the version from the official image, fetches the matching source tag, and replaces the image's web bundle with the patched build. Open the web app served by this image to use XLSX import.

Put this directory on a branch that Komodo can clone. Create a [Komodo Build](https://komo.do/docs/build) with these settings:

| Field              | Value                                                    |
| ------------------ | -------------------------------------------------------- |
| Repository         | Your fork, e.g. `ukaszeq96/actual`                       |
| Branch             | The branch containing this directory                     |
| Build Path         | `komodo/actual-xlsx`                                     |
| Dockerfile Path    | `Dockerfile`                                             |
| Builder            | Your Komodo builder                                      |
| Image Registry     | A registry and account your builder can push to          |
| Image Name         | `actual-xlsx`                                            |
| Extra Args         | Add `--pull` so each run checks for a new upstream image |
| Include Latest Tag | Enabled                                                  |
| Image Tag          | Leave blank to publish `:latest` on your custom image    |

Run the Build. Set the image in your Actual Stack or Deployment to the `actual-xlsx:latest` image in **your registry**, and keep the existing `/data` mount and server configuration. To try the image before switching your stack, launch it with a separate data directory, open a demo budget, and choose **Import** in an account. Select one of the example workbooks under `packages/desktop-client/e2e/data/` on the PR branch and check the worksheet, header, and transaction preview.

Create a [Komodo Procedure](https://komo.do/docs/automate/procedures) to run the build on a schedule. This example assumes the Build resource is named `actual-xlsx`.

```toml
[[procedure]]
name = "refresh-actual-xlsx"
config.schedule = "Every day at 04:00"

[[procedure.config.stage]]
name = "Build Actual with XLSX import"
executions = [
  { execution.type = "RunBuild", execution.params.build = "actual-xlsx" },
]
```

Enable **Auto Update** on the [Stack](https://komo.do/docs/deploy/compose) or [Deployment](https://komo.do/docs/deploy/containers) if you want Komodo to redeploy when the custom image digest changes. The build stops if the patch no longer applies or the importer tests fail. In that case, the previous custom image remains available. Update `xlsx.patch` against the new upstream tag and rerun the build.

To build locally with Docker from the repository root:

```sh
docker build --pull -t actual-xlsx:latest komodo/actual-xlsx
```

To rebuild a specific release, set the `ACTUAL_BASE` build argument to that image tag or digest. If a future release changes the required Node version or needs newer translations, update those inputs in the Dockerfile.
