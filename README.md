# Repository transfer issue ops

This repository provides an issue-ops workflow for transferring repositories within an Enterprise Managed Users (EMU) enterprise:

- from one organization to another organization

Repositories in users' personal spaces are not supported. Both the source and destination repository owners must be organizations.

Users submit a GitHub issue form with:

- source repository: `organization/repository`
- destination repository: `organization/repository`
- destination visibility: `internal` or `private` (`internal` by default)
- whether the transferred repository should be archived or unarchived

The workflow uses a user personal access token to validate repositories, perform the transfer, update visibility, and optionally archive or unarchive the transferred repository.

This solution assumes it is deployed in a GitHub Enterprise Cloud environment with Enterprise Managed Users.

## Repository contents

- `/README.md` - deployment and usage documentation
- `/.github/ISSUE_TEMPLATE/repository-transfer.yml` - issue form
- `/.github/workflows/repository-transfer.yml` - automation workflow

## How it works

1. A user opens a **Repository transfer request** issue.
2. The workflow parses the issue body.
3. The workflow verifies that the source repository exists and is accessible to the configured transfer PAT.
4. The workflow verifies that the destination repository does not already exist.
5. The workflow transfers the source repository to the destination owner and name.
6. The workflow waits for the transferred repository to become available at the destination.
7. If requested, the workflow unarchives the repository.
8. The workflow updates visibility to `internal` or `private`.
9. If requested, the workflow archives the repository. This runs after unarchiving, so archive takes precedence when both options are selected.
10. The workflow comments on the issue and closes it on success.

## Transfer token requirements

Configure a user personal access token that belongs to an account with the access required to complete transfers.

Required token type and scopes:

- **Classic PAT (recommended):**
  - `repo`
  - `admin:org` when the destination organization's transfer rules require organization-owner level authority from the account associated with the PAT
- **Fine-grained PAT:**
  - repository access to each source repository that may be transferred
  - repository permission **Administration: Read and write**
  - repository permission **Metadata: Read-only**

The workflow has been validated with the classic PAT model. If you use a fine-grained PAT, the account associated with the PAT must still have the necessary rights in the source and destination organizations for the transfer to succeed.

Required capabilities for the account associated with the PAT:

- admin access to each source repository that may be transferred
- sufficient rights in each destination organization to receive the transfer
- access to read the source repository and update the transferred repository after it arrives

## Repository secrets

Add these repository or organization secrets to the issue-ops repository:

- `REPO_TRANSFER_PAT` - a user personal access token for an account that can transfer the source repository into the destination organization

## Deploying the solution

1. Create a personal access token for the user account that will perform repository transfers.
2. Ensure that account has the access described above for the source repositories and destination organizations it will handle.
3. Add the workflow credential as a repository secret:
   - `REPO_TRANSFER_PAT`
4. Create the issue label `repository-transfer` in the issue-ops repository.
5. Push this repository's workflow and issue template to the repository that will host the issue-ops process.
6. Confirm GitHub Actions are enabled in the issue-ops repository.

## Usage

1. Open the **Repository transfer request** issue form.
2. Enter the source repository as `organization/repository`.
3. Enter the destination repository as `organization/repository`.
4. Choose the destination visibility:
   - `internal` (default)
   - `private`
5. Select the archive or unarchive option if the repository's archive state should change after transfer. If both are selected, the repository will be archived.
6. Submit the issue.

The workflow will:

- validate the form values
- verify that the source repository exists and that the destination repository name is still available
- transfer the repository with the configured user PAT
- set the requested visibility with the configured user PAT
- optionally unarchive or archive the repository with the configured user PAT, with archive taking precedence
- comment with the result

The workflow only runs for issues that have the `repository-transfer` label. The included issue form applies that label automatically, so the label must already exist in the issue-ops repository before users submit requests.

## Operational considerations

- This solution assumes GitHub Enterprise Cloud with Enterprise Managed Users.
- `internal` visibility is supported for organization-owned repositories in this environment and remains the default option.
- Both the source and destination repository owners must be organizations.
- Repositories cannot be transferred from users' personal spaces with this tool.
- `REPO_TRANSFER_PAT` must belong to a user account that can administer the source repository and complete the transfer into the destination organization.
- This workflow is intentionally limited to validation and transfer operations; it does not provision or install the app during the run.
- If the destination repository name already exists, the transfer will fail.
- The workflow reacts to issue `opened`, `edited`, and `reopened` events for issues created from the transfer template.

## Troubleshooting

Common causes of failure:

- `REPO_TRANSFER_PAT` is missing or belongs to an account that cannot complete the transfer
- the source repository is in a user's personal space rather than an organization
- the source repository does not exist
- the destination organization blocks the transfer
- the destination repository name already exists
