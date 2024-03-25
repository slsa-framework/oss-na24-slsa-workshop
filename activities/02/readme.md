# 02 - Release policy

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### What you will need

To complete this activity, you need:

1. To have read the [policy part](https://docs.google.com/presentation/d/1w3AWWdXQ8ePoT50R6Ujs-Ji_aXGBa1HmxHBcQIGgH2Q).
1. A GitHub account
1. A docker registry account (alternatively you can use GitHub registry)

### Release policy and attestation

A release policy is a set of requirements that software must meet to be released: This could be passing a vulnerability scan, that the correct source repository was used, that the code was not tampered with (i.e. a minimum SLSA level), etc. As an organization, this policy provides visibility and auditability into their supply-chain. Optionally, an organization may _require_ all software meet a minimum set of requirements before releasing internally or externally on public package registries.

After evaluating a release policy, the evaluator generates a so-called "release attestation" as evidence that the policy evaluation succeeded. The attesttaion contains information about the properties of the artifact that were evaluated during evaluation, such as its SLSA level or whether it was scanned for vulnerabilities or not. The attestation can subsequently be consumed by end-users to determine if the artifact meets their requirements for thei use case.

The release attestation hides away details of the development cycle from your consumers. For example, most of them need not know or keep track of which repository your artifact was built from. All that matters to them is the level of integrity protection (SLSA level) the artifact has or whether it was scanned or not. But if certain consumers want to know more details about your artifacts (e.g., what builder you used, what repository it was built from), they can verify them directly using attestations referenced in the release attestation (e.g., SLSA provenance).

In this activity, we will focus on the integrity protections applied to an artifact during its development, i.e. its minimum SLSA level. This will hhelp us achieve the first goal of the workshop, which is to ensure that all produced containers are protected against tampering across the SDLC.

## Deep dive

### Release policy setup

#### Repository setup

TODO: skip https://github.com/laurentsimon/slsa-policy/blob/main/README.md#release-policy

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

1. What is a release policy? 
2. What are trusted roots? Who configures them?
3. What is a release attestation? Who creates it? What information does it contain?
4. What metadata is needed to verify a release attestation? Why is a policy URI not needed in our example? Are there cases where it may be needed? Why?
