# Amazon EKS AMI Build Specification
> [!IMPORTANT]
> Amazon EKS stopped publishing EKS-optimized Amazon Linux 2 (AL2) AMIs on November 26, 2025. AL2023 and Bottlerocket based AMIs for Amazon EKS are available for all supported Kubernetes versions including 1.33 and higher.

This repository contains resources and configuration scripts for building a
custom Amazon EKS AMI with [HashiCorp Packer](https://www.packer.io/). This is
the same configuration that Amazon EKS uses to create the official Amazon
EKS-optimized AMI.

**Check out the [📖 documentation](https://awslabs.github.io/amazon-eks-ami/) to learn more.**

---

## 🚀 Getting started

If you are new to Amazon EKS, we recommend that you follow
our [Getting Started](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
chapter in the Amazon EKS User Guide. If you already have a cluster, and you
want to launch a node group with your new AMI, see [Launching Amazon EKS Worker
Nodes](https://docs.aws.amazon.com/eks/latest/userguide/launch-workers.html).

## 🔢 Pre-requisites

You must have [Packer](https://www.packer.io/) version 1.8.0 or later installed on your local system.
For more information, see [Installing Packer](https://www.packer.io/docs/install/index.html)
in the Packer documentation. You must also have AWS account credentials
configured so that Packer can make calls to AWS API operations on your behalf.
For more information, see [Authentication](https://www.packer.io/docs/builders/amazon.html#specifying-amazon-credentials)
in the Packer documentation.

## 👷 Building the AMI

A Makefile is provided to build the Amazon EKS Worker AMI, but it is just a small wrapper around
invoking Packer directly. You can initiate the build process by running the
following command in the root of this repository:

```bash
# build an AMI with a specific Kubernetes version
make k8s=1.29 os_distro=al2023

# check default value and options in help doc
make help
```

The Makefile chooses a particular kubelet binary to use per Kubernetes version which you can [view here](https://github.com/awslabs/amazon-eks-ami/blob/main/Makefile).

## GitHub Actions

The Notify workflows follow the conventions in
[notification-terraform](https://github.com/cds-snc/notification-terraform) and
[notification-manifests](https://github.com/cds-snc/notification-manifests):
SHA-pinned actions, least-privilege permissions, AWS OIDC authentication, and
Slack build-failure notifications.

- **CI** runs shell/workflow linting, mocked SSM build tests, nodeadm generation,
  vendor, unit-test and build checks, and documentation linting on pull requests
  and pushes to `main`. PR checks do not receive AWS credentials.
- **Check pinned GitHub Actions** uses the shared zizmor check when actions change.
- **Build AMI** is manual only. Run it from `main`, supplying a Kubernetes minor
  version with published EKS binaries and either `x86_64` or `arm64`. It calls
  [the SSM-native builder](https://github.com/cds-snc/notification-eks-ami/blob/main/hack/build-ssm.sh), not Packer, and creates a private,
  encrypted AL2023 AMI in `ca-central-1`. It does not publish or deploy the AMI.
  The run summary contains the AMI ID and S3 log location; build metadata is
  retained as a workflow artifact for 30 days.
- **Workflow failure** reports failed AMI builds to the optional repository
  secret `NOTIFY_DEV_SLACK_WEBHOOK`. An unset webhook produces a warning.

### Configure manual builds

Create a GitHub environment named `ami-build`, restrict deployments to `main`,
and configure required reviewers before enabling builds. Set these environment
variables:

| Variable | Purpose |
| --- | --- |
| `AWS_ROLE_ARN` | OIDC role assumed by the build workflow |
| `SUBNET_ID` | Private builder subnet with outbound repository and SSM access |
| `SECURITY_GROUP_ID` | Existing builder security group; inbound SSH is not needed |
| `ARTIFACT_BUCKET` | Existing private S3 bucket in `ca-central-1`, owned by the build account |
| `INSTANCE_PROFILE` | Optional existing builder instance profile; otherwise the script creates and removes a temporary role/profile |
| `KMS_KEY_ID` | Optional customer-managed key for builder-volume encryption |

The OIDC role must trust `token.actions.githubusercontent.com` with audience
`sts.amazonaws.com` and subject
`repo:cds-snc/notification-eks-ami:environment:ami-build`. Set its maximum session
duration to at least **six hours** (`21600` seconds); jobs have a five-hour timeout
to allow the four-hour provisioning limit plus image creation and cleanup.
Grant the EC2, SSM, S3, IAM/pass-role and optional KMS permissions required by
[the build script](https://github.com/cds-snc/notification-eks-ami/blob/main/hack/build-ssm.sh). An existing instance profile must allow SSM
registration, downloading the source archive, and uploading version information
and command logs to the artifact bucket.

Builds are serialized and new runs do not cancel active builds. Avoid manually
cancelling a build: the script attempts cleanup on exit, but a forced runner
termination can leave AWS resources behind. Inspect the builder instance, any
temporary IAM resources, uploaded source archive, and any AMI/snapshots mentioned
in the logs after an interrupted or failed run. Successful AMIs, snapshots,
version information and SSM logs are retained; configure S3 lifecycle rules and
an AMI retention policy separately.

The AWS-upstream workflows and their local actions have been removed, including
scheduled AWS builds, janitors, dependency/ENI updates, bot commands, CodeCommit
synchronization, changelog automation, and GitHub Pages publishing. Nodeadm
container-based end-to-end tests remain available locally via
`make -C nodeadm test-e2e`; they are not part of the replacement CI.

> **Note**
> The default instance type to build this AMI does not qualify for the AWS free tier.
> You are charged for any instances created when building this AMI.

## 🔒 Security

For security issues or concerns, please do not open an issue or pull request on GitHub. Please report any suspected or confirmed security issues to [AWS Security](https://aws.amazon.com/security/vulnerability-reporting/).

## ⚖️ License Summary

This sample code is made available under a MIT-0 license. See the LICENSE file.

Although this repository is released under the MIT license, when using NVIDIA accelerated AMIs you agree to the NVIDIA Cloud End User License Agreement: https://s3.amazonaws.com/EULA/NVidiaEULAforAWS.pdf.

Although this repository is released under the MIT license, NVIDIA accelerated AMIs
use the third party [open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules). The open-gpu-kernel-modules project's licensing includes the dual MIT/GPLv2 license.

Although this repository is released under the MIT license, NVIDIA accelerated AMIs
use the third party [nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit). The nvidia-container-toolkit project's licensing includes the Apache-2.0 license.

Although this repository is released under the MIT license, Neuron accelerated AMIs
use the third party [Neuron Driver](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/release-notes/runtime/aws-neuronx-dkms/index.html). The Neuron Driver project's licensing includes the GPLv2 license.

Although this repository is released under the MIT license, accelerated AMIs
use the third party [Elastic Fabric Adapter Driver](https://github.com/amzn/amzn-drivers/tree/master/kernel/linux/efa). The Elastic Fabric Adapter Driver project's licensing includes the GPLv2 license.
