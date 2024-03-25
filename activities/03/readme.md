# 02 - Deployment policy

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### What you will need

To complete this activity, you need:

1. To have read the [policy part](https://docs.google.com/presentation/d/1w3AWWdXQ8ePoT50R6Ujs-Ji_aXGBa1HmxHBcQIGgH2Q).
1. A GitHub account
1. A docker registry account (alternatively you can use GitHub registry)

### Deployment policy and attestation

A delpoyment policy expresses which environment an artifact is allowed to be deployed to. Binding an artifact to its expected deployment environment is one of the principles used internally at Google; it is also a feature provided by [Google Cloud Binauthz](https://cloud.google.com/binary-authorization/). A deployment environment may be a cloud machine wher we deploy containers, a developer workstation where we delpoy packages (pip, npm, etc.) or even an Android device where we deploy applications. 

When deploying an artifact (e.g., a container, a smartphone app), we want to restrict which environment the artifact is allowed to be deployed / run. The environment has access to resources we want to protect, such as a service account, a Spiffe ID, a Kubernetes pod ID, an SEAndroid context, etc. The deployment attestation authoritatively binds an artifact to a deployment environment where an artifact is allowed to be deployed.

The ability to bind an artifact to an environment is paramount to reduce the blast radius if vulnerabilties are exploited or environments are compromised. Attackers who gain access to an environment will pivot based on the privileges of this environment, so it is imperative to follow the privilege of least principle and restrict which code is allowed to run in which environment. For example, we would not want to deploy a container with remote shell capabilities on a pod that processes user credentials, even if this container is integrity protected at the highest SLSA level. Conceptually, this is similar to how we think about sandboxing and least-privilege principle on operating systems. The same concepts apply to different types of environments, including cloud environments.

The decision to allow or deny a deployment request may happen in "real-time", i.e. the control plane may query an online authorization service at the time of the deployment. Such an authorization service requires low-latency / high-availability SLOs to avoid deployment outage. This is exacerbated in systems like Kubernetes where admission webhooks run for every pod deployed. Thus it is often desirable to "shift-left" and perform an authorization evaluation ahead of time before a deployment request reaches the control plane. The deployment attestation is the proof of authorization that the control plane may use to make its final decision, instead of querying an online service itself. Verification of the deployment attestation is simple, fast and may be performed entirely offline. Overall, this shift-left strategy provides the following advantages: less likely to cause production issues, better debugging UX for devs, less auditing and production noise for SREs and security teams.

## Deep dive

### Deployment policy setup

#### Repository setup

TODO: skip https://github.com/laurentsimon/slsa-policy/blob/main/README.md#deployment-policy

TODO Fork repo

Explain: 
- Organization roots
- Evaluator service
- Pre-submits

#### Team setup

##### Configure the policy

Configure policy

##### Call the evaluator in CI

Call the evaluator. Run it. Generates an attestation and signs with SIgstore. Attestation is stored on the same registry and repositry as the artifact. Read more at X.

##### Verify release attestation manually

Verify manually with cosign

TODO: To install cosign, follow the instructions from [here](TODO).

Make sure you have access to your image by authenticating to docker:

```shell
REGISTRY_TOKEN=<your-token>
REGISTRY_USERNAME=<registru-username>
docker login -u "${REGISTRY_USERNAME}" "${REGISTRY_TOKEN}"
```

To verify your container, use the following command:

```shell
# Update the image as recorded in your logs
image=docker.io/laurentsimon/oss-na24-slsa-workshop-project1-echo-server@sha256:3cea74d6b3d033869f4185a9478d66009c97fda18631c0650f7ea3be4fca722c
source_uri=github.com/laurentsimon/oss-na24-slsa-workshop-project1
path/to/slsa-verifier verify-image "${image}" --source-uri "${source_uri}" --builder-id=https://github.com/slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml
```

The slsa-verifier verifies:

1. The cryptographic digest of the container matches the one in the attestation
2. The source origin of the repository
3. The builder who built and generated the attestation.

Pass command `--print-provenance | jq` to see the plaintext intoto attestation.

Explore other commands by using the `--help` command. You can read more on the project repository [here](https://github.com/slsa-framework/slsa-verifier).

Run the same verification command but remove the `sha256:xxx` part of the image name: `image=image=docker.io/laurentsimon/oss-na24-slsa-workshop-project1-echo-server`. Why is this failing? See hints [here](https://github.com/slsa-framework/slsa-verifier/tree/main?tab=readme-ov-file#toctou-attacks).

#### Do it at home

##### Set up ACLs

And pre-submits

##### Use an organization store

TODO: cannot write

## Take the quizz!

After completing this activity, you should be able to answer the following questions:

1. What is a deployment policy? What information does it take as input?
2. What are trusted roots? Who configures them?
3. What is a deployment attestation? Who creates it? What information does it contain?
4. What metadata is needed to verify a deployment attestation? Why is a policy URI not needed in our example? Are there cases where it may be needed? Why?
