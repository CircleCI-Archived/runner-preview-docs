## Installing the CircleCI Runner

Currently only Ubuntu 18.04 running on an AMD64 or ARM64 architecture is officially supported. Running in a container is not supported at this time.

### Prerequisites

#### Authentication

In order to complete this process you'll need to have selected a label for your custom [`resource_class`](#runnerresource_class) and been issued an authentication token for it.

The steps to follow are:

1. [Install](https://circleci.com/docs/2.0/local-cli/#installation) the `circleci` command-line tool.
2. Create a namespace for your organization's runner resources under:
   * Each organization can only create a single namespace.
   * If you already use orbs, this will be same namespace as the orbs use.
   * Use the following command: `circleci namespace create <name> <vcs-type> <org-name>`.
   * e.g.: `circleci namespace create my-namespace github circleci`.
3. Create a [`resource class`](#runnerresource_class) for your runner in the above namespace:
   * Use the following command: `circleci runner resource-class create <resource-class> <description>`.
   * e.g. `circleci runner resource-class create my-namespace/my-resource-class my-description`
4. Create a token for authenticating the above resource-class:
   * Use the following command: `circleci runner token create [--config] <resource-class> <nickname>`.
   * e.g. `circleci runner token create my-namespace/my-resource-class my-token`.
   * This will print the authentication token. Note that the token cannot be retrieved again, so store it safely.
   * It is also possible to generate a runner configuration with the above command, using the `--config` switch.
   * e.g. `circleci runner token create --config my-namespace/my-resource-class my-token > launch-agent-config.yaml`.

#### Installation Tooling

The installation process assumes that the following utilities are installed on the system:

* curl
* sha256sum (installed as part of `coreutils`)
* systemd

Installation also requires permissions to create a user, and create directories under `/opt`.

#### Job Running Requirements

Running jobs requires that the following tools are available on the machine:

* tar
* gzip
* coreutils

### Download the Launch Agent Binary and Verify the Checksum

The Launch Agent can be installed using the following script in a bash shell.

This will use `/opt/circleci` as the base install location.

First, set one of these variables as appropriate for for your installation target.

```bash
# For Linux x86_64
platform=linux/amd64
# For Linux ARM
platform=linux/arm64
```

And then run the following steps to download, verify and install the binary.

```bash
prefix=/opt/circleci
sudo mkdir -p "$prefix/workdir"
base_url="https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent"
echo "Determining latest version of CircleCI Launch Agent"
agent_version=$(curl "$base_url/release.txt")
echo "Using CircleCI Launch Agent version $agent_version"
echo "Downloading and verifying CircleCI Launch Agent Binary"
curl -sSL "$base_url/$agent_version/checksums.txt" -o checksums.txt
IFS=" " read -r -a selected <<< "$(grep -F "$platform" checksums.txt)"
checksum=${selected[0]}
file=${selected[1]:1}
mkdir -p "$platform"
echo "Downloading CircleCI Launch Agent: $file"
curl --compressed -L "$base_url/$agent_version/$file" -o "$file"
echo "Verifying CircleCI Launch Agent download"
sha256sum --check --ignore-missing checksums.txt && chmod +x "$file"; sudo mv "$file" "$prefix/circleci-launch-agent" || echo "Invalid checksum for CircleCI Launch Agent, please try download again"
```

### Configure Launch Agent

A YAML file is used to configure the Launch Agent; how it communicates with our servers and how it will launch the Task Agent.

The configuration file uses the following format, the parameters are explained in more detail below:

```yaml
api:
  auth_token: AUTH_TOKEN
runner:
  name: RUNNER_NAME
  resource_class: NAMESPACE/RESOURCE_CLASS
  command_prefix: ["/opt/circleci/launch-task"]
  working_directory: /opt/circleci/workdir/%s
  cleanup_working_directory: true
```

Once created save the configuration file to `/opt/circleci/launch-agent-config.yaml` owned by `root` with permissions `600`.

```bash
sudo chown root: /opt/circleci/launch-agent-config.yaml
sudo chmod 600 /opt/circleci/launch-agent-config.yaml
```

#### runner.name

This is a unique name assigned to this particular running Launch Agent, we recommend using the hostname of the machine. It will be used to identify the agent when viewing statuses and job results in the CircleCI UI.

#### api.auth_token

This is a token used to identify the Launch Agent to CircleCI, it will be provided by your Customer Success Manager. An existing token may be shared among many installations, but only allows a particular `resource_class` to be specified.

#### runner.resource_class

The `resource_class` field is used to uniquely identify the type of resource a job needs. It takes the format of `{{ namespace }}/{{ identifier }}`.

The namespace identifies the organization that owns the resource, while the identifier identifies the specific type of resource. For example, a medium sized docker resource on CircleCI would use `circleci/docker-medium`. The valid characters for the identifier are letters, numbers, `-` and `_`.

Along with the token, this value will be arranged through your Customer Success Manager.

#### runner.command_prefix

This allows you to customize how the Task Agent process is launched, our example uses the `launch-task` script provided below.

#### runner.working_directory

This allows you to control the default working directory used by each job.

If the directory already exists, Task Agent will need permissions to write to it.

If the directory does not exist, then the Task Agent will need permissions to create it.

If `%s` is present in the value, it will be replaced with a different value for each job. These directories will *not* be automatically removed.

#### runner.cleanup_working_directory

This allows for control of working directory cleanup after each job.

### Create the circleci user & working directory

These will be used when executing `task-agent`.

```bash
id -u circleci &>/dev/null || adduser --uid 1500 --disabled-password --gecos GECOS circleci

mkdir -p /opt/circleci/workdir
chown -R circleci /opt/circleci/workdir
```

### Create the Launch Script

This wrapper script will be used by Launch Agent to execute the Task Agent, while ensuring appropriate sandboxing and a clean shutdown.

Create `/opt/circleci/launch-task` owned by `root` with permissions `755`

```bash
#!/bin/bash

set -euo pipefail

## This script launches the build-agent using systemd-run in order to create a
## cgroup which will capture all child processes so they're cleaned up correctly
## on exit.

# The user to run the build-agent as - must be numeric
USER_ID=$(id -u circleci)

# Give the transient systemd unit an inteligible name
unit="circleci-$CIRCLECI_LAUNCH_ID"

# When this process exits, tell the systemd unit to shut down
abort() {
  systemctl stop "$unit"
}
trap abort EXIT

systemd-run \
    --pipe --collect --quiet --wait \
    --uid "$USER_ID" --unit "$unit" -- "$@"
```

### Create the Stop Script

This script will be used by systemd to perform an orderly shutdown of the Launch agent. It will first request that launch agent stops accepting new tasks by sending a `SIGINT` signal, and then it will follow up with a `SIGTERM` to abort the current task if it is still going for too longer.

The wait times in the environment variables should be used to tune how long you wish to wait for shutdown - the `DRAIN_TIMEOUT` should be set slightly longer than your jobs normally take if you want to avoid aborting any jobs early.

Create `/opt/circleci/stop-agent` owned by `root` with permissions `755`

```bash
#!/bin/bash

set -uo pipefail

## This script performs an orderly shutdown of the agents

# How long to wait for draining to complete
DRAIN_TIMEOUT=5m

# How long to wait for cancellation to complete
CANCEL_TIMEOUT=1m

# First send a SIGINT, this tells the launch-agent to stop accepting new tasks
kill -s SIGINT $MAINPID
timeout $DRAIN_TIMEOUT tail --pid=$MAINPID -f /dev/null

# If the process is still running, then SIGTERM to cancel the running task
if [ $? -eq 124 ]; then
    kill -SIGTERM $MAINPID
    timeout $CANCEL_TIMEOUT tail --pid=$MAINPID -f /dev/null
fi

# If the process is _still_ running at this point, we'll leave systemd to
# perform a SIGKILL and forcibly shut down the process
```

### Starting Launch Agent with Systemd

Create `/opt/circleci/circleci.service` owned by `root` with permissions `755`.

You must ensure that `TimeoutStopSec` is greater than the total amount of time the `stop-agent` script will take.

If you want to configure the CircleCI runner installation to start on boot, it is imporant to note that the Launch Agent will attempt to consume and start jobs as soon as it starts, so it should be configured appropriately before starting. The Launch Agent may be configured as a service and be managed by systemd with the following scripts:

```
[Unit]
Description=CircleCI Runner
After=network.target
[Service]
ExecStart=/opt/circleci/circleci-launch-agent --config /opt/circleci/launch-agent-config.yaml
Restart=always
User=root
NotifyAccess=exec
TimeoutStopSec=600
ExecStop=/opt/circleci/stop-agent
[Install]
WantedBy = multi-user.target
```

You can now enable and start the service

```bash
prefix=/opt/circleci
systemctl enable $prefix/circleci.service
```

#### Start the CircleCI Runner Service

When the CircleCI runner Service starts, it will immediately attempt to start running jobs, so it should be fully configured before the first start of the service.

```bash
systemctl start circleci.service
```

### Verify the Service is Running

The system reports a very basic health status through the `Status` field in `systemctl`.
This will report **Healthy** or **Unhealthy** based on connectivity to the CircleCI APIs.

You can see the status of the agent by running:

```bash
systemctl status circleci.service --no-pager
```

Which should produce output similar to:

```
● circleci.service - CircleCI Runner
   Loaded: loaded (/opt/circleci/circleci.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2020-05-29 14:33:31 UTC; 18min ago
 Main PID: 5592 (circleci-launch)
   Status: "Healthy"
    Tasks: 8 (limit: 2287)
   CGroup: /system.slice/circleci.service
           └─5592 /opt/circleci/circleci-launch-agent --config /opt/circleci/launch-agent-config.yaml
```

You can also see the logs for the system by running:

```bash
journalctl -u circleci
```
