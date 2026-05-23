# Deployment Guide

## Prerequisites

- AWS CLI configured with `eu-central-2` as default region
- uv installed (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
- Node.js 20+ for frontend

## 1. Bootstrap CDK

```bash
cd stoa-infra
uv sync
aws sts get-caller-identity   # verify credentials
uv run cdk bootstrap aws://ACCOUNT_ID/eu-central-2
```

## 2. Deploy Infrastructure

```bash
# Deploy all stacks (order handled by CDK dependency graph)
uv run cdk deploy --all --context env=dev --require-approval never

# Note outputs — you'll need these for frontend .env
uv run cdk deploy StoaAuthStack --outputs-file outputs/auth.json
```

## 3. Deploy Backend

The Lambda code is packaged from `stoa-backend/src/` during `cdk deploy StoaApiStack`.  
For manual updates:

```bash
cd stoa-backend
uv export --no-dev -o requirements.txt
zip -r function.zip src/ requirements.txt
aws lambda update-function-code --function-name stoa-api --zip-file fileb://function.zip
```

## 4. Deploy Frontend

```bash
cd stoa-frontend
cp .env.example .env
# Fill in outputs from CDK

npm install
npm run build

# Sync to S3 (bucket name from StoaFrontendStack output)
aws s3 sync dist/ s3://stoa-frontend-ACCOUNT_ID/ --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation --distribution-id DIST_ID --paths "/*"
```

## 5. Verify

```bash
curl https://API_ENDPOINT/health
# → {"status":"ok","version":"0.1.0"}
```
