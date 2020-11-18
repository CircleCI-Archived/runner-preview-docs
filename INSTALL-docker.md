# Installing the CircleCI Runner on a host with Docker

The host needs to have Docker installed. Once the `runner` container is started, it will immediately attempt to start running jobs.  The container will be reused to run more jobs indefinitely until it's stopped.

The number of containers running in parallel on the host is constrained by the host's available resources and your jobs' performance requirements.

## Create a Dockerfile that extends the CircleCI Runner image

Here we install python3 on top of the base image

`Dockerfile.runner.extended`

```
FROM circleci/runner:launch-agent
RUN apt-get update; \
    apt-get install --no-install-recommends -y \
        python3
```

## Build the Docker image

```bash
docker build --file ./Dockerfile.runner.extended .
```

## Start the Docker container

```bash
CIRCLECI_RESOURCE_CLASS=<resource-class> CIRCLECI_API_TOKEN=<runner-token> docker run --env CIRCLECI_API_TOKEN --env CIRCLECI_RESOURCE_CLASS --name <container-name> <container-id-from-previous-step>
```

When the container starts, it will immediately attempt to start running jobs.

## Stopping the Docker container

``` bash
docker stop <container-name>
```

## Additional Resources

- [CircleCI Runner Image on Docker Hub](https://github.com/CircleCI-Public/runner-preview-docs/)
- [CircleCI Runner Image on Github](https://github.com/CircleCI-Public/circleci-runner-docker)
- [CircleCI Docs](https://circleci.com/docs/) - The official CircleCI Documentation website.
- [Docker Docs](https://docs.docker.com/)
