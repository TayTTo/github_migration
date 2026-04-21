# Atlantis + Terraform + GitHub Enterprise — Setup Notes

## 1. Requirements
Works with any Terraform repo. Atlantis auto-discovers `.tf` files when a PR is opened. Needs a running server reachable by GitHub Enterprise.

## 2. Create Atlantis User
Create a regular user on GitHub Enterprise (e.g. `atlantis-bot`), grant write permission to the desired repo.

## 3. Generate Access Token (GitHub Enterprise)
Log in as the Atlantis user → **Settings → Developer settings → Personal access tokens → Generate new token**

Grant the `repo` scope 

Pass to Atlantis via `--gh-token` or `ATLANTIS_GH_TOKEN`, and set `--gh-hostname` to GHE hostname (e.g. `github.mycompany.com`).

## 4. Webhook Secret
Generate one:
```bash
openssl rand -hex 32
```
Paste this in two places: the GitHub Enterprise webhook config, and Atlantis via `--gh-webhook-secret` or `ATLANTIS_GH_WEBHOOK_SECRET`.

## 5. Deploy with Helm
```bash
helm repo add runatlantis https://runatlantis.github.io/helm-charts
helm install atlantis runatlantis/atlantis -f values.yaml -n atlantis --create-namespace
```

Key fields in `values.yaml`:
```yaml
atlantisUrl: https://atlantis.mycompany.com
github:
  user: atlantis-bot
  token: <token>
  secret: <webhook-secret>
```

## 6. Configure Webhook (GitHub Enterprise)
Go to repo (or org) → **Settings → Webhooks → Add webhook**

| Field | Value |
|---|---|
| Payload URL | `https://atlantis.mycompany.com/events` |
| Content type | `application/json` |
| Secret |  webhook secret |
| Events | Pull requests, Issue comments, Pull request reviews |

## 7. AWS Provider — EC2 IAM Role
 Attach an IAM Instance Profile to the EC2 node. 
```hcl
provider "aws" {
  region = "ap-southeast-1"
}
```

## 8. Webhook URL When Atlantis is Inside a Private Cluster
GitHub Enterprise needs to reach Atlantis at `/events`. Options:

- **ALB/Ingress (recommended):** Put an Application Load Balancer in front of the cluster, point a domain at it, restrict inbound to GHE's IP via Security Group.
- **Internal only:** If GHE is on-prem in the same network (VPN/Direct Connect), an internal load balancer with a private DNS name is enough — no public exposure needed.
- **NodePort (quick/dirty):** Expose via NodePort and point the webhook at `http://<ec2-node-ip>:<port>/events`. Fine for testing.
