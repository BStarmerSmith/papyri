# Amazon ECR Integration

This directory contains Kubernetes manifests for enabling authentication with Amazon Elastic Container Registry (ECR).

## Overview

The setup consists of:

1. A namespace for ECR credentials (`ecr-credentials`)
2. A Kubernetes Secret containing AWS credentials
3. A CronJob that refreshes ECR authentication tokens every 6 hours
4. A patch for the default ServiceAccount to include the ECR image pull secret

## Setup Instructions

1. Edit the `credentials-secret.yaml` file to add your AWS credentials:

   ```bash
   # Open the file in your editor
   nano credentials-secret.yaml
   
   # Add your AWS credentials
   AWS_ACCESS_KEY_ID: "your-access-key"
   AWS_SECRET_ACCESS_KEY: "your-secret-key"
   AWS_REGION: "your-aws-region"  # e.g., us-east-1
   ```

2. Apply the configuration via Flux GitOps:

   ```bash
   # Commit and push changes to your repository
   git add .
   git commit -m "Add ECR credentials configuration"
   git push
   ```

   Flux will automatically apply these changes to your cluster.

3. Verify the setup:

   ```bash
   # Check if the namespace was created
   kubectl get namespace ecr-credentials
   
   # Check if the CronJob exists
   kubectl get cronjob -n ecr-credentials
   
   # Check if the credentials Secret exists
   kubectl get secret -n ecr-credentials
   ```

4. Trigger the ECR credential helper manually (optional):

   ```bash
   # Create a manual job from the CronJob
   kubectl create job --from=cronjob/ecr-credential-helper ecr-manual-job -n ecr-credentials
   
   # Check job logs
   kubectl logs -n ecr-credentials jobs/ecr-manual-job
   ```

## Usage in Deployments

Once the ECR credential helper is running, all deployments using the default ServiceAccount will be able to pull images from your ECR repositories.

Example image reference in a deployment:

```yaml
image: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/repository-name:tag
```

## Security Notes

- The AWS credentials should have minimal permissions, ideally only ECR read access
- Consider using IAM roles for service accounts (IRSA) for production environments
- Rotate your AWS credentials periodically
