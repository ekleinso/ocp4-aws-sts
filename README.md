# Instructions for running OpenShift on AWS using STS in manual mode

[Configure AWS STS for OpenShift](sts.md) - This document walks through configuring OpenShift with AWS STS tokens so it can leverage EBS and EFS CSI drivers. The OpenShift cluster in this scenario was installed in AWS as a bare-metal cluster so the cloud provider is none and no CSI drivers are provided. The original source for this documentation was provided in the [cloud-credential-operator](https://github.com/openshift/cloud-credential-operator/blob/master/docs/sts.md) repository.

[Self-hosted AWS STS tokens for OpenShift](self-hosted-sts.md) - This document builds on [Configure AWS STS for OpenShift](sts.md) walking through configuring OpenShift to host the STS tokens in NGINX rather than a public S3 bucket. These steps were adopted from the provided documentation in the [cloud-credential-operator](https://github.com/openshift/cloud-credential-operator/blob/master/docs/sts-private-bucket.md) repository.
