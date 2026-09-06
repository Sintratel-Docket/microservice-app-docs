# Working Guidelines: Infrastructure with Terraform

This document defines how we are going to work on the infrastructure as code part of Docket. It is not an exhaustive Terraform manual, it is the set of rules everyone must follow so that each person's work stays consistent with the rest of the team and with what the project expects. Read it before writing the first line of infrastructure code.

## 1. Two repositories, not one

The infrastructure code lives in two separate repositories inside the organization. One is called `terraform-modules`, and it only contains the reusable modules: the network module, the Kubernetes cluster module, the database module, and so on. None of these modules know anything about a specific environment, they simply receive parameters and create resources.

The other repository is the one that actually spins up the infrastructure, for example `docket-infra`. It has one folder per environment: `environments/dev`, `environments/staging`, and `environments/prod`. Each of these folders calls the modules from the first repository, specifying which version to use and with what values.

## 2. Modules are versioned like any other library

Every time we make an important change to a module, we tag it in Git with a version following semantic versioning: v1.0.0, v1.1.0, v2.0.0, depending on whether the change is small, adds something new without breaking anything, or breaks compatibility with what existed before.

When an environment uses a module, it must always point to a specific tag, never to the main branch. This is key: if someone improves the network module tomorrow, that change should not automatically affect production. Each environment decides when to update its version.

## 3. Internal structure of a module

Every module must have at least these files: `main.tf` with the resources, `variables.tf` with the inputs and their descriptions, `outputs.tf` with what the module exposes to the outside, and `versions.tf` where the Terraform version and the providers used are pinned. It's also a good idea to add a short `README.md` explaining in a few lines what the module does, what it receives, and what it returns. If someone else on the team needs to use the module, that README should be enough for them to understand it without having to read the whole codebase.

## 4. No hardcoded values in the code

No resource name, instance size, network range, or environment-specific data can be written directly inside a `.tf` file. All of that is handled as a variable. Each environment has its own variables file, for example `dev.tfvars`, `staging.tfvars`, and `prod.tfvars`, with the values that apply to it. That way the same code works for all three environments, the only thing that changes is those files.

## 5. Terraform state is untouchable and separated by environment

The state file is never stored in the repository, nor does it stay only on someone's machine. We use a remote backend to store it, and each environment has its own state, completely independent from the others. This is fundamental: a mistake while applying changes in dev should never be able to touch the state of production, because they are literally different files in different places. We also enable state locking, so two people can't apply changes to the same environment at the same time.

## 6. Secrets are never written anywhere in the code

Passwords, access keys, or any credential are never written in a `.tf` file or in a `.tfvars` file that gets pushed to the repository. Any file that could contain sensitive information gets added to `.gitignore` from day one. Sensitive values are injected through environment variables or a secrets manager, they never end up written in plain text in any file Git can see.

## 7. Consistent naming and tagging

All resources must follow the same naming pattern, including the project and the environment they belong to, for example `docket-dev-cluster` or `docket-prod-db`. Resources must also carry tags indicating the project, the environment, and who manages them. This isn't just for tidiness, it will help us later to identify costs and for any work related to cloud spend optimization.

## 8. Format and validate before pushing any change

Before pushing a change, run `terraform fmt` so the indentation and style stay consistent across the code, no matter who wrote it. Also run `terraform validate` to confirm the syntax is correct. These two commands take seconds and avoid unnecessary formatting discussions during code review.

## 9. Always plan before apply, never manual changes

No `apply` is run without first calmly reviewing the output of `terraform plan`. That's where you see exactly what will be created, modified, or destroyed, and it's the moment to catch something that shouldn't happen. Manual changes through the cloud provider's console are not allowed either. If something gets changed outside of Terraform, the code stops reflecting reality, and over time that becomes impossible to track.

## 10. Controlled teardown

It must be documented how to tear down the infrastructure in an orderly way, environment by environment, without leaving orphaned resources behind. For the most critical resources, such as databases, the `prevent_destroy` protection is enabled, so they can't be accidentally deleted with a single command.

---

These ten rules are the foundation we'll use to work on all the project's infrastructure. Any doubt about how to apply one of them to a specific case should be discussed with us before moving forward, not after.
