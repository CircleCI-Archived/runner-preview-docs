# CircleCI runner preview docs
Run CircleCI jobs on your own infrastructure.

**DISCLAIMER**
The CircleCI runner is currently in a closed preview. We expect to make changes to the runner and its documentation throughout the preview period based on customer feedback. We will aim to maintain backwards compatibility but do not guarantee it during the preview stage.

The purpose of this repository is to provide documentation and feedback tracking during the preview period.

### In this document:
- [What is the CircleCI runner?](#what-is-the-circleci-runner-)
- [Prerequisites for the runner preview](#prerequisites-for-the-runner-preview)
- [Why use the runner?](#why-use-the-runner-)
- [How the runner works](#how-the-runner-works)
- [Limitations](#limitations)
- [How do I get access?](#how-do-i-get-access-)
- [How to set up the runner](#how-to-set-up-the-runner)
- [Leave us feedback!](#leave-us-feedback-)
- [FAQs](#faqs)
  * [What is the pricing for the CircleCI runner?](#what-is-the-pricing-for-the-circleci-runner-)
  * [What is the security model for the runner?](#what-is-the-security-model-for-the-runner-)
  * [How do I install dependencies needed for my jobs?](#how-do-i-install-dependencies-needed-for-my-jobs-)
  * [How do caching and workspaces and artifacts work with the runner?](#how-do-caching-and-workspaces-and-artifacts-work-with-the-runner-)


## What is the CircleCI runner?
The CircleCI runner allows you to use your own infrastructure for running jobs. This means you’ll be able to  build and test on a wider variety of architectures and environments, as well as giving you additional control over the security of the environment.

## Prerequisites for the runner preview
To take part in the CircleCI runner preview, you must:
1. Be on a CircleCI Custom (annual) plan with Gold or Platinum premium support
1. Be using CircleCI cloud (circleci.com)
1. Request preview access from your CircleCI account team
1. Have experience in managing infrastructure
1. Be willing to provide feedback throughout experience with using the runner to help us improve it.

## Why use the runner?
There are two key use cases we are aiming to meet with the runner:
- **Security requirements** - We understand that some customers that require running jobs on on-premises or limited-access infrastructure due to strict security requirements. Some things that the runner enables are:
  - **IP restrictions** - Runners can have static IP addresses that you can control.
  - **IAM permissions** - If you set up runners in AWS, they can be assigned IAM permissions.
  - **Monitor the operating system**
  - **Connect to private networks**
- **Unique compute requirements** - Customers who need to run jobs on an environment or architecture that CircleCI doesn’t offer as a resource class can use the runner to fill that need.

## How the runner works
Once installed, the runner polls circleci.com for work, runs jobs, and returns status, logs, and artifacts to CircleCI.

The runner consists of two components: the Launch Agent and the Task Agent

 - *Launch Agent* (`launch-agent`) - manages gathering the information required to run a task. It also downloads and launches a Task Agent process
 - *Task Agent* (`task-agent`) - handles running a task retrieved and configured by the Launch Agent.

![CircleCI runner model](/src/img/runner-model.png)

The system has been written to allow agent administrators to configure the `task-agent` to run with a lower level of privileges than the `launch-agent` - as any user who is able to execute a job will be able to gain the same privileges as `task-agent`. The instructions below are for our recommended deployment which follows this approach - Launch Agent will run as `root`, but Task Agent will run as `circleci`.

## Limitations
There are a few limitations to be aware of while using the runner preview:

- Only the following platforms are currently supported (we plan to add support for additional platforms soon. If you require support for another platform, please reach out to your account team):
  - Ubuntu 16.04 / 18.04 / 20.04
  - x86 architecture
- Currently these features of CircleCI are not yet implemented:
  - SSH access to jobs
  - Test splitting
- For open source repositories **only**:
  - The Enable Fork PRs setting currently allows any GitHub user to run a build when submitting a PR. Therefore any GitHub user will be able to run a job on the customer-hosted CircleCI runner resource. This can be a security risk, and the customer is responsible for securing their CircleCI runner machine or installation

## How do I get access?
If you are currently on a CircleCI Custom plan, you can reach out to your account team to request access to the runner preview.

## How to set up the runner
If you’re ready to get started, see our [runner installation docs](https://circleci-binary-releases.s3.amazonaws.com/circleci-launch-agent/docs/install.html).

## Leave us feedback!
We encourage you to leave us feedback on the runner preview by [opening an issue](https://github.com/CircleCI-Public/runner-preview-docs/issues/new) on this repository or by reaching out to your CircleCI account team.

---

## FAQs
Frequently asked questions for the CircleCI runner preview

### What is the pricing for the CircleCI runner?
Currently, we don’t charge for runner usage, but we will begin to charge for it soon. If you are a preview customer, your account team will keep you informed of any pricing changes for the runner.

### What is the security model for the runner?
When installing the runner you will be able to choose the user that executes jobs, it is up to you to ensure that this user has only the permissions that you are comfortable letting jobs use.
**NOTE** allowing jobs to access a docker daemon is equivalent to providing root access to the machine.

### How do I install dependencies needed for my jobs?
There are two main approaches available for installing dependencies:
- Allow the jobs to install their own dependencies
  - This approach is the most flexible
  - This will require giving the jobs sufficient privileges to install tools
  - Or to install the tools in a non-overlapping manner (eg. into the working directory)
- Pre-install dependencies on the runner machine
  - This approach is the most secure
  - However it means that if the job’s dependencies change, the runner machine must be reconfigured

### How do caching and workspaces and artifacts work with the runner?
Caches and workspaces and artifacts will be stored in the us-east-1 region of S3. If your runners are not in this region then you may see reduced performance. Although we are not currently charging for runner usage, the transfer and storage of data will incur costs when pricing is introduced.
