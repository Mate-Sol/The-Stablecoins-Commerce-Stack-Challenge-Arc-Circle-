---
name: arc-cicd-deploy-target
description: Arc hackathon CI/CD deploys to the beta GKE cluster via OIDC, never prod
metadata:
  type: project
---

The Arc stablecoins repo's GitHub Actions (`.github/workflows/arc-cicd.yml`, a
matrix over arc-ui/arc-admin/arc-be) deploys to the **beta** GKE cluster, never prod.

- Cluster: `beta-env`, location `us-central1-a` (NOT `im-prod` / `us-central1`).
- Auth: OIDC Workload Identity Federation — secrets `BETA_GKE_WIF_PROVIDER` +
  `BETA_GKE_GKE_SA` (NOT the `GKE_BETA_CREDS` JSON key that the original template used).
- Namespace `arc-hackathon`; Deployments arc-ui/arc-admin/arc-be must already exist there.

**Why:** user was explicit — "use these credentials not the prod never the prod."

**How to apply:** any future deploy/workflow edits for Mate-Sol hackathon repos stay on
beta-env + OIDC. Don't reintroduce im-prod or credentials_json.

CI build gotchas already solved (see git history): client Dockerfiles use
`npm install` (not ci) for npm 10/11 lock drift; client-legacy build needs
`--ignore-scripts` (native usb/node-hid) and `NODE_OPTIONS=--max-old-space-size=6144`
(Rollup OOM on the wallet-SDK graph). Related: [[arc-admin-subpath-wiring]].
