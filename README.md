# iamdeny-next2025
IAM Deny Repo for Next 2025

# Terraform Google Cloud IAM Deny and Organization Policies

## Description

This Terraform configuration sets up a series of security guardrails within a Google Cloud organization using IAM Deny Policies and Organization Policies. It aims to restrict specific high-privilege permissions and enforce organizational standards at both the organization and a designated folder level.

Key components include:
* An organization-level IAM Deny Policy targeting specific administrative permissions on resources tagged with `iam_deny=enabled`.
* A folder-level IAM Deny Policy restricting Billing, Security (including numerous Security Command Center permissions), and Networking permissions on resources *unless* they have any tag applied.
* A Custom Organization Policy Constraint to prevent the use of the primitive `roles/owner` role.
* An Organization Policy restricting the usage of specific Google Cloud services (`securitycenter.googleapis.com`, `accessapproval.googleapis.com`) within a designated folder.

## Features

* Applies granular IAM Deny policies based on permissions defined in external JSON files (`billing.json`, `networking.json`, `securitycenter.json`) and an internal list (`denied_perms.tf`).
* Utilizes resource tags (`iam_deny=enabled`) to conditionally apply the organization-level deny policy.
* Applies folder-level deny policies based on the *absence* of any resource tags, covering Billing, Networking, and Security Center permissions.
* Provides exceptions for specific principals (e.g., dedicated groups for networking, billing, security) for each deny policy rule.
* Enforces a custom constraint against the `roles/owner` role.
* Restricts specific service usage within a target folder using a standard Organization Policy constraint.

## Prerequisites

1.  **Terraform:** Terraform CLI (version compatible with provider requirements) installed.
2.  **Google Cloud Provider:** Configured authentication for the Terraform Google providers (e.g., via `gcloud auth application-default login` or Service Account key).
3.  **Permissions:** The identity running Terraform needs sufficient permissions in the target Google Cloud organization and folder, including:
    * `iam.denyPolicies.create`, `iam.denyPolicies.update`, `iam.denyPolicies.get` at the Organization and Folder levels.
    * `orgpolicy.policy.set`, `orgpolicy.customconstraint.create`, `orgpolicy.policy.get` at the Organization and Folder levels.
    * `resourcemanager.organizations.get`
    * `resourcemanager.folders.get`
    * Permissions to read resource tags (`resourcemanager.tagValues.get`, `resourcemanager.tagKeys.get` or broader).
4.  **Organization ID:** Your Google Cloud Organization ID.
5.  **Target Folder ID:** The ID of the specific Google Cloud Folder where folder-level policies will be applied.
6.  **Tag Setup:** The tag `iam_deny=enabled` (specifically `tagKeys/281482384217710` = `tagValues/281477230819536`) must exist in your organization for the organization-level deny policy condition to function correctly.
7.  **Permission Files:** Ensure the following JSON files exist in the `./profiles/` directory relative to your `main.tf`:
    * `billing.json`
    * `networking.json`
    * `securitycenter.json`

## Usage

1.  **Clone/Download:** Place these Terraform files (`main.tf`, `variables.tf`, `providers.tf`, `denied_perms.tf`) and the `profiles/` directory (containing `billing.json`, `networking.json`, `securitycenter.json`) in your working directory.
2.  **Configure Variables:**
    * Update the `default` values in `variables.tf` for `org_id` and `iamdeny_folder_id`.
    * Populate the `default` lists in `variables.tf` for `networking_exception_principals`, `billing_exception_principals`, `sec_exception_principals`, and `top_exception_principals` with the appropriate principal identifiers (e.g., `principalSet://goog/group/your-group@example.com`).
    * Alternatively, create a `terraform.tfvars` file or use command-line flags (`-var="org_id=YOUR_ORG_ID"`) to provide these values.
3.  **Initialize Terraform:**
    ```bash
    terraform init
    ```
4.  **Review Plan:**
    ```bash
    terraform plan
    ```
5.  **Apply Configuration:**
    ```bash
    terraform apply
    ```

## Resources Created

This configuration will create the following Google Cloud resources:

* **`google_iam_deny_policy.top_level_deny`**: An IAM Deny Policy attached at the organization level, denying a broad set of permissions on resources tagged `iam_deny=enabled`.
* **`google_iam_deny_policy.profile-deny-policy`**: An IAM Deny Policy attached at the folder level (`var.iamdeny_folder_id`), denying specific Billing, Security, and Networking permissions on resources without any tags.
* **`google_org_policy_custom_constraint.constraint`**: A Custom Organization Policy Constraint preventing the assignment of `roles/owner`.
* **`google_organization_policy` (via module `gcp_org_policy_v2`)**: An Organization Policy attached at the folder level (`var.iamdeny_folder_id`) enforcing `gcp.restrictServiceUsage` to deny specific services.

## Inputs

| Name                            | Description                                                                                                                | Type         | Default                  | Required |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------ | :------: |
| `org_id`                        | Your Google Cloud Organization ID.                                                                                         | `string`     | `""`                     |   Yes    |
| `iamdeny_folder_id`             | The folder ID where folder-level policies will be attached.                                                                | `string`     | `""`                     |   Yes    |
| `networking_exception_principals` | List of principals (e.g., groups) exempt from the networking deny rule. Format: `principalSet://goog/group/GROUP_EMAIL` | `list(string)` | `[""]`                   |    No    |
| `billing_exception_principals`  | List of principals (e.g., groups) exempt from the billing deny rule. Format: `principalSet://goog/group/GROUP_EMAIL`      | `list(string)` | `[""]`                   |    No    |
| `sec_exception_principals`      | List of principals (e.g., groups) exempt from the security deny rule. Format: `principalSet://goog/group/GROUP_EMAIL`     | `list(string)` | `[""]`                   |    No    |
| `top_exception_principals`      | List of principals (e.g., groups) exempt from the organization-level deny policy. Format: `principalSet://goog/group/GROUP_EMAIL` | `list(string)` | `[]`                   |    No    |
| `iamdeny_folder_path`           | The prefix for the folder resource path.                                                                                    | `string`     | `"cloudresourcema..."` |    No    |
| `region`                        | The default Google Cloud region for the provider.                                                                          | `string`     | `"us-central1"`          |    No    |
| `zone`                          | The default Google Cloud zone for the provider.                                                                            | `string`     | `"us-central1-c"`        |    No    |

*(Note: Defaults for `org_id`, `iamdeny_folder_id`, and exception principals likely need to be updated or provided via `.tfvars`)*

## Outputs

No outputs are defined in this configuration.

## Providers

| Name          | Version |
| ------------- | ------- |
| hashicorp/google | >= 5.0.0 |
| hashicorp/google-beta | >= 5.0.0 |

## Modules

| Name                | Source                                                     | Version |
| ------------------- | ---------------------------------------------------------- | ------- |
| gcp_org_policy_v2 | terraform-google-modules/org-policy/google//modules/org_policy_v2 | ~> 5.3.0 |

## License

This code is licensed under the Apache License, Version 2.0. See the license headers in the `.tf` files for details.