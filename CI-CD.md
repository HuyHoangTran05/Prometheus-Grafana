CI/CD with GitHub Actions + Argo CD

This repository is wired for GitOps-style continuous delivery:
- GitHub Actions builds and pushes a container image to GitHub Container Registry (GHCR).
- The workflow then updates the Kubernetes manifests (via Kustomize) with the new image tag and commits back to main.
- Argo CD watches the k8s folder and automatically syncs those changes into your cluster.

Folder overview
- complete/: Spring Boot app + Dockerfile (multi-stage Gradle build)
- k8s/: Kubernetes base manifests + Kustomize entrypoint (kustomization.yaml)
- .github/workflows/cicd.yml: GitHub Actions workflow

Prerequisites
1) A GitHub repository with default branch main containing this code.
2) GHCR access via the built-in GITHUB_TOKEN. No extra secrets are needed, but you must allow the token to publish packages:
   - Repository Settings > Actions > General > Workflow permissions: enable "Read and write permissions".
   - The workflow already requests packages: write.
3) Decide image pull strategy in your cluster:
   - Easiest: Make the GHCR image public (Repository > Packages > your image > Package settings > Change visibility to public). No imagePullSecrets needed.
   - Private: Create a registry secret in the target cluster and reference it in the Deployment's imagePullSecrets. For GHCR:
     kubectl create secret docker-registry ghcr-pull-secret \
       --docker-server=ghcr.io \
       --docker-username=YOUR_GH_USERNAME \
       --docker-password=YOUR_GH_PAT \
       -n securingweb
     Then add under spec.template.spec.imagePullSecrets in k8s/app-deployment.yaml:
       imagePullSecrets:
         - name: ghcr-pull-secret

Bootstrap Argo CD
1) Install Argo CD in your cluster (namespace argocd). See https://argo-cd.readthedocs.io/.
2) After your first push to main (so the workflow rewrites repoURL), apply the Application manifest:
   kubectl apply -f k8s/argocd-application.yaml
   This Application points Argo CD at this repo's k8s/ directory and enables auto-sync with prune and self-heal.

How the workflow works
- Triggers on pushes to main affecting complete/, k8s/, or the workflow itself.
- Steps:
  1) Run unit tests with Gradle in complete/.
  2) Build and push container image to ghcr.io/<owner>/<repo>:<git-sha> and :latest using Docker Buildx.
  3) Use `kustomize edit set image` to pin image to the new tag in k8s/kustomization.yaml.
  4) Ensure k8s/argocd-application.yaml repoURL points to this repository.
  5) Commit changed manifests back to main. Argo CD detects and syncs to the cluster.

Local development
- Build and run locally (requires Java+Gradle or Docker):
  Option A (Docker):
    docker build -t securingweb:dev ./complete
    docker run -p 8080:8080 securingweb:dev
  Option B (Gradle):
    cd complete
    ./gradlew bootRun

Notes and tips
- Kustomize image mapping matches the base image name used in app-deployment.yaml (currently `securingweb:1.0`). Do not change the name `securingweb` unless you also update the workflow step that calls `kustomize edit set image`.
- If your default branch is not main, update the workflow "on.push.branches" and argocd-application.yaml targetRevision accordingly.
- If Argo CD cannot pull from GHCR (private images), validate your imagePullSecrets and ServiceAccount configuration.

