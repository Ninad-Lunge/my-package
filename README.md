![AWS_Vulnerability_Scanning](https://github.com/user-attachments/assets/6f23e94c-e4e4-49d6-9978-139a92b02086)

# Python SBOM Pipeline

## Purpose
This pipeline automates the generation and management of Software Bill of Materials (SBOM) for Python applications. An SBOM provides a detailed inventory of all components and dependencies in your software, enhancing security and compliance.

## Associated AWS Services
- AWS CodePipeline: Orchestrates the entire CI/CD workflow
- AWS CodeBuild: Builds the project and generates SBOM
- Amazon S3: Stores pipeline artifacts, SBOM files, and Snyk scan reports
- AWS CodeCommit/GitHub: Source code repository
- Amazon ECR: Stores container images (if containerized)
- AWS CodeArtifact: Manages and stores Python package dependencies
- Snyk: Performs security scanning and vulnerability assessment

## Pipeline Steps
1. **Source Stage**
   - Monitors the source repository for changes
   - Triggers pipeline execution on code commits

2. **Build Stage**
   - Sets up Python environment
   - Installs project dependencies
   - Generates SBOM using tools like CycloneDX or SPDX
   - Validates SBOM format and contents

3. **Analysis Stage**
   - Scans SBOM for security vulnerabilities using Snyk
   - Performs dependency vulnerability assessment
   - Checks for outdated dependencies
   - Generates detailed security and compliance reports
   - Stores scan reports in dedicated S3 bucket
   - Integrates with Snyk's security database for up-to-date vulnerability information

4. **Artifact Storage**
   - Uploads SBOM to S3 bucket
   - Archives build artifacts
   - Maintains version history

## Prerequisites
- AWS account with appropriate permissions
- Python project with requirements.txt or setup.py
- Source code repository configured
- AWS CLI installed and configured

## Setup Instructions
1. Configure AWS credentials and permissions
2. Create necessary S3 buckets for artifacts
3. Set up CodeBuild project with appropriate buildspec
4. Create CodePipeline with required stages
5. Configure source repository webhooks/triggers

## Security Considerations
- Enable encryption for artifacts in S3
- Use IAM roles with least privilege
- Implement security scanning in pipeline
- Regular monitoring and auditing
- Secure storage of sensitive information using AWS Secrets Manager

## Monitoring and Maintenance
- Monitor pipeline executions in AWS Console
- Set up CloudWatch alerts for pipeline failures
- Regular review of SBOM reports
- Update dependencies and tools as needed

## Additional Resources
- [AWS CodePipeline Documentation](https://docs.aws.amazon.com/codepipeline)
- [SBOM Documentation](https://www.ntia.gov/SBOM)
- [CycloneDX Specification](https://cyclonedx.org/)
- [SPDX Specification](https://spdx.dev/)

## AWS CLI Commands and Resource Creation

### 1. Create S3 Buckets
```bash
# Create bucket for pipeline artifacts
aws s3 mb s3://sbom-pipeline-artifacts --region us-east-1

# Create bucket for Snyk scan reports
aws s3 mb s3://sbom-security-reports --region us-east-1

# Enable bucket encryption
aws s3api put-bucket-encryption \
    --bucket sbom-pipeline-artifacts \
    --server-side-encryption-configuration '{
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                }
            }
        ]
    }'
```

### 2. Create CodeArtifact Repository
```bash
# Create domain
aws codeartifact create-domain \
    --domain python-packages \
    --region us-east-1

# Create repository
aws codeartifact create-repository \
    --domain python-packages \
    --repository python-sbom-repo \
    --description "Repository for Python SBOM packages" \
    --region us-east-1
```

### 3. IAM Roles and Policies

#### CodeBuild Service Role
Contents of `codebuild-trust-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

Contents of `codeartifact-policy.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codeartifact:GetAuthorizationToken",
                "codeartifact:GetRepositoryEndpoint",
                "codeartifact:ReadFromRepository",
                "codeartifact:PublishPackageVersion",
                "codeartifact:PutPackageMetadata"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "sts:GetServiceBearerToken",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "sts:AWSServiceName": "codeartifact.amazonaws.com"
                }
            }
        }
    ]
}

Create the CodeBuild role:
```bash
# Create role
aws iam create-role \
    --role-name sbom-codebuild-role \
    --assume-role-policy-document file://codebuild-role-trust-policy.json

# Attach policy
aws iam put-role-policy \
    --role-name sbom-codebuild-role \
    --policy-name sbom-codebuild-policy \
    --policy-document file://codebuild-role-policy.json
```

#### CodePipeline Service Role
Create a file named `codepipeline-role-trust-policy.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "codepipeline.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Create a file named `codepipeline-role-policy.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::sbom-pipeline-artifacts/*",
                "arn:aws:s3:::sbom-security-reports/*"
            ],
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:GetBucketVersioning"
            ]
        },
        {
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
                "codecommit:CancelUploadArchive",
                "codecommit:GetBranch",
                "codecommit:GetCommit",
                "codecommit:GetUploadArchiveStatus",
                "codecommit:UploadArchive"
            ]
        },
        {
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
                "codebuild:BatchGetBuilds",
                "codebuild:StartBuild"
            ]
        }
    ]
}
```

Create the CodePipeline role:
```bash
# Create role
aws iam create-role \
    --role-name sbom-pipeline-role \
    --assume-role-policy-document file://codepipeline-role-trust-policy.json

# Attach policy
aws iam put-role-policy \
    --role-name sbom-pipeline-role \
    --policy-name sbom-pipeline-policy \
    --policy-document file://codepipeline-role-policy.json
```

### 4. Create CodeBuild Project
Create a file named `create-build-project.json`:
```json
{
    "name": "python-sbom-build",
    "source": {
        "type": "CODEPIPELINE"
    },
    "artifacts": {
        "type": "CODEPIPELINE"
    },
    "environment": {
        "type": "LINUX_CONTAINER",
        "image": "aws/codebuild/amazonlinux2-x86_64-standard:4.0",
        "computeType": "BUILD_GENERAL1_SMALL",
        "environmentVariables": [
            {
                "name": "CODEARTIFACT_DOMAIN",
                "value": "python-packages"
            },
            {
                "name": "CODEARTIFACT_REPO",
                "value": "python-sbom-repo"
            }
        ]
    },
    "serviceRole": "arn:aws:iam::<ACCOUNT_ID>:role/sbom-codebuild-role"
}
```

Create the CodeBuild project:
```bash
aws codebuild create-project \
    --cli-input-json file://create-build-project.json
```

### 5. Create Pipeline
Contents of `pipeline.json`:
```json
{
  "pipeline": {
    "name": "python-sbom-pipeline",
    "roleArn": "arn:aws:iam::908027380341:role/CodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-pipeline-artifacts-908027380341"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeStarSourceConnection",
              "version": "1"
            },
            "configuration": {
              "ConnectionArn": "arn:aws:codestar-connections:us-east-1:908027380341:connection/0356e8cb-35a6-411f-a0a6-18304ccb28aa",
              "FullRepositoryId": "Ninad-Lunge/my-package",
              "BranchName": "main"
            },
            "outputArtifacts": [
              {
                "name": "SourceCode"
              }
            ]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAndGenerateSBOM",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "python-sbom-generator"
            },
            "inputArtifacts": [
              {
                "name": "SourceCode"
              }
            ],
            "outputArtifacts": [
              {
                "name": "BuildOutput"
              }
            ]
          }
        ]
      }
    ]
  }
}

Create the pipeline:
```bash
aws codepipeline create-pipeline \
    --cli-input-json file://create-pipeline.json
```

Note: Replace `<ACCOUNT_ID>` in the above JSON files with your AWS account ID.

Remember to:
1. Update the region in the commands if you're not using us-east-1
2. Modify bucket names to ensure they are globally unique
3. Adjust the IAM policies based on your specific security requirements
4. Update the CodeBuild environment variables as needed
5. Customize the pipeline stages based on your workflow
