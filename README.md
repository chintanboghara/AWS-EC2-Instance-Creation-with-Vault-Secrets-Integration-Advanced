# AWS EC2 Instance Creation with Vault Secrets Integration Advanced

This Terraform configuration demonstrates how to use the AWS and Vault providers to securely fetch a secret from a Vault server and create an AWS EC2 instance with that secret embedded as a tag.

## Prerequisites

- Terraform installed (version 0.13.0 or later recommended)
- An AWS account with permissions to create EC2 instances
- A Vault server with:
  - A KV v2 secrets engine mounted (default: `kv`)
  - A secret stored at the specified path (default: `secret`) with a key like `username`
  - An AppRole configured with policies to read the secret

## Configuration

### AWS Provider

The AWS provider is configured to use a specified region, defined by the variable `aws_region`:

```hcl
provider "aws" {
  region = var.aws_region
}
```

Ensure your AWS credentials are configured (e.g., via environment variables or the AWS CLI).

### Vault Provider

The Vault provider is set up to connect to a Vault server using AppRole authentication. The address, role ID, and secret ID are provided via variables:

```hcl
provider "vault" {
  address         = var.vault_address
  skip_child_token = true

  auth_login {
    path = "auth/approle/login"
    parameters = {
      role_id   = var.vault_role_id
      secret_id = var.vault_secret_id
    }
  }
}
```

- `vault_address`: The URL of your Vault server (e.g., `http://vault.example.com:8200`)
- `vault_role_id` and `vault_secret_id`: AppRole authentication credentials

The `skip_child_token = true` setting uses the same token for all requests.

### Secrets in Vault

Secrets are fetched from a KV v2 engine using the specified mount and secret name:

```hcl
data "vault_kv_secret_v2" "example" {
  mount = var.vault_kv_mount
  name  = var.vault_secret_name
}
```

- `vault_kv_mount`: The mount point of the KV engine (default: `kv`)
- `vault_secret_name`: The name of the secret (default: `secret`)

Ensure the secret includes a key like `username` for use as a tag.

### EC2 Instance Configuration

The EC2 instance is created with a specified AMI and instance type, and the secret is applied as a tag:

```hcl
resource "aws_instance" "my_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name   = "DemoEC2Instance"
    Secret = data.vault_kv_secret_v2.example.data["username"]
  }
}
```

- `ami_id`: The AMI ID for the instance (default: `ami-053b0d53c279acc90`)
- `instance_type`: The instance type (default: `t2.micro`)

Verify the AMI ID is valid for your region.

## Usage

1. **Set Variable Values**:
   - Provide values for `vault_address`, `vault_role_id`, and `vault_secret_id`.
   - Optionally, override defaults for `aws_region`, `vault_kv_mount`, `vault_secret_name`, `ami_id`, and `instance_type`.

   You can set variables in a `terraform.tfvars` file or via command-line flags. For example:

   ```hcl
   # terraform.tfvars
   vault_address   = "http://vault.example.com:8200"
   vault_role_id   = "your-role-id"
   vault_secret_id = "your-secret-id"
   ```

   **Security Note:** Handle sensitive values like `vault_secret_id` securely. Consider using environment variables or a secrets manager.

2. **Initialize Terraform**:
   ```bash
   terraform init
   ```

3. **Plan the Deployment**:
   ```bash
   terraform plan
   ```

4. **Apply the Configuration**:
   ```bash
   terraform apply
   ```
   Confirm when prompted.

5. **Verify the EC2 Instance**:
   - In the AWS Management Console, go to the **EC2 Dashboard**.
   - Select the instance and check the **Tags** tab for the `Secret` tag.
   - Alternatively, use the AWS CLI:
     ```bash
     aws ec2 describe-instances --instance-ids <instance_id> --query 'Reservations[].Instances[].Tags'
     ```

## Outputs

- `ec2_instance_id`: The ID of the created EC2 instance.
- `vault_secret`: The secret retrieved from Vault (for verification).

## Security Note

Tags are visible in the AWS console and not encrypted. Avoid using sensitive secrets in tags in production.

## Cleanup

Remove all resources with:
```bash
terraform destroy
```

## Troubleshooting

- **Vault Errors**: Check AppRole credentials and policies.
- **AWS Errors**: Confirm credentials have EC2 permissions.
- **AMI Issues**: Ensure the AMI ID is valid for the region.
