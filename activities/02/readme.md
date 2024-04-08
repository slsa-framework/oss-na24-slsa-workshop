# 02 - Publish policy

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### What you will need

Install the [necessary software](https://github.com/slsa-framework/oss-na24-slsa-workshop/blob/main/INSTALLATION.md).

### Publish policy and attestation

A publish policy is a set of requirements that software must meet to be published: This could be passing a vulnerability scan, that the correct source repository was used, that the code was not tampered with (i.e. a minimum SLSA level), etc. As an organization, this policy provides visibility and auditability into their supply-chain. An organization may require all software meet a minimum set of requirements before releasing internally or externally on public package registries.

After evaluating a publish policy, the policy evaluator generates a so-called "publish attestation" as evidence that the policy evaluation succeeded. The attestation expresses the fact that software is ready to be published and optionally contains information about the properties of the artifact evaluated, such as its SLSA level, whether it was scanned for vulnerabilities, its compliance with development standards (Fedramp, EO 14028), etc. Note that evaluating a policy prior to generating a publish attestation is _not_ a requirement, but encouraged. The attestation can be used in two ways:

1. Package registries may provide a mechanism for maintainers to enforce that only packages that have a valid publish attestation are published. This effectively allows organizations to ensure that their published software meet the requirements set by the publish policy across all their projects.
2. Consumers of packages may read the properties in the publish attestations and decide if it meets the requirements for their use case.

The publish attestation hides away details of the development cycle from your consumers. For example, most of them need not know or keep track of which repository your artifact was built from. All that matters to them is the set of properties to help make an informed decision whether to use an artifact or not (integrity protection, presence of vulnerabilities, compliance with known development standards, etc). If certain consumers want to know more details about your artifacts (e.g., what builder you used, what repository it was built from), they can verify upstream attestations (e.g., SLSA provenance from the builder) themselves using attestations referenced in the publish attestation. Note that if a project is not developed as open source or if it is not built with a public builder (GitHub, GCB), the publish attestation still allows producers to convey certain characteritics about their published artifact.

In this activity, we will focus on the integrity protections applied to an artifact during its development, i.e. its minimum SLSA level. This will help us achieve the first goal of the workshop, which is to ensure that all produced containers are protected against tampering across the SDLC.

NOTE: The name and schema of this attestation is under discussion in the community and is subject to changes.

## Deep dive

### Publish policy setup

The policy is stored in a central location administered by the organization. This gives the organization a central view of 
all the projects and their configuration. In this activity, we store the policy in a GitHub repository owned by the organization. As we will see shortly, the policy is sub-devided
into:

1. A configuration maintained by the organization which applies to all teams within the organization
2. Team-specific configuration maintained by teams for their project.

#### Repository protections

As described in the previous paragraph, the organization stores all policy configurations in a central location. To reduce the bottleneck on the organization admins,
it is important for team policies to be editable _without_ admin intervention. We need to enable teams to review and edit changes on their own, while
preventing unauthorized changes from other teams. Similarly, the configuration maintained by the organization must be protected again unauthorized
changes by other teams. We can echieve these protections by setting up [branch protection rules](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/managing-rulesets-for-a-repository) and [CODEOWNER settings](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#additional-settings). To simplify the workshop and due to time constraint,
we will assume these protections are in place. If you wish to implement these protections after the workshop, refer to the steps described in the [policy-setup](https://github.com/laurentsimon/slsa-policy/blob/main/README.md#policy-setup).

#### Organization roots

Fork this repository [https://github.com/slsa-framework/oss-na24-slsa-workshop-organization](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization) by clicking this [link](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/fork).

Under directory [policies/publish](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/tree/main/policies/publish) are the configuration files for the publish policy. The file maintained by the organization admins
is [org.json](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/tree/main/policies/publish/org.json). This file contains a list of "trusted roots", which is a list of trusted entities. In this demo,
each trusted root is a SLSA builder allowed to build projects for the organization, along with its corresponding SLSA level. For example, the [first listed builder](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/org.json#L5-L8) has `id:https://github.com/slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml`, `slsa_level:3` and is given the short name `github_generator_level_3`.

#### Evaluator service

The repository contains a GitHub workflow [.github/workflows/image-publisher.yml](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-publisher.yml) which evaluates the publish policy. It contains the following logic:

1. [Detects the refs](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-publisher.yml#L47-L67) at which it was called by a project. This is due to a quirk of how GitHub reusable workflows work. You can ignore this part of the code.
1. [Install the policy CLI](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-publisher.yml#L116-L126).
1. [Run the policy CLI](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-publisher.yml#L110-L120).

#### Pre-submits

Across the policy, there is an important invariant to maintain, which is that a package must be owned by at most _one_ team. In other words, we must ensure that across the policy,
a package is only referenced once across all configuration files. For this, we make use of pre-submits run on pull requests.
The pre-submits are configured in the workflow file [.github/workflows/pre-submit.publish-policy.yml](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/pre-submit.publish-policy.yml).

An additional required pre-submit is to ensure that new team policy files are accompanied by a new CODEOWNER file. We leave this as [future work](#pre-submits-for-codeowner).

### Team setup

#### Project ownership

To ensure only the team that owns a package is allowed to edit its configuration, the team must edit or create a CODEOWNER file that give them ownership of the configurations (files or entire directories).
As explained in [repository protections](#repository-protections), for time constraints in this workshop we will assume this is done, but you can set it up [after the workshop](#set-up-acls).

##### Configure the policy

The file to be protected by the CODEOWNER file is [echo-server.json](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json) which describes the team policy for the container built in [Activity 01](https://github.com/slsa-framework/oss-na24-slsa-workshop/blob/main/activities/01/readme.md). The file contains the following sections:

1. The [package](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json#L3) section describes the package to publish, i.e., [docker.io/laurentsimon/oss-na24-slsa-workshop-project1-echo-server](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json#L4) and will be used both for [staging and prod](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json#L7). NOTE: The environment (prod, staging) is optional.
1. The [build](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json#L11) section describes how to build the container, i.e. it must be built from the source repository [github.com/slsa-framework/oss-na24-slsa-workshop-project1](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json#L14) by builder [github_generator_level_3](https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/blob/main/policies/publish/echo-server.json#L12).


Follow these steps:

1. Update the value of the package name with your container image, as built in [Activity 01](https://github.com/slsa-framework/oss-na24-slsa-workshop/blob/main/activities/01/readme.md).
1. Update the value  of the source respository as per [Activity 01](https://github.com/slsa-framework/oss-na24-slsa-workshop/blob/main/activities/01/readme.md).

##### Call the evaluator in CI

To evaluate the publish policy, the evaluator must be called from CI. It is up to teams to decide _when_ to do that. In this activity, we provide a helper workflow
[.github/workflows/publish-image.yml](https://github.com/slsa-framework/oss-na24-slsa-workshop-project1/blob/main/.github/workflows/publish-image.yml) that can be called manually for testing purposes.
In practice, teams would call it automatically after building their containers.

If the policy evaluation succeeds, the evaluator creates a publish attestation and signs it with [Sigstore](sigstore.dev). The attestation is stored along the container on the registry, [a-la-cosign](https://github.com/sigstore/cosign). NOTE: SLSA does not prescribe where to store the provenance, nor does it prescribe the use of Sigstore for signing.

Follow these steps:

1. Update the [organization workflow call](https://github.com/slsa-framework/oss-na24-slsa-workshop-project1/blob/main/.github/workflows/publish-image.yml#L37) that evaluates the publish policy.
1. Update the [registry-username](https://github.com/slsa-framework/oss-na24-slsa-workshop-project1/blob/main/.github/workflows/publish-image.yml#L43) to yours.
1. (Already done in Activity 01): Create a [docker regitry token](https://docs.docker.com/security/for-developers/access-tokens/#create-an-access-token) with read, write and delete access. 
1. (Already done in Activity 01): Store your docker token as a new GitHub repository secret called `REGISTRY_PASSWORD`: [Settings > New repository secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository).
1. Run the workflow via the [GitHub UI](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow). It will take ~40s to complete. If all goes well, the workflow run will display a green icon.


##### Verify publish attestation manually

To verify the publish attestation and inspect it, you can use cosign. Install [cosign](https://github.com/slsa-framework/oss-na24-slsa-workshop/blob/main/INSTALLATION.md#cosign).

Make sure you have access to your image by authenticating to docker:

```shell
$ REGISTRY_TOKEN=<your-token>
$ REGISTRY_USERNAME=<registry-username>
$ docker login -u "${REGISTRY_USERNAME}" "${REGISTRY_TOKEN}"
```

To verify a publish attestation, use the following command:

```shell
# Update the image as recorded in your logs
$ image=docker.io/laurentsimon/oss-na24-slsa-workshop-project1-echo-server@sha256:4004ae316501b67d4d2f7eb82b02f36f32f91101cc9a53d5eb4dd044c16a552e
# Update the repository name storing your policies.
$ creator_id="https://github.com/slsa-framework/oss-na24-slsa-workshop-organization/.github/workflows/image-publisher.yml@refs/heads/main"
$ type=https://slsa.dev/publish/v0.1
$ path/to/cosign verify-attestation "${image}" \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    --certificate-identity "${creator_id}" 
    --type "${type}" | jq -r '.payload' | base64 -d | jq
```

### Do it at home

#### Set up ACLs

Remember to try setting up the [protection ACLs to protect the policy and allow teams to edit the files they own](https://github.com/laurentsimon/slsa-policy/blob/main/README.md#org-setup).

#### Pre-submits for CODEOWNER

We must ensure that new team policy files are accompanied by a new CODEOWNER file. If you implement this feature, please share it with us!

### UX improments

In this Activity, users need to explicitly call the publish policy evaluator from CI. We may improve UX by calling the evaluator automatically instead:

1. If your organization has an internal registry where teams publish their containers / artifacts, the registry itself may evaluate the publish policy.
2. Integration with tooling: The package manager calls the evaluator. For example, docker CLI could be extended with a `--publish-policy-evaluator-url` to allow users to provide the URL of the evaluator. Users would run `docker push ${image} --publish-policy-evaluator-url=https://publish.mycompany.com/policy`. This would require standardizing the interface to the evaluator. Package registries may provide a mechanism for maintainers to enforce that only packages that have a valid publish attestation are published. This effectively allows organizations to ensure that their published software meet the requirements set by the publish policy. Integration with tooling is a flexible long-term solution.

### Additional artifact properties

In this demo, we have focused on integrity propertion of the published artifact, i.e., its SLSA level. Can you add support for other properties?

## Take the quizz!

After completing this activity, you should be able to answer the following questions:

1. What is a publish policy?
2. What invariant is enforced across all policy files? Why is it necessary?
3. What are trusted roots? Who configures them?
4. What is a publish attestation? Who creates it? What information does it contain?
5. What artifact properties may be useful to report in the attestation beyong its SLSA level?
6. What metadata is needed to verify a publish attestation?
7. What improvements can we make to improve UX for teams?
