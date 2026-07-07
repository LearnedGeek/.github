# LearnedGeek — organization defaults

This `.github` repository holds LearnedGeek-wide defaults that inherit
into every repo in the org that doesn't override them: shared workflows,
starter workflow templates, and this profile README (which renders on
[github.com/LearnedGeek](https://github.com/LearnedGeek)).

## What's here

| Path | What it is |
|---|---|
| `profile/README.md` | The public org profile that renders on the org home page. |
| `.github/workflows/release-drafter.yml` | Reusable Release Drafter workflow (`on: workflow_call`) invoked by consuming repos. |
| `.github/workflow-templates/` | Starter workflow templates. Show up in every org repo's Actions → New workflow → "By LearnedGeek" section. |
| `templates/release-drafter/config.yml` | Canonical Release Drafter config to copy into consuming repos as `.github/release-drafter.yml`. Not a workflow template — those don't cover non-workflow files. |

## Onboarding a new repo to Release Drafter

Three steps:

1. **Add the caller workflow.** In the consuming repo, Actions → New workflow → pick "Release Drafter (LearnedGeek)" under the "By LearnedGeek" section. That creates `.github/workflows/release-drafter.yml` with the caller stub.
2. **Add the config.** Copy `templates/release-drafter/config.yml` from this repo into the consuming repo as `.github/release-drafter.yml`. Edit if the repo needs different category/version rules; keep it default otherwise.
3. **Add the repo to Terraform label management.** In `learnedgeek-infra/github-labels/variables.tf`, append the repo name to `var.managed_repos`. Run `terraform plan` + `terraform apply`. The 12 canonical labels (feature, fix, chore, docs, dependencies, breaking, major, minor, patch, enhancement, bug, documentation) appear with matching colors and descriptions.

After the first PR merges with a category label applied, Releases → Draft will start populating.

## Managing the canonical label set

Labels live in [learnedgeek-infra/github-labels](https://github.com/LearnedGeek/learnedgeek-infra/tree/main/github-labels), not here. Any change to color, description, or set membership goes through Terraform + PR. Editing a canonical label directly in the GitHub UI on a managed repo will be reverted on the next `terraform apply` — that's the whole point of centralizing them.
