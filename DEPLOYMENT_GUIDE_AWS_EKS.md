# AWS EKS Deployment Guide for Fake Shop (HTTPS Only)

This guide provides step-by-step instructions to deploy the Fake Shop application on AWS EKS using HTTPS only, with all commands using the AWS CLI and eksctl.

---

## Should I Use an HTTP Fallback?

**Recommendation:** For security and best practices, do NOT use HTTP fallback in production. Serve your app over HTTPS only to protect user data and prevent downgrade attacks. For testing, you may enable HTTP if you want to verify redirection or compatibility, but always prefer HTTPS for real deployments.

If you want to enable HTTP fallback (not recommended for production):
- Add HTTP (port 80) to your Ingress `listen-ports` annotation.
- Optionally, configure a redirect from HTTP to HTTPS using Ingress annotations.

---

## Prerequisites
- AWS CLI, `kubectl`, and `eksctl` installed and configured.
- Docker installed (for building images, if needed).
- A public domain you control (for ACM and ALB).

---

## 1. Create an EKS Cluster (with eksctl)
```bash
# Replace <cluster-name> and <region> as needed
eksctl create cluster \
  --name <cluster-name> \
  --region <region> \
  --nodes 2 \
  --managed
```

---

## 2. Install the AWS Load Balancer Controller
Follow the official AWS guide:
https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

**Key steps:**
- Create IAM OIDC provider for your cluster
- Create IAM policy and service account for the controller
- Install the controller with `kubectl apply -f ...`

---

## 3. Request an ACM Certificate (for your domain)
```bash
# In AWS Console or with AWS CLI:
aws acm request-certificate \
  --domain-name <your-domain.com> \
  --validation-method DNS \
  --region <region>
# Complete DNS validation as instructed by ACM
```
- Note the ACM certificate ARN for use in your Ingress manifest.

---

## 4. (Optional) Set Up Route53 for DNS Management
If you want to manage your DNS with Route53, follow these steps:

### Create a Hosted Zone
```bash
aws route53 create-hosted-zone \
  --name <your-domain.com> \
  --caller-reference $(date +%s)
```
- Note the Hosted Zone ID from the output.

### Add a CNAME Record for the ALB
After deploying your Ingress and getting the ALB DNS name:
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <hosted-zone-id> \
  --change-batch ' {
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "<your-domain.com>",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [{"Value": "<alb-dns-name>"}]
        }
      }
    ]
  }'
```

### Delete Route53 Resources After Testing
```bash
# Delete the CNAME record (edit as needed):
aws route53 change-resource-record-sets \
  --hosted-zone-id <hosted-zone-id> \
  --change-batch ' {
    "Changes": [
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "<your-domain.com>",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [{"Value": "<alb-dns-name>"}]
        }
      }
    ]
  }'
# Delete the hosted zone:
aws route53 delete-hosted-zone --id <hosted-zone-id>
```

---

## 5. Update Your Ingress Manifest
- Edit `k8s/fake-shop-ingress.yaml`:
  - Set `alb.ingress.kubernetes.io/certificate-arn` to your ACM certificate ARN.
  - Set the `host` to your domain.

---

## 6. Deploy Application Resources
```bash
kubectl apply -f k8s/postgres-secret.yaml
kubectl apply -f k8s/postgres-pvc.yaml
kubectl apply -f k8s/postgres-deploy.yaml
kubectl apply -f k8s/fake-shop-deploy.yaml
kubectl apply -f k8s/fake-shop-ingress.yaml
```

---

## 7. Configure DNS (if not using Route53)
- After applying the Ingress, AWS will provision an ALB and provide a DNS name (see in EC2 > Load Balancers or with `kubectl get ingress`).
- In your DNS provider, create a CNAME record pointing your domain to the ALB DNS name.

---

## 8. Verify Deployment
- Visit `https://<your-domain.com>` in your browser.
- Ensure the app loads and is served over HTTPS.

---

## 9. Clean Up
- Delete all EKS, ALB, ACM, and DNS resources after testing to avoid charges:
```bash
eksctl delete cluster --name <cluster-name> --region <region>
# Delete ACM cert and DNS records manually if needed
# If you used Route53, see the Route53 cleanup commands above
```

---

## Notes
- This setup is secure and suitable for both testing and production.
- All traffic is served over HTTPS only (no HTTP fallback, unless you explicitly enable it).
- PostgreSQL data is persisted using an EBS-backed PVC.
- All sensitive data is managed via Kubernetes Secrets.
