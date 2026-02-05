---
layout: post
title:  "Introduction to Flux"
date:   2026-02-01 08:52:11 +0100
categories: Software
mathjax: false
---

Flux is a tool based on GitOps principles that automatically applies changes in the Kubernetes cluster based on configuration stored in version control. It runs as a set of controllers inside the cluster that monitor the linked repositories for changes. These core controllers are:

* Source controller
* Helm controller
* Kustomize controller
* Notifications controller

These can be seen running in the cluster by looking at the pods running in the "Flux namespace".

# Source controller

It monitors source repositories (Git repositories, Helm repositories, OCI registers...) and if it detects that something has changed, it pulls it.

For example, if there is a `HelmRepository` resource that looks as follows:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: example-charts
  namespace: example-namespace
spec:
  interval: 5m0s
  url: https://example-charts.github.io/charts/
```

the source controller will monitor the Helm Chart repository https://example-charts.github.io/charts/ and download any new charts that are released. The pulled Helm Charts will be tracked by `HelmChart` resources.

Another example, if there is a `GitRepository` in the resource that looks as follows:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: example-git
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  url: ssh://git@github.com/organisation/example-repo
  secretRef:                    
    name: flux-system
```

the source controller will monitor the `main` branch of this Github repository and pull any changes locally.

### Helm controller

The Helm controller monitors the `HelmRelease` resources. These can be defined per application, for example:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: example-application
  namespace: example-namespace
spec:
  chart:
    spec:
      chart: example-chart
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: example-charts # The name of the Helm Chart repository defined in the HelmRepository resource
      version: 1.1.0  
  interval: 30s
  values: {} # The values needed for the chart
```

In the `HelmRelease` resource, you can specify the version of the Helm chart to deploy and all the values that are needed.
Whenever there is a change in a `HelmRelease` resource, the Helm controller performs the `helm install` or `helm upgrade` steps to reconcile the state of the cluster. 

### Kustomize controller

The Kustomize controller ensures that the cluster state matches the desired state described by Kubernetes manifests. The `GitRepository`, `HelmRelease` and the `HelmRepository` resources that were mentioned above are part of these manifests. If there are changes in these manifests, then the Kustomize controller applies them and the other controllers then do their job.

The manifests that should be applied are specified in the `Kustomization` resources. These can specify the resources of the Flux system itself or the `HelmRelease` and/or the `GitRepository` resources needed to deploy the applications.

An example `Kustomization` that makes the Kustomization controller apply the `GitRepository` resource can defined as:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: example-kustomization
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./path/to/GitRepository-Resource
  prune: true
  sourceRef:
    kind: GitRepository
    name: example-git
  timeout: 2m0s
  wait: true
```

**Note:** the Source controller relies on the `GitRepository` resource to track the changes to the Flux repository, but it can only do so after the manifest of the `GitRepository` resource has been applied. The Kustomize controller reliest on the Source to pull from the Github repository defined in the `GitRepository` to be able to apply the manifests. This means that the first time that Flux is set up there is a bootstrap step to apply the manifests for the first time.

### Notification controller

The notification controller is an event forwarder and notification dispatcher for the reconciliation controllers. It can send notifications (for example on Slack) based on events from the reconciliation controllers. It can also handle incoming events from external systems (GitHub, GitLab, Bitbucket, Harbor, Jenkins, etc) and notify the reconciliation controllers.