# 03 - Admission controller

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### What you will need

To complete this activity, you need:

1. To have read the admission controller part of the [slides](https://docs.google.com/presentation/d/1w3AWWdXQ8ePoT50R6Ujs-Ji_aXGBa1HmxHBcQIGgH2Q).
1. A GitHub account
1. A docker registry account (alternatively you can use GitHub registry)

### Admission controller

TODO: what it is and what it needs for verification. Use deploy attestation doc / PR.

In this workshop, use Kyverno

## Deep dive

### Installation

#### cosign

You should already have cosign installed as required for Activities [02](https://github.com/laurentsimon/oss-na24-slsa-workshop/tree/main/activities/02) and [03](https://github.com/laurentsimon/oss-na24-slsa-workshop/tree/main/activities/03). If that's not the case, run:

```shell
$ go install github.com/sigstore/cosign/v2/cmd/cosign@v2.1.1
```

#### Local Kubernetes

In this demo, we use a local Kubernetes installation called [minikube](https://minikube.sigs.k8s.io/docs/start/). Any other installation shoudl work, such as [k3s](https://k3s.io/), [kind](https://kind.sigs.k8s.io/), [microk8s](https://microk8s.io/) or other.

To install minikube, follow the instructions [here](https://minikube.sigs.k8s.io/docs/start/).

If you're on Ubuntu, you can use apt:

```shell
$ sudo apt install minikube
$ minikube version
minikube version: v1.31.2
commit: fd7ecd9c4599bef9f04c0986c4a0187f98a4396e
```

Then start it:

```shell
# Kyverno 1.11.4 supports Kubernetes version v1.25-v.18, see https://kyverno.io/docs/installation/#compatibility-matrix.
$ minikube start --kubernetes-version=v1.28
```

Set the alias:

```shell
$ alias kubectl="minikube kubectl --"
```

#### Kyverno policy engine

Install [Kyverno policy engine](https://kyverno.io) as instructed in the [documentation](https://kyverno.io/docs/installation/).
NOTE: Each Kyverno releases supports a range of Kubernetes version. Check out the [compatibility matrix](https://kyverno.io/docs/installation/#compatibility-matrix) to learn more.

Install Kyverno by running:

```shell
# Official installation file
$ kubectl create -f ttps://github.com/kyverno/kyverno/releases/download/v1.11.4/install.yaml
# Verbose mode enabled (in this repository).
# -dumpPayload=true and --v=6 for kyverno-admission-controller 
$ kubectl create -f https://raw.githubusercontent.com/laurentsimon/oss-na24-slsa-workshop/main/activities/04/kyverno/install_verbose_v1.11.4.yml
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

Open a new terminal and monitor the logs for the admission controller and keep this terminal open:

```shell
# Replace 'kyverno-admission-controller-6dd8fd446c-4qck5' with the value for your installation (previous command).
$ kubectl -n kyverno logs -f kyverno-admission-controller-6dd8fd446c-4qck5
```

### Admission controller configuration

We need to configure Kyverno to verify the deployment attestation we created in [Activity 03](TODO).
To learn about verification for attestations signed with cosign, check out their [documentation](https://kyverno.io/docs/writing-policies/verify-images/sigstore/#keyless-signing-and-verification).

Install the policy engine:

```shell
$ kubectl apply -f https://raw.githubusercontent.com/laurentsimon/oss-na24-slsa-workshop/main/activities/04/kyverno/slsa-policy.yml
clusterpolicy.kyverno.io/slsa-policy created
```

### Deploy a pod

Let's deploy the container we built in Activity 01. For that, we will use the [ctivities/04/k8/echo-server-deployment.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/04/k8/echo-server-deployment.yml) pod configuration.

Follow these steps:

1. Update the 

### Future work

#### Additional protection

Remember to try setting up the protection ACLs to protect the policy and allow teams to edit the files they own. See [here](https://github.com/laurentsimon/slsa-policy/blob/main/README.md#org-setup) for details.

#### Pre-submits for CODEOWNER

We must ensure that new team policy files are accompanied by a new CODEOWNER file. If you implement this feature, please share it with us!

#### Deployment of other team's artifacts

In this demo, the attestations are stored along the container. This means that to store the delpoyment attestation, the team calling the evaluator need write access to the registry, so it will not work if you try to delpoy an image that you do not own since you will not have write access to the registry account. The workaround is to create an organization regiistry account on docker, and use that to store all attestations. Follow these steps:

1. Update the [Sign function](https://github.com/laurentsimon/slsa-policy/blob/main/cmd/evaluator/internal/deployment/evaluate/evaluate.go#L91) used to sign the delpoyment attestation. This function is also used for signing the release attestation, but we shoudl not change the logics for signing the release attestation. You will need to add `RegistryClientOpts` to  [cosign.CheckOpts](https://github.com/laurentsimon/slsa-policy/blob/main/cmd/evaluator/internal/utils/crypto/crypto.go#L191-L199) - See [here](https://github.com/slsa-framework/slsa-verifier/blob/v2.5.1/verifiers/internal/gha/verifier.go#L275-L281) for example.
2. Add an option to the evaluator CLI.
3. Update your deployment evaluator to use the new option.
4. Share your code with us! We can merge it in [slsa-policy repository](https://github.com/laurentsimon/slsa-policy).

### UX improments

In this Activity, users need to explicitly call the delpoyment policy evaluator from CI. We may improve UX by integrating the evaluation in gitops tooling such as ArgoCD or as a kubectl plugin. If you are interested in implementing such solution, let us know!

## Take the quizz!

After completing this activity, you should be able to answer the following questions:

1. What is a deployment policy?
2. What invariant is enforced across all policy files?
3. What are trusted roots? Who configures them?
4. What is a deployment attestation? Who creates it? What information does it contain?
5. What metadata is needed to verify a deployment attestation? What happens if the invariant from (2) were not satisfied? Hint: We need to add a policy URI field and verify it. TODO: Link to intoto attestation specs
6. What improvements can we make to improve UX for teams?
