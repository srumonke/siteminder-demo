# Siteminder Demo — Harness + ECR + Helm

## Repo Structure

```
.
├── app/
│   └── Dockerfile                        # Placeholder nginx app
├── helm/
│   └── dummy-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml           # Uses <+artifact.image>
│           └── service.yaml
└── .github/
    └── workflows/
        └── build-push-ecr.yml            # CI: builds & pushes to ECR on merge to main
```

---

## How It Works

1. Dev merges PR to `main`
2. GitHub Actions builds the Docker image, tags it with the **Git SHA** (e.g. `a1b2c3d`), pushes to ECR
3. Dev opens Harness pipeline, **selects the tag** from a dropdown (no config file needed)
4. Harness resolves `<+artifact.image>` → full ECR path + selected tag → deploys via Helm to EKS

---

## GitHub Actions Setup

Add these secrets to your GitHub repo (`Settings → Secrets → Actions`):

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |

The IAM user needs these permissions:
- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:PutImage`
- `ecr:InitiateLayerUpload`
- `ecr:UploadLayerPart`
- `ecr:CompleteLayerUpload`

---

## Harness Setup

### 1. Add AWS Connector
`Account Settings → Connectors → New → AWS`
- Auth: IAM or Access Key
- Region: `us-west-2`

### 2. Add Service
- Type: **Kubernetes**
- Manifests: **Helm Chart** → point to `helm/dummy-app` in this repo
- Artifacts: **ECR**
  - Registry: `664418987337.dkr.ecr.us-west-2.amazonaws.com`
  - Repository: `sruthi/dummy-example`
  - Tag: `<+input>` ← this gives the dropdown at runtime

### 3. Key Expression in deployment.yaml
```yaml
image: <+artifact.image>
```
At runtime this resolves to:
```
664418987337.dkr.ecr.us-west-2.amazonaws.com/sruthi/dummy-example:<selected-tag>
```

### 4. Add Artifact Trigger (optional but recommended)
`Triggers → New → Artifact → ECR`
- This auto-detects new tags pushed to ECR so they appear in the dropdown immediately
