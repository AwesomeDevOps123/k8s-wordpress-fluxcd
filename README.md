# fluxcd-wordpress

In this example we aim to provide code and instructions on how to:

- Create a Kubernetes cluster
- Bootstrap FluxCD to manage the resources in the cluster
- Automate the process of container image updates using FluxCD's image automation mechanism

## Prerequisites

**Kubernetes cluster**

For this example, we will be creating kubernetes clusters locally, using kind. kind runs a local Kubernetes cluster by using Docker containers as “nodes”.

Install dependencies:

```sh
brew install docker kind fluxcd/tap/flux git watch
```

To install kubectl please see https://kubernetes.io/docs/tasks/tools/

**Github access**

A [personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) will be required with full admin permissions for repositories.

## Repository structure

The repo is split into 2 main sections, each with 2 envs, dev and prod:

- **apps**, which contains a base folder with templates for all the required kubernetes resources to be deployed, and an environment specific folder with any environment specific configuration required

- **clusters**, which contains the flux system configuration (auto-generated by flux) and a kustomization to deploy the apps resources

```
├── apps
│   ├── base
│   ├── dev 
│   └── prod
│
└── clusters
    ├── dev
    └── prod
```

## Creating the Kubernetes cluster

Create a cluster:

```sh
kind create cluster --name dev-cluster
kind create cluster --name prod-cluster
```

Check clusters have been successfully created:

```sh
kind get clusters
```

In order to interact with a specific cluster, you only need to specify the cluster name as a context in kubectl:

```sh
kubectl cluster-info --context kind-dev-cluster
kubectl cluster-info --context kind-prod-cluster
```

## Bootstrapping fluxCD

Fork this repository to a personal GitHub account and export your GitHub personal access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Make sure you are in the dev cluster context to start with:

```sh
kubectl config use-context kind-dev-cluster
```

Verify that your dev cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Bootstrap flux:

```sh
  flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --personal \
    --token-auth \
    --branch=main \
    --path=./clusters/dev \
    --components-extra=image-reflector-controller,image-automation-controller
```

To check if the helm release is being successfully installed:

```console
$ watch flux get helmreleases -n wordpress

NAMESPACE    	NAME         	REVISION	SUSPENDED	READY	MESSAGE 
wordpress      	wordpress    	20.1.2   	False    	True 	Helm install succeeded for release wordpress/wordpress.v1 with chart wordpress@20.1.2
```

Verify that the demo wordpress app can be accessed:

```console
$ kubectl get pods --namespace wordpress

$ kubectl port-forward pod/<wordpress-pod> 8080:8080 --namespace wordpress
```

The wordpress service should now be accessible at http://localhost:8080

To bootstrap flux in the prod envirnment, set the kubectl context to `kind-prod-cluster` and set the path in the bootstrap command below to your production cluster:

```sh
  flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --personal \
    --token-auth \
    --branch=main \
    --path=./clusters/prod \
    --components-extra=image-reflector-controller,image-automation-controller
```

## Image Automation

The image automation has been configured so that dev and prod follow different tags.

**Dev** - tracking the latest release candidate (e.g 6.3.2-debian-11-r7, 6.4.2-debian-11-r4)

**Prod** - tracking the latest stable version (e.g. 6.3.2, 6.4.2)

For a detailed guide on how image automation works, please visit - https://fluxcd.io/flux/guides/image-update/

**Check if image automation works**

The initial image tag used in this repo was 6.3.1 for both environments, which can be seen in the first commit made to the repo. If the flux image automation is working as expected, we should see extra commits from flux, updating each environment with the relevant tag.

**Clean up**

To delete all the resources including the kind clusters run:

```sh
kind delete cluster --name dev-cluster
kind delete cluster --name prod-cluster
```

## FluxCD official docs
For more info and best practices on how to configure Flux please see https://fluxcd.io/flux.