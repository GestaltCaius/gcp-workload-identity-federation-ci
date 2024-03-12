Setup Workload Identity Provider with Github Actions CI
===

# Useful links

* [Google Auth GitHub Actions](https://github.com/google-github-actions/auth)
* [GitHub docs: how to setup GCP OIDC in CI](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform#adding-a-google-cloud-workload-identity-provider)
* [Enabling keyless authentication from GitHub Actions](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions?hl=en)


# Setup GCP resources

```bash
export PROJECT_ID=XXX
export PROJECT_NUMBER=XXX
export SERVICE_ACCOUNT_NAME=github-ci
export WIF_POOL=github-pool
export WIF_PROVIDER=github-provider

# Create WIF Pool
gcloud iam workload-identity-pools create "${WIF_POOL}" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub pool"

# Create GitHub identity provider, mapping JWT attributes
gcloud iam workload-identity-pools providers create-oidc "${WIF_PROVIDER}" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="${WIF_POOL}" \
  --display-name="GitHub provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Create Service Account
gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME} \
  --project="${PROJECT_ID}" \
  --description="GitHub CI service account" \
  --display-name="GitHub CI"

# Give permissions to SA (editor role gives way too much permissions, DO NOT use this role in a real CI Service Account)
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"

# Bind GitHub identity to a GCP Service Account, using the repository name
gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${WIF_POOL}/attribute.repository/GestaltCaius/gcp-workload-identity-federation-ci"
```