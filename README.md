# Gantry

*Gantry* is a tool to update docker swarm services, enhanced [Shepherd](https://github.com/containrrr/shepherd).

## Usage

We release *Gantry* as a container [image](https://hub.docker.com/r/shizunge/gantry). You can create a docker service and run it on a swarm manager node.

```
docker service create \
  --name gantry \
  --mode replicated-job \
  --constraint "node.role==manager" \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
  --env "GANTRY_NODE_NAME={{.Node.Hostname}}" \
  shizunge/gantry
```

Or with docker compose, see the [example](examples/docker-compose.yml).

You can also run *Gantry* as a script outside the container `source ./src/entrypoint.sh`. *Gantry* is written to work with `busybox ash`. It should also work with `bash`.

## Configurations

You can configure the most behaviors of *Gantry* via environment variables.

### Common ones

| Environment Variable  | Default |Description |
|-----------------------|---------|------------|
| GANTRY_LOG_LEVEL      | INFO | Control how many logs generated by *Gantry*. Valid values are `NONE`, `ERROR`, `WARN`, `INFO`, `DEBUG` (case sensitive). |
| GANTRY_NODE_NAME      |      | Add node name to logs. |
| GANTRY_SLEEP_SECONDS  | 0    | Sleep time between two updates. Set it to 0 to run *Gantry* once and then exit. |
| TZ                    |      | Set timezone for time in logs. |

### To login to registries

| Environment Variable  | Default | Description |
|-----------------------|---------|-------------|
| GANTRY_REGISTRY_CONFIG        | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_CONFIG_FILE   | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_CONFIGS_FILE  | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_HOST          | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_HOST_FILE     | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_PASSWORD      | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_PASSWORD_FILE | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_USER          | | See [Authentication](#authentication). |
| GANTRY_REGISTRY_USER_FILE     | | See [Authentication](#authentication). |

### To select services

| Environment Variable  | Default | Description |
|-----------------------|---------|-------------|
| GANTRY_SERVICES_EXCLUDED         | | A space separated list of services names that are excluded from updating. |
| GANTRY_SERVICES_EXCLUDED_FILTERS | | A space separated list of [filters](https://docs.docker.com/engine/reference/commandline/service_ls/#filter). Exclude services which match the given filters from updating. |
| GANTRY_SERVICES_FILTERS          | | A space separated list of [filters](https://docs.docker.com/engine/reference/commandline/service_ls/#filter) that are accepted by `docker service ls --filter` to select services to update. |
| GANTRY_SERVICES_SELF             | | To indicate whether a service is *Gantry* itself. *Gantry* will be the first service being updated. The manifest inspection will be always performed on the *Gantry* service to avoid an infinity loop of updating itself. |

### To check if new images are available

| Environment Variable  | Default | Description |
|-----------------------|---------|-------------|
| GANTRY_MANIFEST_INSPECT          | true  | Set to `true` to check manifest of the image. Set to `false` to skip checking the manifest. As a result of skipping, `docker service update` always runs. In case you add `--force` to `GANTRY_UPDATE_OPTIONS`, you also want to disable the inspection. |
| GANTRY_MANIFEST_OPTIONS          |       | [Options](https://docs.docker.com/engine/reference/commandline/buildx_imagetools_inspect/#options) added to the `docker buildx imagetools inspect` or [options](https://docs.docker.com/engine/reference/commandline/manifest_inspect/#options) to `docker manifest inspect`, depending on `GANTRY_MANIFEST_USE_MANIFEST_CMD` value. |
| GANTRY_MANIFEST_USE_MANIFEST_CMD | false |Set to `true` to run `docker manifest inspect`. Set to `false` to run `docker buildx imagetools inspect`. `docker manifest inspect` could [fail on some registries](https://github.com/orgs/community/discussions/45779). |

### To add options to services update

| Environment Variable  | Default | Description |
|-----------------------|---------|-------------|
| GANTRY_ROLLBACK_ON_FAILURE    | true  | Set to `true` to enable rollback when updating fails. Set to `false` to disable the rollback. |
| GANTRY_ROLLBACK_OPTIONS       |       | [Options](https://docs.docker.com/engine/reference/commandline/service_update/#options) added to the `docker service update --rollback` command. |
| GANTRY_UPDATE_JOBS            | false | Set to `true` to update replicated-job or global-job. Set to `false` to disable updating jobs. |
| GANTRY_UPDATE_OPTIONS         |       | [Options](https://docs.docker.com/engine/reference/commandline/service_update/#options) added to the `docker service update` command. |
| GANTRY_UPDATE_TIMEOUT_SECONDS | 300   | Error out if updating of a single service takes longer than the given time. |

### After updating

| Environment Variable  | Default | Description |
|-----------------------|---------|-------------|
| GANTRY_CLEANUP_IMAGES           | true  | Set to `true` to clean up the updated images. Set to `false` to disable the cleanup. Before cleaning up, *Gantry* will try to remove any *exited* and *dead* containers that are using the images. |
| GANTRY_NOTIFICATION_APPRISE_URL |       | Enable notifications on service update with [apprise](https://github.com/djmaze/apprise-microservice). |
| GANTRY_NOTIFICATION_TITLE       |       | Add an additional message to the notification title. |

## Authentication

If you only need to login to a single registry, you can use the environment variables  `GANTRY_REGISTRY_USER`, `GANTRY_REGISTRY_PASSWORD`, `GANTRY_REGISTRY_HOST` and `GANTRY_REGISTRY_CONFIG` to provide the authentication information. You may also use the `*_FILE` variants to pass the information through files. The files can be added to the service via [docker secret](https://docs.docker.com/engine/swarm/secrets/). `GANTRY_REGISTRY_HOST` and `GANTRY_REGISTRY_CONFIG` are optional. Use `GANTRY_REGISTRY_HOST` when you are not using Docker Hub. Use `GANTRY_REGISTRY_CONFIG` when you only want to enable authentication for selected services.

If the images of services are hosted on multiple registries that are required authentication, you should provide a configuration file to the *Gantry* and set `GANTRY_REGISTRY_CONFIGS_FILE` correspondingly. You can use [docker secret](https://docs.docker.com/engine/swarm/secrets/) to provision the configuration file. The configuration file must be in the following format:

* Each line should contain 4 columns, which are either `<TAB>` or `<SPACE>` separated. The columns are 
```
<config name> <host> <user> <password>
```
> * config name: an identifier for the account. This should be an acceptable [Docker config name](https://docs.docker.com/engine/swarm/configs/).
> * host: the registry to authenticate against, e.g. docker.io.
> * user: the user name to authenticate as.
> * password: the password to authenticate with.
* Lines starting with  `#` are comments.
* Empty lines, comment lines and invalid lines are ignored.

You need to tell *Gantry* to use a named config rather than the default one when updating a particular service. The named configurations are set via either `GANTRY_REGISTRY_CONFIG`, `GANTRY_REGISTRY_CONFIG_FILE` or `GANTRY_REGISTRY_CONFIGS_FILE`. This can be done by adding the following label to the service `gantry.auth.config=<config-name>`.

> NOTE: You also want to manually add `--with-registry-auth` to `GANTRY_UPDATE_OPTIONS` and `GANTRY_ROLLBACK_OPTIONS` when you enable authentication.

## FAQ

[FAQ](docs/faq.md)

[Migrate from *Shepherd*](docs/migration.md)

## Development

*Gantry* is written to work with `busybox ash` (v1.35+), thus it could run easily in an alpine-based container without additional packages installed. One exception is that the notification feature requires `curl`. *Gantry* is also tested in `bash`.

[shellcheck](https://github.com/koalaman/shellcheck) will run on push to enforce the best practices of writing shell scripts. Some checks are disabled thanks to `busybox ash` supports more features than POSIX `sh`. You can find the list of disabled checks in [.shellcheckrc](.shellcheckrc).

To run `shellcheck` locally:
```
pushd src
shellcheck *.sh
popd
pushd tests
shellcheck *.sh
popd
```

Majority of the configuration options are covered by end-to-end tests. It would be a good enhancement to generate coverage metrics.

You can also run the tests locally. Before starting a test, you need to prepare a container repository for the test images. And login to the registry via `docekr login` thus the tests can push images to the repository.

To run all the tests locally:
```
./tests/run_all_tests.sh <repository-name> [registry]
```

To run an individual test locally:
```
pushd tests
source ./lib-gantry-test.sh
source ./test_entrypoint.sh
<test-name> [registry/]<repository-name>
popd
```

## Contacts

If you have any problems or questions, please contact me through a [GitHub issue](https://github.com/shizunge/gantry/issues).
