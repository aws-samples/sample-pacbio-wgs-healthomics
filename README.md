# PacBio Whole Genome Sequencing (WGS) Analysis with AWS HealthOmics Workflows

This repository contains resources for benchmarking and running PacBio Whole Genome Sequencing (WGS) variant pipeline analysis using AWS HealthOmics Workflows.

## Architecture

![PacBio WGS Architecture](./images/Pacbio_architecture.drawio.png)

## Overview

This project demonstrates how to implement PacBio Whole Genome Sequencing (WGS) variant analysis pipelines using AWS HealthOmics Workflows. The repository includes CloudFormation templates for infrastructure setup, parameter templates for different compute environments, and workflow definitions optimized for various GPU and CPU configurations.

## Repository Structure

```
├── images/
│   └── Pacbio_architecture.drawio.png          # Architecture diagram
├── cloudformation/
│   └── pacbio-dockers-migration-cfn.yaml       # Infrastructure deployment template
├── healthomics-templates/
│   ├── parameters-template.json                # Base parameter template
│   ├── parameters-a10g-values.json            # A10G GPU optimized parameters
│   ├── parameters-l4-values.json              # L4 GPU optimized parameters
│   ├── parameters-t4-values.json              # T4 GPU optimized parameters
│   └── parameters-default-cpu-values.json     # CPU-based parameters
└── README.md
```

## Prerequisites

Before getting started, ensure you have:

- AWS Account with access to HealthOmics service
- Appropriate IAM permissions for HealthOmics, S3, and CloudFormation
- AWS CLI configured with your credentials
- PacBio WGS data available in S3
- Virtual Private Cloud (VPC) with two public subnets
- VPC endpoints for S3 gateway, CodeBuild, and CloudFormation
- Customer Managed Key (CMK) in AWS KMS for security compliance

## Getting Started

### Step 1: Infrastructure Setup

Deploy the required infrastructure using the CloudFormation template:

```bash
aws cloudformation deploy \
    --template-file cloudformation/pacbio-dockers-migration-cfn.yaml \
    --stack-name pacbio-healthomics-stack \
    --capabilities CAPABILITY_IAM \
    --profile <YOUR_AWS_PROFILE>
```

### Step 2: Create Security Group (One-time setup)

Create a security group for HTTPS traffic:

```bash
# Create the security group and capture its ID
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
    --group-name pacbio-https-sg \
    --description "Security group for HTTPS traffic - self-referencing" \
    --vpc-id <YOUR_VPC_ID> \
    --query 'GroupId' \
    --output text \
    --region <YOUR_AWS_REGION> \
    --profile <YOUR_AWS_PROFILE>)

echo "Created Security Group: $SECURITY_GROUP_ID"

# Add inbound rule for HTTPS (port 443) from the security group itself
aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 443 \
    --source-group $SECURITY_GROUP_ID \
    --region <YOUR_AWS_REGION> \
    --profile <YOUR_AWS_PROFILE>
```

### Step 3: Prepare Workflow Definition

Clone and prepare the PacBio HiFi WGS workflow:

```bash
# Clone the PacBio workflow repository
git clone https://github.com/PacificBiosciences/HiFi-human-WGS-WDL.git

# Create workflow package
workflow_name="HiFi-human-WGS-WDL"
(cd ./${workflow_name} && zip -9 -r "${OLDPWD}/${workflow_name}.zip" . -x "./.git/*")

# Upload to S3
aws s3 cp HiFi-human-WGS-WDL.zip s3://<YOUR_BUCKET>/omics-workflows/ \
    --profile <YOUR_AWS_PROFILE>
```

### Step 4: Create HealthOmics Workflow

Create the workflow in AWS HealthOmics using your preferred parameter template:

```bash
# Set workflow variables
workflow_name="HiFi-human-WGS-WDL"
definition_uri="s3://<YOUR_BUCKET>/omics-workflows/${workflow_name}.zip"

# Create workflow with parameter template
workflow_id=$(aws omics create-workflow \
    --engine WDL \
    --definition-uri ${definition_uri} \
    --name "Pacbio${workflow_name}-$(date +%Y%m%dT%H%M%SZ%z)" \
    --parameter-template file://healthomics-templates/parameters-template.json \
    --query 'id' \
    --output text \
    --main workflows/singleton.wdl \
    --profile <YOUR_AWS_PROFILE>)

echo "Created workflow with ID: ${workflow_id}"

# Wait for workflow to become active
aws omics wait workflow-active --id "${workflow_id}" --profile <YOUR_AWS_PROFILE>

# Get workflow details
aws omics get-workflow --id "${workflow_id}" \
    --profile <YOUR_AWS_PROFILE> > "workflow-${workflow_name}.json"
```

### Step 5: Run the Workflow

Once the workflow is active, start a workflow run:

```bash
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --output text --query "Account" --profile <YOUR_AWS_PROFILE>)

# Set the IAM role ARN (replace with actual role name from CloudFormation output)
OMICS_WORKFLOW_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/<OMICS_ROLE_NAME>"

# Start workflow run
WORKFLOW_RUN_ID=$(aws omics start-run \
    --role-arn "${OMICS_WORKFLOW_ROLE_ARN}" \
    --workflow-id "$(jq -r '.id' workflow-${workflow_name}.json)" \
    --name "pacbio-run-$(date +%Y%m%d-%H%M%S)" \
    --output-uri "s3://<YOUR_BUCKET>/omics-output/pacbio-results" \
    --parameters file://healthomics-templates/parameters-a10g-values.json \
    --query 'id' \
    --output text \
    --profile <YOUR_AWS_PROFILE>)

echo "Started workflow run with ID: ${WORKFLOW_RUN_ID}"
```

## Parameter Templates

Choose the appropriate parameter template based on your compute requirements:

- **[parameters-template.json](./healthomics-templates/parameters-template.json)** - Base template for customization
- **[parameters-a10g-values.json](./healthomics-templates/parameters-a10g-values.json)** - Optimized for A10G GPU instances
- **[parameters-l4-values.json](./healthomics-templates/parameters-l4-values.json)** - Optimized for L4 GPU instances  
- **[parameters-t4-values.json](./healthomics-templates/parameters-t4-values.json)** - Optimized for T4 GPU instances
- **[parameters-default-cpu-values.json](./healthomics-templates/parameters-default-cpu-values.json)** - CPU-based configuration

## AWS HealthOmics Commands Reference

### Key Commands and Parameters:

- `aws omics create-workflow`: Creates a new workflow definition
  - `--engine`: Workflow engine (WDL, Nextflow, CWL)
  - `--definition-uri`: S3 URI containing workflow files
  - `--parameter-template`: JSON file defining workflow parameters
  - `--profile`: AWS CLI profile to use (replace with your profile name)

- `aws omics start-run`: Executes a workflow
  - `--workflow-id`: ID of the workflow to run
  - `--role-arn`: IAM role for workflow execution
  - `--output-uri`: S3 location for results
  - `--parameters`: Runtime parameters file
  - `--cache-id`: (Optional) Cache ID for workflow progress caching
  - `--cache-behavior`: (Optional) Caching behavior (CACHE_ON_FAILURE, etc.)

- `aws omics get-run`: Retrieves run status and details
- `aws omics list-runs`: Lists all workflow runs
- `aws omics cancel-run`: Cancels a running workflow

## Monitoring and Troubleshooting

Monitor your workflow runs:

```bash
# Check run status
aws omics get-run --id ${WORKFLOW_RUN_ID} --profile <YOUR_AWS_PROFILE>

# List all runs
aws omics list-runs --profile <YOUR_AWS_PROFILE>

# Get run logs (if available)
aws omics get-run-task --id ${WORKFLOW_RUN_ID} --task-id <TASK_ID> --profile <YOUR_AWS_PROFILE>
```

## Cost Optimization

- Use appropriate instance types based on your data size and processing requirements
- Implement workflow caching to avoid re-running completed tasks
- Monitor resource utilization and adjust instance types accordingly

## Security Best Practices

- Use IAM roles with minimal required permissions
- Enable encryption for S3 buckets and HealthOmics workflows
- Use VPC endpoints to keep traffic within AWS network
- Regularly rotate access keys and review permissions

## Support and Documentation

- [AWS HealthOmics Documentation](https://docs.aws.amazon.com/omics/)
- [PacBio HiFi WGS Workflow](https://github.com/PacificBiosciences/HiFi-human-WGS-WDL)
- [CloudFormation Template](./cloudformation/pacbio-dockers-migration-cfn.yaml)

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## License

This project is licensed under the terms specified in [LICENSE](LICENSE).
