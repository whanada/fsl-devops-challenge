# Lightsail deployment setup

The workflow deploys the image tagged with the main commit SHA to the existing
`fsl-challenge` container service in `us-east-1`, exposes Nginx port 80, waits for
that deployment version to become ACTIVE, and checks the public URL.

## One-time private ECR access

In Lightsail, open Containers > fsl-challenge > Images > Add repository.
Select `whanada/fsl-devops-challenge`, choose Add, and wait for completion.
This enables the Lightsail image puller role and grants it repository access.
The GitHub publishing role and Lightsail image puller role have separate jobs.

Official instructions:
https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-container-service-ecr-private-repo-access.html

## GitHub IAM role permissions

Keep the existing main-only OIDC trust and ECR permissions on
`GitHubActions-FSL-Deploy`. Add an inline policy with the two statements below.
Replace `LIGHTSAIL_SERVICE_ARN` with the actual ARN returned by this command,
run in AWS CloudShell or an authenticated administrator terminal:

```bash
aws lightsail get-container-services --service-name fsl-challenge \
  --region us-east-1 --query 'containerServices[0].arn' --output text
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadLightsailServices",
      "Effect": "Allow",
      "Action": "lightsail:GetContainerServices",
      "Resource": "*",
      "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
    },
    {
      "Sid": "DeployFSLContainer",
      "Effect": "Allow",
      "Action": "lightsail:CreateContainerServiceDeployment",
      "Resource": "LIGHTSAIL_SERVICE_ARN"
    }
  ]
}
```

Repository Actions variables remain `AWS_REGION`, `AWS_ROLE_ARN` and
`ECR_REPOSITORY`. No additional variable or static AWS credential is needed.

Merge the workflow into main after PR validation passes. The push to main
validates the app, publishes the image, and deploys it. PR runs do not deploy.
The local Kubernetes Ingress basic authentication is not part of this image;
the Lightsail endpoint serves the application publicly.
