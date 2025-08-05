# Amazon ECR Integration

This directory contains Kubernetes manifests for enabling authentication with Amazon Elastic Container Registry (ECR).

## Overview

The setup consists of:

1. A namespace for ECR credentials (`ecr-credentials`)
2. A Kubernetes Secret containing AWS credentials
3. A CronJob that refreshes ECR authentication tokens every 6 hours

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

2. Apply the secrets

   ```bash
   kubectl apply -f credentials-secret.yaml
   ```

3. push the configuration to git, Flux will automatically apply these changes to your cluster.

4. Verify the setup:

   ```bash
   # Check if the namespace was created
   kubectl get namespace ecr-credentials
   
   # Check if the CronJob exists
   kubectl get cronjob -n ecr-credentials
   
   # Check if the credentials Secret exists
   kubectl get secret -n ecr-credentials
   ```

5. Trigger the ECR credential helper manually (optional):

   ```bash
   # Create a manual job from the CronJob
   kubectl create job --from=cronjob/ecr-credential-helper ecr-manual-job -n ecr-credentials
   
   # Check job logs
   kubectl logs -n ecr-credentials jobs/ecr-manual-job
   ```

## Usage in Deployments

You will need to add the imagePullSecrets to your deployment. And add the secret name to the imagePullSecrets.

```yaml
imagePullSecrets:
  - name: ecr-docker-registry
  - namespace: ecr-credentials

## Security Notes

- The AWS credentials should have minimal permissions, ideally only ECR read access
- Consider using IAM roles for service accounts (IRSA) for production environments
- Rotate your AWS credentials periodically
