# 03 - Deployment policy

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### What you will need

To complete this activity, you need:

1. To have read the policy part of the [slides](https://docs.google.com/presentation/d/1w3AWWdXQ8ePoT50R6Ujs-Ji_aXGBa1HmxHBcQIGgH2Q).
1. A GitHub account
1. A docker registry account (alternatively you can use GitHub registry)

### Deployment policy and attestation

A deployment policy expresses which environment an artifact is allowed to be deployed to. Binding an artifact to its expected deployment environment is one of the principles used internally at Google; it is also a feature provided by [Google Cloud Binauthz](https://cloud.google.com/binary-authorization/). A deployment environment may be a cloud machine where we deploy containers, a developer workstation where we deploy packages (pip, npm, etc.) or even an Android device where we deploy applications. 

When deploying an artifact (e.g., a container, a smartphone app), we want to restrict which environment the artifact is allowed to be deployed / run. The environment has access to resources we want to protect, such as a cloud service account, a Spiffe ID, a Kubernetes pod ID, an SEAndroid context, etc. The deployment attestation authoritatively binds an artifact to a (set of) deployment environment(s) where an artifact is allowed to be deployed.

The ability to bind an artifact to an environment is paramount to reduce the blast radius if vulnerabilties are exploited or environments are compromised. Attackers who gain access to an environment will pivot based on the privileges of this environment, so it is imperative to follow the privilege of least principle and restrict which code is allowed to run in which environment. For example, we would not want to deploy a container with remote shell capabilities on a pod that processes user credentials, even if this container is integrity protected at the highest SLSA level. Conceptually, this is similar to how we think about sandboxing and least-privilege principle on operating systems. The same concepts apply to different types of environments, including cloud environments.

The decision to allow or deny a deployment request may happen in "real-time", i.e. the control plane may query an online authorization service at the time of the deployment. Such an authorization service requires low-latency / high-availability SLOs to avoid deployment outage. This is exacerbated in systems like Kubernetes where admission webhooks run for every pod deployed. Thus it is often desirable to "shift-left" and perform an authorization evaluation ahead of time before a deployment request reaches the control plane. The deployment attestation is the proof of authorization that the control plane may use to make its final decision, instead of querying an online service itself. Verification of the deployment attestation is simple, fast and may be performed entirely offline. Overall, this shift-left strategy provides the following advantages: less likely to cause production issues, better debugging UX for devs, less auditing and production noise for SREs and security teams.

In this activity, we will define a deployment environment by its google service account. This will help us achieve the second goal of the workshop, which is to ensure that all deployed containers run with a defined set of privileges, the same way that OS processes are restricted to a set of running privileges (Service accounts have IAM permissions).

In this workshop, due to time constraint, we will focus on the use case of teams deploying containers that they build. Deployment of containers built by other teams is left as [future work](#deployment-of-other-teams-artifacts).

## Deep dive

### Deployment policy setup

The deployment policy is stored alongside the release policy. The policy is stored in a central location administered by the organization.
This gives the organization a central view of all the projects and their configuration. In this activity, we store the policy in a GitHub repository owned by the organization. Like the release policy, the deployment policy is sub-devided into:

1. A configuration maintained by the organization which applies to all teams within the organization
2. Team-specific configuration maintained by teams for their project.

#### Repository protections

As described in the previous paragraph, the organization stores all policy configurations in a central location. To reduce the bottleneck on the organization admins,
it is important for team policies to be editable _without_ admin intervention. We need to enable teams to review and edit changes on their own, while
preventing unauthorized changes from other teams. Similarly, the configuration maintained by the organization must be protected again unauthorized
changes by other teams. We can echieve these protections by setting up [branch protection rules](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/managing-rulesets-for-a-repository) and [CODEOWNER settings](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#additional-settings). To simplify the workshop and due to time constraint,
we will assume these protections are in place. If you wish to implement these protections after the workshop, refer to the steps described in the [policy-setup](https://github.com/laurentsimon/slsa-policy?tab=readme-ov-file#policy-setup-1).

#### Organization roots

(Already done in [Activity 02](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/02/readme.md)): Fork this repository [https://github.com/laurentsimon/oss-na24-slsa-workshop-organization](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization) by clicking this [link](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/fork).

Under [policies/deployment](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/tree/main/policies/deployment) are the configuration files for the deployment policy. The file maintained by the organization admins
is [org.json](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/org.json). This file contains a list of "trusted roots", which is a list of trusted entities. In this demo,
each trusted root is a "releaser" identity allowed to evaluate the release policy and generate release attestations for the organization. For example, the [only listed releaser](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/org.json#L6) has `id:https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/.github/workflows/image-releaser.yml@refs/heads/main` and is trusted to claim up to `max_slsa_level:3`. NOTE: It is important the releaser identity include the reference `refs/heads/main`, since other branches may _not_ be protected.

#### Evaluator service

The repository contains a GitHub workflow [.github/workflows/image-deployer.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-deployer.yml) which evaluates the deployment policy. The workflow follows the same structure as the release policy. It contains the following logic:

1. [Detects the refs](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-deployer.yml#L47-L67) at which it was called by a project. This is due to a quirk of how GitHub reusable workflows work. You can ignore this part of the code.
1. [Install the policy CLI](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-deployer.yml#L106-L109).
1. [Run the policy CLI](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/image-deployer.yml#L110-L120).

#### Pre-submits

Across the policy, there is an important invariant to maintain, which is that a service account must be owned by at most _one_ team. In other words, we must ensure that across the policy, a service account is only referenced once across all configuration files. For this, we make use of pre-submits run on pull requests.
The pre-submits are configured in the workflow file [.github/workflows/pre-submit.deployment-policy.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/.github/workflows/pre-submit.deployment-policy.yml). NOTE: Unlike the release policy, it is fine for any team to reference / use a package, even if it is owned and built by another team. This is by design, i.e., we want anyone in the organization to be able to take another team's artifact / container as a dependency.

An additional required pre-submit is to ensure that new team policy files are accompanied by a new CODEOWNER file. We leave this as [future work](#pre-submits-for-codeowner).

### Team setup

#### Project ownership

To ensure only the team that owns a service account is allowed to edit its configuration, the team must edit or create a CODEOWNER file that give them ownership of the configurations (files or entire directories).
As explained in [repository protections](#repository-protections), for time constraints in this workshop we will assume this is done, but you can set it up [after the workshop](#set-up-acls).

##### Configure the policy

The two files to be protected by the CODEOWNER file are prod's [servers-prod.json](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-prod.json) and staging's [servers-staging](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-staging.json). The prod file describes the team policy to deploy under google service account [name@prod-project-id.iam.gserviceaccount.com](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-prod.json#L4). The staging file is similar but is for the staging environment that runs under service account [name@staging-project-id.iam.gserviceaccount.com](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-staging.json#L4). Each file contains the following sections:

1. A [google_service_account](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-prod.json#L4) section describes the service account deployed containers are allowed to run under. NOTE: The environment (prod, staging) must match the one used for the release policy. In other words, if a container was released for "staging", the deployment policy must contain the same environment value "staging".
1. A [SLSA level for the builder](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-prod.json#L7). NOTE: The deployment policy uses release attestations to determine the SLSA levels.
1. A [list of packages](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-prod.json#L9-L33) allowed to run under the service account.

Follow these steps:

1. Update the list of [packages](https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/blob/main/policies/deployment/servers-prod.json#L11) using the container you built in [Activity 01](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/01/readme.md) and released in [Activity 02](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/02/readme.md).

##### Call the evaluator in CI

To evaluate the deployment policy, the evaluator is called from CI in this demo. It is up to teams to decide _when_ to do that. In this activity, we provide a helper workflow
[.github/workflows/deploy-image.yml](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/.github/workflows/deploy-image.yml) that can be called manually for testing purposes. In practice, teams would call it automatically before sending a depoyment request to their admission controller.

If the policy evaluation succeeds, the evaluator creates a deployment attestation and signs it with [Sigstore](sigstore.dev). The attestation is stored along the container on the registry, [a-la-cosign](https://github.com/sigstore/cosign). NOTE: SLSA does not prescribe where to store attestations, nor does it prescribe the use of Sigstore for signing.

Follow these steps:

1. Update the [organization workflow call](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/.github/workflows/deploy-image.yml#L41) that evaluates the deployment policy.
1. Update the [registry-username](https://github.com/laurentsimon/oss-na24-slsa-workshop-project1/blob/main/.github/workflows/deploy-image.yml#L47) to yours.
1. (Already done in Activity [01](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/01/readme.md) and [02](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/02/readme.md)): Create a docker regitry token (with push access), see [here](https://docs.docker.com/security/for-developers/access-tokens/#create-an-access-token). 
1. (Already done in Activity [01](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/01/readme.md) and [02](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/02/readme.md)): Store your docker token as a new GitHub repository secret called `REGISTRY_PASSWORD`: [Settings > New repository secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository).
1. Run the workflow via the [GitHub UI](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow). It will take ~40s to complete. If all goes well, the workflow run will display a green icon.

##### Verify deployment attestation manually

To verify the deployment attestation and inspect it, you can use cosign. To install it, follow the [instructions](https://github.com/sigstore/cosign?tab=readme-ov-file#installation).

Make sure you have access to your image by authenticating to docker:

```shell
$ REGISTRY_TOKEN=<your-token>
$ REGISTRY_USERNAME=<registry-username>
$ docker login -u "${REGISTRY_USERNAME}" "${REGISTRY_TOKEN}"
```

To verify a deployment attestation, use the following command:

```shell
# Update the image as recorded in your logs
$ image=docker.io/laurentsimon/oss-na24-slsa-workshop-project1-echo-server@sha256:4004ae316501b67d4d2f7eb82b02f36f32f91101cc9a53d5eb4dd044c16a552e
# Update the repository name storing your policies.
$ creator_id="https://github.com/laurentsimon/oss-na24-slsa-workshop-organization/.github/workflows/image-deployer.yml@refs/heads/main"
$ type=https://slsa.dev/deployment/v0.1
$ path/to/cosign verify-attestation "${image}" \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    --certificate-identity "${creator_id}" 
    --type "${type}" | jq -r '.payload' | base64 -d | jq
```

The command above only verifies the authenticity of the attestation, i.e., that it was created by the right entity (the reusable workflow). In practice, before a container is deployed, the admission controller must also verify each `scope` field against the deployment environment. In our demo, the google service account must be compared to the service account the pod is running under. We will do that in [Activity 04](https://github.com/laurentsimon/oss-na24-slsa-workshop/blob/main/activities/04/readme.md).

### Do it at home

#### Set up ACLs

Remember to try setting up the protection ACLs to protect the policy and allow teams to edit the files they own. See [here](https://github.com/laurentsimon/slsa-policy/blob/main/README.md#org-setup) for details.

#### Pre-submits for CODEOWNER

We must ensure that new team policy files are accompanied by a new CODEOWNER file. If you implement this feature, please share it with us!

#### Deployment of other team's artifacts

In this demo, the attestations are stored along the container. This means that to store the deployment attestation, the team calling the evaluator needs write access to the registry, so it will not work if you try to deploy an image that you do not own since you will not have write access to the registry account. The workaround is to create an organization registry account on docker, and use that to store all attestations. You will need to follow these steps:

1. Update the [Sign function](https://github.com/laurentsimon/slsa-policy/blob/main/cmd/evaluator/internal/deployment/evaluate/evaluate.go#L91) used to sign the deployment attestation. This function is also used for signing the release attestation, but we should not change the logic for the latter. You will need to add `RegistryClientOpts` to  [cosign.CheckOpts](https://github.com/laurentsimon/slsa-policy/blob/main/cmd/evaluator/internal/utils/crypto/crypto.go#L191-L199) - See [here](https://github.com/slsa-framework/slsa-verifier/blob/v2.5.1/verifiers/internal/gha/verifier.go#L275-L281) for example.
2. Add an option to the evaluator CLI.
3. Update your deployment evaluator to use the new option.
4. Share your code with us! We can merge it in [slsa-policy repository](https://github.com/laurentsimon/slsa-policy).

#### Challenge yourself

Can you update the policy engine to support other types of protections, e.g., GKE cluster ID, etc. When you implement this feature, make sure to think whether there are invariants your policy configuration needs For example, if you were to implement a protection for Kubernetes namespaces, you would not be able to enforce invariant on it, because namespaces are _not_ unique across projects.

### UX improments

In this Activity, users need to explicitly call the deployment policy evaluator from CI. We may improve UX by integrating the evaluation in gitops tooling such as ArgoCD or as a kubectl plugin. If you are interested in implementing such solution, let us know!

## Take the quizz!

After completing this activity, you should be able to answer the following questions:

1. What is a deployment policy?
2. What invariant is enforced across all policy files?
3. What are trusted roots? Who configures them?
4. What is a deployment attestation? Who creates it? What information does it contain?
5. What metadata is needed to verify a deployment attestation? What happens if the invariant from (2) were not satisfied? Hint: We need to add a policy URI field and verify it. TODO: Link to intoto attestation specs
6. What improvements can we make to improve UX for teams?
