# Installing the CircleCI Runner

## Supported platforms

At present, the supported platforms are:
* Ubuntu 18.04 running on an `x86_64` (`AMD64`) or `ARM64` architecture
* macOS Catalina (`10.15.6+`) or above, running on an `x86_64` architecture.

Running in a container is not supported at this time.

## Prerequisites

### Authentication

In order to complete this process you'll need to have selected a label for your custom [`resource_class`](#runnerresource_class) and been issued an authentication token for it.

The steps to follow are:

1. [Install](https://circleci.com/docs/2.0/local-cli/#installation) the `circleci` command-line tool.
2. Create a namespace for your organization's runner resources under:
   * Each organization can only create a single namespace.
   * If you already use orbs, this will be same namespace as the orbs use.
   * Use the following command: `circleci namespace create <name> <vcs-type> <org-name>`.
   * e.g. If your GitHub URL is https://github.com/circleci, then: `circleci namespace create my-namespace github circleci`.
3. Create a [`resource class`](#runnerresource_class) for your runner in the above namespace:
   * Use the following command: `circleci runner resource-class create <resource-class> <description>`.
   * e.g. `circleci runner resource-class create my-namespace/my-resource-class my-description`
4. Create a token for authenticating the above resource-class:
   * Use the following command: `circleci runner token create [--config] <resource-class> <nickname>`.
   * e.g. `circleci runner token create my-namespace/my-resource-class my-token`.
   * This will print a generated Runner config including the authentication token. Note that the token cannot be retrieved again, so store it safely.

### Installation Tooling

The installation process assumes that the following utilities are installed on the system:

* curl (installed by default on macOS)
* sha256sum (installed as part of `coreutils` on Linux, installed by default on macOS)
* systemd version `235+` (Linux only)

Installation also requires permissions to create a user, and create directories under `/opt`.

### Job Running Requirements

Running jobs requires that the following tools are available on the machine:

* tar
* gzip
* coreutils (Linux only)
* git (recommended, but not required)

## Installation

### Download the Launch Agent Binary and Verify the Checksum

The Launch Agent can be installed using the following script in a bash shell.

This will use `/opt/circleci` as the base install location.

First, set one of these variables as appropriate for for your installation target.

#### For Linux x86_64
```bash
platform=linux/amd64
```

#### For Linux ARM64
```bash
platform=linux/arm64
```

#### For macOS x86_64
```bash
platform=darwin/amd64
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
file="$(grep -F "$platform" checksums.txt | cut -d ' ' -f 2)"
file="${file:1}"
mkdir -p "$platform"
echo "Downloading CircleCI Launch Agent: $file"
curl --compressed -L "$base_url/$agent_version/$file" -o "$file"
echo "Verifying CircleCI Launch Agent download"
sha256sum --check --ignore-missing checksums.txt && chmod +x "$file"; sudo cp "$file" "$prefix/circleci-launch-agent" || echo "Invalid checksum for CircleCI Launch Agent, please try download again"
```

## Platform-specific instructions

Please refer to the platform-specific installation instructions:
* [linux](./INSTALL-linux.md)
* [macOS](./INSTALL-macos.md)

## Configuration file reference

A YAML file is used to configure the Launch Agent; how it communicates with our servers and how it will launch the Task Agent.

The configuration file uses the following format, the parameters are explained in more detail below:

```yaml
api:
  auth_token: AUTH_TOKEN
runner:
  name: RUNNER_NAME
```

### runner.name

This is a unique name assigned to this particular running Launch Agent, we recommend using the hostname of the machine. It will be used to identify the agent when viewing statuses and job results in the CircleCI UI.

### api.auth_token

This is a token used to identify the Launch Agent to CircleCI, it will be provided by your Customer Success Manager. An existing token may be shared among many installations, but only allows a particular `resource_class` to be specified.

### runner.command_prefix

This allows you to customize how the Task Agent process is launched, our example uses the `launch-task` script provided below.

### runner.working_directory

This allows you to control the default working directory used by each job.

If the directory already exists, Task Agent will need permissions to write to it.

If the directory does not exist, then the Task Agent will need permissions to create it.

If `%s` is present in the value, it will be replaced with a different value for each job. These directories will *not* be automatically removed.

### runner.cleanup_working_directory

This allows for control of working directory cleanup after each job.

The default value is `false`.
