# Installation

This file documents the installation steps for the entire workshop.

## GitHub Account

Create a [GitHub account](https://docs.github.com/en/get-started/start-your-journey/creating-an-account-on-github).

## git

Install [git](https://github.com/git-guides/install-git).

## Docker

Install [docker](https://docs.docker.com/engine/install/).

## Slsa-verifier

Install [slsa-verifier](https://github.com/slsa-framework/slsa-verifier?tab=readme-ov-file#option-1-install-via-go).

## Cosign

Install [cosign](https://github.com/sigstore/cosign?tab=readme-ov-file#installation).

Typicallly you can use:

```shell
$ go install github.com/sigstore/cosign/v2/cmd/cosign@v2.1.1
```

## Minikube

Install the local Kubernetes [minikube](https://minikube.sigs.k8s.io/docs/start/).

If you're on Ubuntu, you can use apt:

```shell
$ sudo apt install minikube
$ minikube version
minikube version: v1.31.2
commit: fd7ecd9c4599bef9f04c0986c4a0187f98a4396e

## Kyverno

Install [Kyverno policy engine](https://kyverno.io):

```shell
# Install either the official installation file
$ kubectl create -f ttps://github.com/kyverno/kyverno/publishs/download/v1.11.4/install.yaml
# or a verbose mode enabled from this repository.
# -dumpPayload=true and --v=6 for kyverno-admission-controller 
$ kubectl create -f https://raw.githubusercontent.com/slsa-framework/oss-na24-slsa-workshop/main/activities/04/kyverno/install_verbose_v1.11.4.yml
```

You should now see Kyverno pods:

```shell
$ kubectl get pods -A
...
kyverno       kyverno-admission-controller-6dd8fd446c-4qck5    1/1     Running   0               5s
kyverno       kyverno-background-controller-54f5d9b6f4-whkff   1/1     Running   0               5s
kyverno       kyverno-cleanup-controller-7c5f8bcd79-pwq2d      1/1     Running   0               5s
kyverno       kyverno-reports-controller-7bdb457748-4xbvj      1/1     Running   0               5s
```
