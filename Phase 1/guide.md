## Step 1: Use Claude to Generate Terraform Code

1. Start a conversation with Claude.
2. Ask Claude to create Terraform code for an S3 bucket. Use a prompt like:
"Please provide Terraform code to create an S3 bucket in AWS with a unique name."
3. Claude should generate code similar to this:
'''
provider "aws" {
  region = "us-west-2"  # Replace with your desired region
}

resource "random_id" "bucket_suffix" {
  byte_length = 8
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}

resource "aws_s3_bucket_acl" "my_bucket_acl" {
  bucket = aws_s3_bucket.my_bucket.id
  acl    = "private"
}
'''

## Step 2: Create IAM Role for EC2

1. Log in to the AWS Management Console.
2. Navigate to the IAM dashboard.
3. Click "Roles" in the left sidebar, then "Create role".
4. Choose "AWS service" as the trusted entity type and "EC2" as the use case.
5. Search for and attach the "AdministratorAccess" policy.
Note: In a production environment, use a more restricted policy.
6. Name the role "EC2Admin" and provide a description.
7. Review and create the role.
