![](https://raw.githubusercontent.com/opentofu/brand-artifacts/main/full/transparent/SVG/on-dark.svg#gh-dark-mode-only)
![](https://raw.githubusercontent.com/opentofu/brand-artifacts/main/full/transparent/SVG/on-light.svg#gh-light-mode-only)

# Migrating OpenTofu to Massdriver

[OpenTofu](https://github.com/opentofu/opentofu) is an infrastructure-as-code tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share. It is hosted by the Linux Foundation.

This repo contains materials for the Massdriver webinar on migrating OpenTofu modules into Massdriver bundles.

## Prerequisites

- OpenTofu is installed on your system.
- You have a Massdriver account.
- You have the Massdriver CLI installed and configured.

## Step 1: Creating an S3 Bucket with OpenTofu

1. Navigate to the S3 bucket module:
    ```bash
    cd modules/s3-bucket
    ```

2. Initialize the OpenTofu configuration:
    ```bash
    tofu init
    ```

3. Plan the changes to see what will be applied:
    ```bash
    tofu plan
    ```

4. Apply the changes to create the S3 bucket:
    ```bash
    tofu apply
    ```

    - **Note:** If the command fails, it may be due to a naming conflict on the bucket. Make sure your bucket name is unique.

5. Destroy the bucket once you're done to clean up resources:
    ```bash
    tofu destroy
    ```

## Step 2: Bundling the OpenTofu Module in Massdriver

1. Navigate back to the root of your repository:
    ```bash
    cd ../../
    ```

2. Create a new Massdriver bundle:
    ```bash
    mass bundle new
    ```

    - **Name the bundle:** `webinar-s3-bucket`
    - **Type of bundle:** `opentofu-module`
    - **Import the OpenTofu module from:** `modules/s3-bucket`
    - **Add connection:** `aws-iam-role`
    - **Connection name:** `aws_authentication`

4. Review the files created by the bundle:
    - Open `massdriver.yaml` to review imported parameters and the specified AWS connection.
    - Open the `src` directory and modify `providers.tf`:
        - Copy `assume_role` block from generated `_providers.tf` and default tags into `providers.tf`.
        ```hcl
        provider "aws" {
            region     = var.region
            assume_role {
                role_arn    = var.aws_authentication.data.arn
                external_id = var.aws_authentication.data.external_id
            }
            default_tags {
                tags = var.md_metadata.default_tags
            }
        }
        ```
    - Delete the old `_providers.tf` file.

5. Publish the bundle to Massdriver:
    ```bash
    mass bundle publish
    ```

## Step 3: Deploying and Managing the Bundle on Massdriver

1. Log into [Massdriver Cloud](https://massdriver.cloud).
2. Drag the `webinar-s3-bucket` bundle onto the canvas and deploy it.
3. Decommission the bundle when done.

## Step 4: Migrating Local OpenTofu State to Massdriver

If you have existing infrastructure managed via IaC, you'll have existing state. This section shows how you can migrate state to Massdriver along with your IaC modules.

1. Navigate back to the S3 bucket module:
    ```bash
    cd ../modules/s3-bucket
    ```

2. Apply the OpenTofu configuration with local state:
    ```bash
    tofu apply
    ```

3. Set the following environment variables for migrating state to Massdriver. Be sure to replace `<package name>` with package name from Massdriver. Review the documentation on [managing state with Massdriver](https://docs.massdriver.cloud/guides/managing-state) for more details:
    ```bash
    export TF_HTTP_USERNAME=${MASSDRIVER_ORG_ID}
    export TF_HTTP_PASSWORD=${MASSDRIVER_API_KEY}
    export TF_HTTP_ADDRESS="https://api.massdriver.cloud/state/<package name>/src"
    export TF_HTTP_LOCK_ADDRESS=${TF_HTTP_ADDRESS}
    export TF_HTTP_UNLOCK_ADDRESS=${TF_HTTP_ADDRESS}
    ```

4. Uncomment the `backend` block in `providers.tf` and reinitialize:
    ```bash
    tofu init
    ```

5. Approve the state migration. The state should now be in Massdriver.

6. Redeploy the bundle on the Massdriver canvas and watch the logs. No resources should be added (just updates to bucket tags).

## Step 5: Modifying the OpenTofu Module

Part of maintaining IaC is performing updates to add/remove features and capabilities. In this section we'll modify the OpenTofu in our bundle and publish the changes.

1. Navigate to the bundle directory:
    ```bash
    cd ../../bundle
    ```

2. Open `main.tf` and modify the `force_destroy` parameter to be:
    ```hcl
    var.force_destroy
    ```

3. Open `variables.tf` and add the following variable declaration:
    ```hcl
    variable "force_destroy" {
      type        = bool
      default     = true
      description = "A boolean that indicates all objects should be deleted from the bucket so that the bucket can be destroyed without error."
    }
    ```

4. Run the Massdriver linter to identify issues:
    ```bash
    mass bundle lint
    ```

    - The linter will identify that there is a variable declared in IaC that isn't in `massdriver.yaml`. Let's import it.

5. Run `mass bundle import` command to import changes:
    ```bash
    mass bundle import
    ```

    - This updates `massdriver.yaml` with the new variable automatically.

6. Publish the updated bundle:
    ```bash
    mass bundle publish
    ```

7. Log into Massdriver, refresh the page, and verify the new configuration:
    - Deploy the updated bundle.