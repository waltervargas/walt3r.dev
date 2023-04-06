---
title: Managing k8s with FluxCD
date: <2023-04-06 Thu>
draft: false
tags: fluxcd k8s terraform
---

# Managing k8s with FluxCD

FluxCD is a powerful tool that helps streamline the deployment and management of
applications in a Kubernetes cluster. It is based on the principles of DevOps
and GitOps, which are essential for modern software development and deployment.

GitOps is a specific approach to implementing DevOps practices that is based on
the use of Git as the primary source of truth for infrastructure and application
code. With GitOps, all changes to the infrastructure and application code *are
made* through Git pull requests or change requests, which are then automatically
deployed to the Kubernetes cluster using [FluxCD](https://fluxcd.io/).

## Benefits

The benefits of managing a Kubernetes cluster with FluxCD are numerous. First,
FluxCD automates the deployment of applications, reducing the need for manual
intervention and reducing the risk of human error. This makes the deployment
process faster and more reliable, which is critical in today's fast-paced
software development environment.

Second, FluxCD provides a unified view of the entire deployment process, making
it easier to monitor and troubleshoot issues. With FluxCD, all
deployment-related information is stored in a single place, which makes it easy
to track changes and identify the source of any problems.

Finally, FluxCD enables teams to implement GitOps practices, which provide a
number of benefits. By using Git as the primary source of truth for
infrastructure and application code, teams can ensure that all changes are
tracked and versioned, which improves transparency and accountability. GitOps
also enables teams to implement automated testing and quality control processes,
which further improves the reliability and quality of the software.

In summary, FluxCD is a powerful tool that enables teams to implement DevOps and
GitOps practices, automate the deployment of applications, and improve the
reliability and quality of software deployed on a Kubernetes cluster. By using
FluxCD, teams can streamline their software development and deployment processes
and achieve greater agility and efficiency.

## Installation

Installing FluxCD with the Terraform provider for FluxCD is a relatively
straightforward process (after talking with
[JuanMesa](https://github.com/overdrive3000)) that can be completed in just a
few steps. 

First, you'll need to ensure that you have Terraform installed on your machine.
Next, you'll need to create a Terraform configuration file that specifies the
resources you want to create, including the FluxCD deployment and the Git
repository that FluxCD will use as its source of truth.

```hcl
terraform {
  required_providers {
    flux = {
      source  = "fluxcd/flux"
      version = ">= 0.20.0"
    }
  }
}

/* flux provdier */
provider "flux" {
  kubernetes = {
    config_path = "~/.kube/config"
    config_context = "k3d-k3s-default"
  }
  git = {
    url = "http://gitlab.home.walter.bio/walter/k8s.git"
    http = {
      username = "git"
      password = local.gitlab_token
      allow_insecure_http = true
    }
  }
}

locals {
  gitlab_token = "secret"
}

resource "flux_bootstrap_git" "k3d" {
  path = "clusters/k3d"
}
```

```sh
terraform init
terraform apply
```

```hcl
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # flux_bootstrap_git.k3d will be created
  + resource "flux_bootstrap_git" "k3d" {
      + cluster_domain       = "cluster.local"
      + components           = [
          + "helm-controller",
          + "kustomize-controller",
          + "notification-controller",
          + "source-controller",
        ]
      + id                   = (known after apply)
      + interval             = "1m0s"
      + log_level            = "info"
      + namespace            = "flux-system"
      + network_policy       = true
      + path                 = "clusters/k3d"
      + registry             = "ghcr.io/fluxcd"
      + repository_files     = (known after apply)
      + secret_name          = "flux-system"
      + version              = "v0.41.2"
      + watch_all_namespaces = true
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

## Troubleshooting

I had an problem related to DNS resolution, I have gitlab-ce installed inside
the VM where I am running the cluster, and the git URL I am using
`http://gitlab.home.walter.bio/walter/k8s.git` is based on a DNS name configured
via dnsmasq, so I had the following error: 

```
dial tcp: lookup gitlab.home.walter.bio on 10.43.0.10:53: no such host
```

```sh
k get events -n flux-system -w
```
```
2m29s       Normal    Created              pod/notification-controller-57fb4fcb68-2dw69    Created container manager
2m29s       Normal    Started              pod/notification-controller-57fb4fcb68-2dw69    Started container manager
2m29s       Warning   Unhealthy            pod/notification-controller-57fb4fcb68-2dw69    Readiness probe failed: Get "http://10.42.1.11:9440/readyz": dial tcp 10.42.1.11:9440: connect: connection refused
2m29s       Normal    LeaderElection       lease/notification-controller-leader-election   notification-controller-57fb4fcb68-2dw69_e7df5603-5ddc-4e06-8417-5a2576293630 became leader
50s         Warning   GitOperationFailed   gitrepository/flux-system                       failed to checkout and determine revision: unable to clone 'http://gitlab.home.walter.bio/walter/k8s.git': Get "http://gitlab.home.walter.bio/walter/k8s.git/info/refs?service=git-upload-pack": dial tcp: lookup gitlab.home.walter.bio on 10.43.0.10:53: no such host
```

I had to configure coredns so that it forwards the DNS queries to the corresponding DNS server:

```sh
k -n kube-system edit cm coredns -o yaml
``` 

```diff
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        hosts /etc/coredns/NodeHosts {
          ttl 60
          reload 15s
          fallthrough
        }
        prometheus :9153   
+       forward home.walter.bio <IP_DNS_SERVER>
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    import /etc/coredns/custom/*.server
```

Finally after applying the changes to coredns, installation process successfully finished: 

```
flux_bootstrap_git.k3d: Still creating... [2m50s elapsed]
flux_bootstrap_git.k3d: Still creating... [3m0s elapsed]
flux_bootstrap_git.k3d: Still creating... [3m10s elapsed]
flux_bootstrap_git.k3d: Still creating... [3m20s elapsed]
flux_bootstrap_git.k3d: Creation complete after 3m27s [id=flux-system]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

```sh
k get events -n flux-system
```

```
...
Deployment/flux-system/source-controller configured
Kustomization/flux-system/flux-system configured
NetworkPolicy/flux-system/allow-egress configured
NetworkPolicy/flux-system/allow-scraping configured
NetworkPolicy/flux-system/allow-webhooks configured
GitRepository/flux-system/flux-system configured
7m59s       Normal    ReconciliationSucceeded   kustomization/flux-system                       Reconciliation finished in 734.262869ms, next run in 10m0s
```
