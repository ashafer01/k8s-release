# k8s-release

A tool for performing repeatable deployments of static Kubernetes manifest files.
This uses the same subcommands and general workflow as
[helm-release](https://github.com/ashafer01/helm-release).

## Installation

Homebrew users:

```
brew install ashafer01/repo/k8s-release
```

Users of other platforms may clone the repo or even just copy/paste
the script. The following dependencies will also need to be fulfilled:

* `kustomize`
* `kubectl`
* `wget`
* bash and coreutils

*In Development* No stability guarantees at present.

## Usage Example

Note that the tool makes use of the `$EDITOR` variable. Be
sure to set it to a text editor you are comfortable with before
continuing.

After your environment is prepped, set up a new release
directory. For this example, we will take the AWS document for
[using FluentD to send EKS logs to CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs.html)
and we will make it a repeatable deployment.

```bash
mkdir my_eks_fluentd
cd my_eks_fluentd
k8s-release init
```

This will open a file in your EDITOR. Read over the comments
and delete the second block. Then, add all the K8s URLs
from the document on separate lines:

```
# This directory is managed by:
# https://github.com/ashafer01/k8s-release

# Implementation of:
# https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs.html
# As retrieved on 2020-01-24

# Step 1
https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/master/k8s-yaml-templates/cloudwatch-namespace.yaml

# Step 2.2
https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/master/k8s-yaml-templates/fluentd/fluentd.yaml
```

When prompted to edit kustomization.yaml now, say no.

The process will finish and successfully render a yaml file, but we
still have to include Step 2.1 from the document. To do so, run
this slightly modified version of the command:

```bash
kubectl create --dry-run -o yaml configmap cluster-info \
--from-literal=cluster.name=YOUR_EKS_CLUSTER_NAME \
--from-literal=logs.region=YOUR_AWS_REGION -n amazon-cloudwatch \
> cluster-info.yaml
```

Then, update kustomization.yaml to reference the new resource.
The final state of the file should look like this:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
# This directory contains all of the unmodified upstream files along
# with a no-op kustomization.yaml including all of them as resources
- .release/upstream
# Any local files should be manually listed here
- cluster-info.yaml

# Add transformers/generators here
# https://github.com/kubernetes-sigs/kustomize/blob/master/docs/fields.md
```

Then, re-render the release to include the new ConfigMap:

```
k8s-release refresh
```

And finally, deploy it to the current context:

```
k8s-release apply
```

## Acceptable URLs

URLs are directly passed through to `wget`. They may refer to a single
YAML file, or a `.tar.gz` or `.tar.bz2` archive containing one or more
YAML files. Any other files in an archive will be ignored.
