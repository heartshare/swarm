# Template for deploying k3s backed by Flux

Template for deploying a single [k3s](https://k3s.io/) cluster with [k3os](https://k3os.io) backed by [Flux](https://toolkit.fluxcd.io/) and [SOPS](https://toolkit.fluxcd.io/guides/mozilla-sops/).

## Overview

- [Introduction](https://github.com/k8s-at-home/template-cluster-k3s#wave-introduction)
- [Prerequisites](https://github.com/k8s-at-home/template-cluster-k3s#memo-prerequisites)
- [Repository structure](https://github.com/k8s-at-home/template-cluster-k3s#open_file_folder-repository-structure)
- [Lets go!](https://github.com/k8s-at-home/template-cluster-k3s#rocket-lets-go)
- [Post installation](https://github.com/k8s-at-home/template-cluster-k3s#mega-post-installation)
- [Thanks](https://github.com/k8s-at-home/template-cluster-k3s#handshake-thanks)

## :wave:&nbsp; Introduction

The following components will be installed in your [k3s](https://k3s.io/) cluster by default. They are only included to get a minimum viable cluster up and running. You are free to add / remove components to your liking but anything outside the scope of the below components are not supported by this template.

Feel free to read up on any of these technologies before you get started to be more familiar with them.

- [flannel](https://github.com/flannel-io/flannel)
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- [flux](https://toolkit.fluxcd.io/)
- [metallb](https://metallb.universe.tf/)
- [cert-manager](https://cert-manager.io/) with Cloudflare DNS challenge
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)

## :memo:&nbsp; Prerequisites

### :computer:&nbsp; Nodes

Already provisioned Bare metal k3os cluster.

### :wrench:&nbsp; Tools

:round_pushpin: You need to install the required CLI tools listed below on your workstation.

| Tool                                                               | Purpose                                                             | Minimum version | Required |
|--------------------------------------------------------------------|---------------------------------------------------------------------|:---------------:|:--------:|
| [k3os](https://k3os.io)                                            | Tool to install k3s on your nodes                                   |    `0.19.5`     |    ✅     |
| [kubectl](https://kubernetes.io/docs/tasks/tools/)                 | Allows you to run commands against Kubernetes clusters              |    `1.21.0`     |    ✅     |
| [flux](https://toolkit.fluxcd.io/)                                 | Operator that manages your k8s cluster based on your Git repository |    `0.12.3`     |    ✅     |
| [SOPS](https://github.com/mozilla/sops)                            | Encrypts k8s secrets with GnuPG                                     |     `3.7.1`     |    ✅     |
| [GnuPG](https://gnupg.org/)                                        | Encrypts and signs your data                                        |    `2.2.27`     |    ✅     |
| [pinentry](https://gnupg.org/related_software/pinentry/index.html) | Allows GnuPG to read passphrases and PIN numbers                    |     `1.1.1`     |    ✅     |
| [direnv](https://github.com/direnv/direnv)                         | Exports env vars based on present working directory                 |    `2.28.0`     |    ❌     |
| [pre-commit](https://github.com/pre-commit/pre-commit)             | Runs checks during `git commit`                                     |    `2.12.0`     |    ❌     |
| [kustomize](https://kustomize.io/)                                 | Template-free way to customize application configuration            |     `4.1.0`     |    ❌     |
| [helm](https://helm.sh/)                                           | Manage Kubernetes applications                                      |     `3.5.4`     |    ❌     |
| [go-task](https://github.com/go-task/task)                         | A task runner / simpler Make alternative written in Go              |     `3.7.0`     |    ❌     |
| [prettier](https://github.com/prettier/prettier)                   | Prettier is an opinionated code formatter.                          |     `2.3.2`     |    ❌     |

### :warning:&nbsp; pre-commit

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) will check to make sure you are not by accident commiting your secrets un-encrypted.

After pre-commit is installed on your machine run:

```sh
pre-commit install-hooks
```

### :closed_lock_with_key:&nbsp; Setting up GnuPG keys

:round_pushpin: Here we will create a personal and a Flux GPG key. Using SOPS with GnuPG allows us to encrypt and decrypt secrets.

1. Create a Personal GPG Key, password protected, and export the fingerprint. It's **strongly encouraged** to back up this key somewhere safe so you don't lose it.

```sh
export GPG_TTY=$(tty)
export PERSONAL_KEY_NAME="First name Last name (location) <email>"

gpg --batch --full-generate-key <<EOF
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${PERSONAL_KEY_NAME}
EOF

gpg --list-secret-keys "${PERSONAL_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       772154FFF783DE317KLCA0EC77149AC618D75581
# uid           [ultimate] k8s@home (Macbook) <k8s-at-home@gmail.com>
# sub   rsa4096 2021-03-11 [E]

export PERSONAL_KEY_FP=772154FFF783DE317KLCA0EC77149AC618D75581
```

2. Create a Flux GPG Key and export the fingerprint

```sh
export GPG_TTY=$(tty)
export FLUX_KEY_NAME="Cluster name (Flux) <email>"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${FLUX_KEY_NAME}
EOF

gpg --list-secret-keys "${FLUX_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       AB675CE4CC64251G3S9AE1DAA88ARRTY2C009E2D
# uid           [ultimate] Home cluster (Flux) <k8s-at-home@gmail.com>
# sub   rsa4096 2021-03-11 [E]

export FLUX_KEY_FP=AB675CE4CC64251G3S9AE1DAA88ARRTY2C009E2D
```

### :sailboat:&nbsp; Installing k3os

TODO!!

### :small_blue_diamond:&nbsp; GitOps with Flux

:round_pushpin: Here we will be installing [flux](https://toolkit.fluxcd.io/) after some quick bootstrap steps.

1. Verify Flux can be installed

```sh
flux --kubeconfig=./kubeconfig check --pre
# ► checking prerequisites
# ✔ kubectl 1.21.0 >=1.18.0-0
# ✔ Kubernetes 1.20.5+k3s1 >=1.16.0-0
# ✔ prerequisites checks passed
```

2. Pre-create the `flux-system` namespace

```sh
kubectl --kubeconfig=./kubeconfig create namespace flux-system --dry-run=client -o yaml | kubectl --kubeconfig=./kubeconfig apply -f -
```

3. Add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
gpg --export-secret-keys --armor "${FLUX_KEY_FP}" |
kubectl --kubeconfig=./kubeconfig create secret generic sops-gpg \
    --namespace=flux-system \
    --from-file=sops.asc=/dev/stdin
```

4. Export more environment variables for application configuration

```sh
# The repo you created from this template
export BOOTSTRAP_GITHUB_REPOSITORY="https://github.com/k8s-at-home/home-cluster"
# Choose one of your domains or use a made up one
export BOOTSTRAP_DOMAIN="k8s-at-home.com"
# Pick a range of unused IPs that are on the same network as your nodes
export BOOTSTRAP_METALLB_LB_RANGE="169.254.1.10-169.254.1.20"
# The load balancer IP for ingress-nginx, choose from one of the available IPs above
export BOOTSTRAP_INGRESS_NGINX_LB="169.254.1.10"
```

5. Create required files based on ALL exported environment variables.

```sh
envsubst < ./tmpl/.sops.yaml > ./.sops.yaml
envsubst < ./tmpl/cluster-secrets.sops.yaml > ./cluster/base/cluster-secrets.sops.yaml
envsubst < ./tmpl/cluster-settings.yaml > ./cluster/base/cluster-settings.yaml
envsubst < ./tmpl/gotk-sync.yaml > ./cluster/base/flux-system/gotk-sync.yaml
envsubst < ./tmpl/secret.sops.yaml > ./cluster/core/cert-manager/secret.sops.yaml
```

6. **Verify** all the above files have the correct information present

7. Encrypt `cluster/cluster-secrets.sops.yaml` and `cert-manager/secret.sops.yaml` with SOPS

```sh
export GPG_TTY=$(tty)
sops --encrypt --in-place ./cluster/base/cluster-secrets.sops.yaml
sops --encrypt --in-place ./cluster/core/cert-manager/secret.sops.yaml
```

:round_pushpin: Variables defined in `cluster-secrets.sops.yaml` and `cluster-settings.sops.yaml` will be usable anywhere in your YAML manifests under `./cluster`

8. **Verify** all the above files are **encrypted** with SOPS

9. Push you changes to git

```sh
git add -A
git commit -m "initial commit"
git push
```

10. Install Flux

:round_pushpin: Due to race conditions with the Flux CRDs you will have to run the below command twice. There should be no errors on this second run.

```sh
kubectl --kubeconfig=./kubeconfig apply --kustomize=./cluster/base/flux-system
# namespace/flux-system configured
# customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/buckets.source.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/gitrepositories.source.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/helmcharts.source.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/helmreleases.helm.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/helmrepositories.source.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/kustomizations.kustomize.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/providers.notification.toolkit.fluxcd.io created
# customresourcedefinition.apiextensions.k8s.io/receivers.notification.toolkit.fluxcd.io created
# serviceaccount/helm-controller created
# serviceaccount/kustomize-controller created
# serviceaccount/notification-controller created
# serviceaccount/source-controller created
# clusterrole.rbac.authorization.k8s.io/crd-controller-flux-system created
# clusterrolebinding.rbac.authorization.k8s.io/cluster-reconciler-flux-system created
# clusterrolebinding.rbac.authorization.k8s.io/crd-controller-flux-system created
# service/notification-controller created
# service/source-controller created
# service/webhook-receiver created
# deployment.apps/helm-controller created
# deployment.apps/kustomize-controller created
# deployment.apps/notification-controller created
# deployment.apps/source-controller created
# unable to recognize "./cluster/base/flux-system": no matches for kind "Kustomization" in version "kustomize.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "GitRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
```

:tada: **Congratulations** you have a Kubernetes cluster managed by Flux, your Git repository is driving the state of your cluster.

### VSCode SOPS extension

[VSCode SOPS](https://marketplace.visualstudio.com/items?itemName=signageos.signageos-vscode-sops) is a neat little plugin for those using VSCode.
It will automatically decrypt you SOPS secrets when you click on the file in the editor and encrypt them when you save  and exit the file.

### :point_right:&nbsp; Debugging

Manually sync Flux with your Git repository

```sh
flux --kubeconfig=./kubeconfig reconcile source git flux-system
# ► annotating GitRepository flux-system in flux-system namespace
# ✔ GitRepository annotated
# ◎ waiting for GitRepository reconciliation
# ✔ GitRepository reconciliation completed
# ✔ fetched revision main/943e4126e74b273ff603aedab89beb7e36be4998
```

Show the health of you kustomizations

```sh
kubectl --kubeconfig=./kubeconfig get kustomization -A
# NAMESPACE     NAME          READY   STATUS                                                             AGE
# flux-system   apps          True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    3d19h
# flux-system   core          True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    4d6h
# flux-system   crds          True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    4d6h
# flux-system   flux-system   True    Applied revision: main/943e4126e74b273ff603aedab89beb7e36be4998    4d6h
```

Show the health of your main Flux `GitRepository`

```sh
flux --kubeconfig=./kubeconfig get sources git
# NAME           READY	MESSAGE                                                            REVISION                                         SUSPENDED
# flux-system    True 	Fetched revision: main/943e4126e74b273ff603aedab89beb7e36be4998    main/943e4126e74b273ff603aedab89beb7e36be4998    False
```

Show the health of your `HelmRelease`s

```sh
flux --kubeconfig=./kubeconfig get helmrelease -A
# NAMESPACE   	    NAME                  	READY	MESSAGE                         	REVISION	SUSPENDED
# cert-manager	    cert-manager          	True 	Release reconciliation succeeded	v1.3.0  	False
# default        	homer                 	True 	Release reconciliation succeeded	4.2.0   	False
# networking  	    ingress-nginx       	True 	Release reconciliation succeeded	3.29.0  	False
```

Show the health of your `HelmRepository`s

```sh
flux --kubeconfig=./kubeconfig get sources helm -A
# NAMESPACE  	NAME                 READY	MESSAGE                                                   	REVISION                                	SUSPENDED
# flux-system	bitnami-charts       True 	Fetched revision: 0ec3a3335ff991c45735866feb1c0830c4ed85cf	0ec3a3335ff991c45735866feb1c0830c4ed85cf	False
# flux-system	ingress-nginx-charts True 	Fetched revision: 45669a3117fc93acc09a00e9fb9b4445e8990722	45669a3117fc93acc09a00e9fb9b4445e8990722	False
# flux-system	jetstack-charts      True 	Fetched revision: 7bad937cc82a012c9ee7d7a472d7bd66b48dc471	7bad937cc82a012c9ee7d7a472d7bd66b48dc471	False
# flux-system	k8s-at-home-charts   True 	Fetched revision: 1b24af9c5a1e3da91618d597f58f46a57c70dc13	1b24af9c5a1e3da91618d597f58f46a57c70dc13	False
```

Flux has a wide range of CLI options available be sure to run `flux --help` to view more!

### :robot:&nbsp; Automation

- [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) is a very useful tool that when configured will start to create PRs in your Github repository when Docker images, Helm charts or anything else that can be tracked has a newer version. The configuration for renovate is located [here](./.github/renovate.json5).

There's also a couple Github workflows included in this repository that will help automate some processes.

- [Flux upgrade schedule](./.github/workflows/flux-schedule.yaml) - workflow to upgrade Flux.
- [Renovate schedule](./.github/workflows/renovate-schedule.yaml) - workflow to annotate `HelmRelease`'s which allows [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) to track Helm chart versions.

