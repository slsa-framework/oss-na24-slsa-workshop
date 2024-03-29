# 04 - Admission controller

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### What you will need

To complete this activity, you need:

1. To have read the admission controller part of the [slides](https://docs.google.com/presentation/d/1w3AWWdXQ8ePoT50R6Ujs-Ji_aXGBa1HmxHBcQIGgH2Q).
1. A GitHub account
1. A docker registry account (alternatively you can use GitHub registry)

### Admission controller

The admission controller is the component that deploys artifacts / containers. It needs to be configured to verify deployment attestations. Verification requires the following metadata:

1. Trusted roots, which is the metadata that defines:
    1. Which evaluators we trust - defined by their identity.
    1. Which protection type (e.g., service account, cluster ID) each evaluator is authoritative for.
    1. Required protection types, which is an optional set of mandatory protection types. A deployment attestation is considered authentic and trusted if it only contains the required protection types. 
1. Mode of enforcement, such as "enforce" or "audit". This allows administrators to onboard new teams and roll out policy upgrades in stages.
1. Failure handling, which configures how unexpected errors or timeouts during evaluation are handled. Fail open behavior admits deployments by default in such a scenario, whereas fail closed behavior defaults to a rejection. The low latency and reliability of using deployment attestations should make these occurrences rare in comparison to real time evaluation

For example, a trusted root could contain (a) public key pubKeyA as evaluator identity, (b) protection type "google service account", (c) a list of service accounts as partition and (d) service account as required protection type.  A deployment attestation is considered authentic and trusted if it is signed using pubKeyA and contains only the protection type "google service account".

In this workshop, we use the open source policy engine [Kyverno](https://kyverno.io/).

## Deep dive

### Installation

#### cosign

You should already have cosign installed as required for Activities [02](https://github.com/laurentsimon/oss-na24-slsa-workshop/tree/main/activities/02) and [03](https://github.com/laurentsimon/oss-na24-slsa-workshop/tree/main/activities/03). If that's not the case, run:

```shell
$ go install github.com/sigstore/cosign/v2/cmd/cosign@v2.1.1
```

#### Local Kubernetes

In this demo, we use a local Kubernetes installation called [minikube](https://minikube.sigs.k8s.io/docs/start/).

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

Install [Kyverno policy engine](https://kyverno.io) by running:

```shell
# Install either the official installation file
$ kubectl create -f ttps://github.com/kyverno/kyverno/releases/download/v1.11.4/install.yaml
# or a verbose mode enabled from this repository.
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

We need to configure Kyverno to verify the deployment attestation we created in [Activity 03](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/03/readme.md).

There are two relevant files to configure it:

1. A verification configuration file containing the trusted roots, in [kyverno/slsa-configuration.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/kyverno/slsa-configuration.yml).
1. A Kyverno enforcer file [kyverno/slsa-enforcer.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/kyverno/slsa-enforcer.yml) that verifies the deployment attestation using the trusted roots.

Clone the repository locally. Then follow the steps:

1. Update the [attestation_creator](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/kyverno/slsa-configuration.yml#L16) field in the verification configuration file. Set it to the value of the evaluator identity that created your deployment attestation in [Activity 03](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/03/readme.md).
1. Install the policy engine

```shell
$ kubectl apply -f kyverno/slsa-configuration.yml
$ kubectl apply -f kyverno/slsa-enforcer.yml
```

### Deploy a pod

Let's deploy the container we built in [Activity 01](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/01/readme.md). For that, we will use the [k8/echo-server-deployment.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/k8/echo-server-deployment.yml) pod definition.


Follow these steps:

1. Edit the [image](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/k8/echo-server-deployment.yml#L23) in the pod definition.
1. WARNING: Since we are running Kubernetes locally, there is no google service account to match against. To simulate one exists for our demo, we make the assumption that its value is exposed via the ["cloud.google.com.v1/service_account" annotation](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/k8/echo-server-deployment.yml#L18). Set the value to the service account configured in your deployment policy for the container.
1. Deploy the container

```shell
$ kubectl apply -f k8/echo-server-deployment.yml
deployment.apps/echo-server-deployment created
```

Run the following commands to confirm the deployment succeeded:

```shell
$ kubectl get polr
NAME                                   KIND         NAME                                      PASS   FAIL   WARN   ERROR   SKIP   AGE
13f78700-4f91-44e6-aa1b-970ed83251dc   ReplicaSet   echo-server-deployment-5bcdd7d764         1      0      0      0       0      4m5s
6b8c9fe2-89fb-4388-92e3-67abdaf3feb0   Pod          echo-server-deployment-5bcdd7d764-87cxt   1      0      0      0       0      4m35s
8863a504-63d6-4455-b9dd-79e15f2bd75f   Pod          echo-server-deployment-5bcdd7d764-2rrrm   1      0      0      0       0      4m35s
977d4976-7ce3-4d97-861a-8a119f3c5e84   Pod          echo-server-deployment-5bcdd7d764-27h96   1      0      0      0       0      4m35s
c482b133-13b1-4678-bb2c-0de2d44c868d   Deployment   echo-server-deployment                    1      0      0      0       0      4m36s
```

Now update the pod definition with an image that is _not_ allowed to run under this service account:

1. Edit the [image](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/k8/echo-server-deployment.yml#L23) in the deployment file. 

```shell
$ kubectl apply -f k8/echo-server-deployment.yml
...THIS SHOULD FAIL...
```

Update the pod definition back to its original value.

1. Edit the ["cloud.google.com.v1/service_account" annotation](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/k8/echo-server-deployment.yml#L18) to a different service account.

```shell
$ kubectl apply -f k8/echo-server-deployment.yml
...THIS SHOULD FAIL...
```

### Future work

#### Limitation

To our knowledge, the google service account is not available to Google's GKE. One way to deploy a real-world example
of this demo is to bind Kubernetes service account to a google service account as described [here](https://github.com/GoogleCloudPlatform/community/blob/master/archived/restrict-workload-identity-with-kyverno/index.md). This is out of scope of the workshop and we leave if for future work. If you take on this task, please share the code with us!

#### Support other protection types

Can you update the policy engine to support other types of protections, e.g., GKE cluster ID, etc. 

## Take the quizz!

After completing this activity, you should be able to answer the following questions:

1. What metadata is needed to configure the admission controller?
