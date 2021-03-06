---
title: Check out multiple repositories in your pipeline
description: Learn how to check out multiple repositories in your pipeline
ms.topic: reference
ms.date: 07/20/2020
monikerRange: "> azure-devops-2019"
---

# Check out multiple repositories in your pipeline

[!INCLUDE [version-team-services](../includes/version-team-services.md)]

Pipelines often rely on multiple repositories. You can have different repositories with source, tools, scripts, or other items that you need to build your code. By using multiple `checkout` steps in your pipeline, you can fetch and check out other repositories in addition to the one you use to store your YAML pipeline.

[!INCLUDE [temp](../../includes/feature-support-cloud-only.md)] 

## Specify multiple repositories

Repositories can be specified as a [repository resource](../yaml-schema.md#repository-resource), or inline with the `checkout` step. 

- [Repository declared using a repository resource](#repository-declared-using-a-repository-resource)
- [Repository declared using inline syntax](#repository-declared-using-inline-syntax)

Supported repositories are Azure Repos Git (`git`), GitHub (`github`), and Bitbucket Cloud (`bitbucket`).

The following combinations of `checkout` steps are supported.

- If there are no `checkout` steps, the default behavior is as if `checkout: self` were the first step.
- If there is a single `checkout: none` step, no repositories are synced or checked out.
- If there is a single `checkout: self` step, the current repository is checked out.
- If there is a single `checkout` step that isn't `self` or `none`, that repository is checked out instead of `self`.
- If there are multiple `checkout` steps, each designated repository is checked out to a folder named after the repository, unless a different `path` is specified in the `checkout` step. To check out `self` as one of the repositories, use `checkout: self` as one of the `checkout` steps.

### Repository declared using a repository resource

You must use a [repository resource](../yaml-schema.md#repository-resource) if your repository type requires a service connection or other extended resources field. You may use a repository resource even if your repository type doesn't require a service connection, for example if you have a repository resource defined already for templates in a different repository.

In the following example, three repositories are declared as repository resources. The [GitHub](../library/service-endpoints.md#sep-github) and [Bitbucket Cloud](../library/service-endpoints.md#sep-bbucket) repository resources require [service connections](../library/service-endpoints.md) which are specified as the `endpoint` for those repository resources. This example has four `checkout` steps, which check out the three repositories declared as repository resources along with the current `self` repository that contains the pipeline YAML.

```yaml
resources:
  repositories:
  - repository: MyGitHubRepo # The name used to reference this repository in the checkout step
    type: github
    endpoint: MyGitHubServiceConnection
    name: MyGitHubOrgOrUser/MyGitHubRepo
  - repository: MyBitbucketRepo
    type: bitbucket
    endpoint: MyBitbucketServiceConnection
    name: MyBitbucketOrgOrUser/MyBitbucketRepo
  - repository: MyAzureReposGitRepository
    type: git
    name: MyProject/MyAzureReposGitRepo

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- checkout: self
- checkout: MyGitHubRepo
- checkout: MyBitbucketRepo
- checkout: MyAzureReposGitRepository

- script: dir $(Build.SourcesDirectory)
```

If the `self` repository is named `CurrentRepo`, the `script` command produces the following output: `CurrentRepo  MyAzureReposGitRepo  MyBitbucketRepo  MyGitHubRepo`. In this example, the names of the repositories are used for the folders, because no `path` is specified in the checkout step. For more information on repository folder names and locations, see the following [Checkout path](#checkout-path) section.

### Repository declared using inline syntax

If your repository doesn't require a service connection, you can declare it inline with your `checkout` step.

> [!NOTE]
> GitHub and Bitbucket Cloud repositories require a [service connection](../library/service-endpoints.md) and must be declared as a [repository resource](#repository-declared-using-a-repository-resource).

```yaml
steps:
- checkout: git://MyProject/MyRepo # Azure Repos Git repository in the same organization
```

> [!NOTE]
> In the previous example, the `self` repository is not checked out. If you specify any `checkout` steps, you must include `checkout: self` in order for `self` to be checked out.

## Authorize resources

When you check out other Azure Repos Git repositories using multi-repo checkout, you may be prompted to authorize access to that resource before the pipeline runs for the first time. You can perform this authorization from the pipeline run summary page. 

:::image type="content" source="media/multi-repo-checkout/pipeline-resource-prompt.png" alt-text="This pipeline needs permission to access a resource":::

:::image type="content" source="media/multi-repo-checkout/authorize-resource-prompt.png" alt-text="Authorize resource":::

Choose **View** or **Authorize resources**, and follow the prompts to authorize the resources.

:::image type="content" source="media/multi-repo-checkout/waiting-for-review.png" alt-text="Waiting for review":::

:::image type="content" source="media/multi-repo-checkout/permit-access.png" alt-text="Permit access":::

For more information, see [Troubleshooting authorization for a YAML pipeline](../process/resources.md#troubleshooting-authorization-for-a-yaml-pipeline).


## Checkout path

Unless a `path` is specified in the `checkout` step, source code is placed in a default directory. This directory is different depending on whether you are checking out a single repository or multiple repositories. 

- **Single repository**: If you have a single `checkout` step in your job, (or you have no checkout step which is equivalent to `checkout: self`), your source code is checked out into a directory called `s` located as a subfolder of `(Agent.BuildDirectory)`. If `(Agent.BuildDirectory)` is `C:\agent\_work\1` then your code is checked out to `C:\agent\_work\1\s`.
- **Multiple repositories**: If you have multiple `checkout` steps in your job, your source code is checked out into directories named after the repositories as a subfolder of `s` in `(Agent.BuildDirectory)`. If `(Agent.BuildDirectory)` is `C:\agent\_work\1` and your repositories are named `tools` and `code`, your code is checked out to `C:\agent\_work\1\s\tools` and `C:\agent\_work\1\s\code`.
  
  > [!NOTE]
  > If no `path` is specified in the `checkout` step, the name of the repository is used for the folder,
  > not the `repository` value which is used to reference the repository in the `checkout` step.

If a `path` is specified for a `checkout` step, that path is used, relative to `(Agent.BuildDirectory)`.

> [!NOTE]
> If you are using default paths, adding a second repository `checkout` step changes the default path of the code for the first repository. For example, the code for a repository named `tools` would be checked out to `C:\agent\_work\1\s` when `tools` is the only repository, but if a second repository is added, `tools` would then be checked out to `C:\agent\_work\1\s\tools`. If you have any steps that depend on the source code being in the original location, those steps must be updated.

## Checking out a specific ref

The default branch is checked out unless you designate a specific ref.

If you are using inline syntax, designate the ref by appending `@<ref>`. For example:

```yaml
- checkout: git://MyProject/MyRepo@features/tools # checks out the features/tools branch
- checkout: git://MyProject/MyRepo@refs/heads/features/tools # also checks out the features/tools branch
- checkout: git://MyProject/MyRepo@refs/tags/MyTag # checks out the commit referenced by MyTag.
```

When using a repository resource, specify the ref using the `ref` property. The following example checks out the `features/tools/ branch.

```yaml
resources:
  repositories:
  - repository: MyGitHubRepo
    type: github
    endpoint: MyGitHubServiceConnection
    name: MyGitHubOrgOrUser/MyGitHubRepo
    ref: features/tools
```

