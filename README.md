## Overview / Goal

Stand up a basic CI/CD + GitOps foundation:

- **Jenkins** for CI jobs and build automation
- **Argo CD** for GitOps deployments into Kubernetes namespaces (**dev** and **prod**)
- A shared **Docker registry pull secret** in Kubernetes for private image pulls

## Architecture / Flow

1. Developer pushes code → images built/pushed (Jenkins).
2. GitOps repo contains Kubernetes manifests (ApplicationSet) (Argo CD).
3. Argo CD syncs manifests into **dev**/**prod** namespaces.
4. Workloads pull images using `dockerhub-secret-shared`.
- Why this flow?
    - **Separation of concerns:** Jenkins builds; Argo CD deploys.
    - **Auditability:** desired state is stored in Git.
    - **Repeatability:** same steps for dev/prod namespaces.

## Prerequisites

- Docker installed and running on the host where Jenkins will run.
- A Kubernetes cluster and `kubectl` configured (`k` alias assumed in commands below).
- Network access to:
    - Docker Hub (for Jenkins image and for pulling/pushing images)
    - GitHub repo used by Argo CD: `https://github.com/jainambarbhaya1509/gitops.git`
- Argo CD CLI installed (`argocd`) for login and repo operations.
- A Docker token available (used later as `DOCKER_TOKEN`).

---

## Step-by-step Installation

## 1) 🧰 Install Jenkins

### 1.1 Pull Jenkins LTS image

- **What it does:** downloads the Jenkins LTS container image.
- **Why required:** Jenkins will run from this image.
- **Expected result:** Docker reports the image is pulled (or already up-to-date).

```bash
docker pull jenkins/jenkins:lts
```

### 1.2 Run Jenkins container

- **What it does:** starts Jenkins as a detached container and maps ports/volumes.
- **Why required:** runs Jenkins and persists its data via `jenkins_home`.
- **Expected result:** Docker returns a container ID; `jenkins` container runs in background.

```bash
docker run -d \<br>--name jenkins \<br>-p 8081:8080 \<br>-p 50000:50000 \<br>-v jenkins_home:/var/jenkins_home \<br>-v /var/run/docker.sock:/var/run/docker.sock \<br>jenkins/jenkins:lts
```

### 1.3 Retrieve initial Jenkins admin password

- **What it does:** prints the initial admin password from the Jenkins home volume.
- **Why required:** needed for first-time Jenkins UI setup/login.
- **Expected result:** a password string is printed to stdout.

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### ✅ Completion checklist (Jenkins)

- [ ]  `jenkins` container is running
- [ ]  Jenkins UI is reachable on `http://localhost:8081`
- [ ]  Initial admin password retrieved

---

## 2) 🚀 Install Argo CD in Kubernetes

### 2.1 Create namespaces

- **What it does:** creates namespaces for Argo CD, dev, and prod.
- **Why required:** isolates Argo CD system components and environment workloads.
- **Expected result:** `namespace/argocd`, `namespace/dev`, `namespace/prod` created (or already exists).

```bash
k create ns argocd
k create ns dev
k create ns prod
```

### 2.2 Install Argo CD manifests

- **What it does:** applies the official Argo CD install manifest.
- **Why required:** installs Argo CD components (server, repo-server, application-controller, etc.).
- **Expected result:** multiple Kubernetes resources created/updated in `argocd` namespace.

```bash
kubectl apply --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2.3 Get Argo CD initial admin password

- **What it does:** reads the `argocd-initial-admin-secret` and decodes the admin password.
- **Why required:** needed to log in to Argo CD for first-time setup.
- **Expected result:** decoded password printed to stdout.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \<br>-o jsonpath="\{.data.password\}" \| base64 -d
```

### 2.4 Port-forward Argo CD API server

- **What it does:** forwards local port `8080` to Argo CD server service port `443`.
- **Why required:** provides local access for CLI login without exposing a LoadBalancer/Ingress.
- **Expected result:** terminal shows port-forwarding logs and continues running.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2.5 Login to Argo CD via CLI

- **What it does:** logs into Argo CD running on `localhost:8080`.
- **Why required:** allows you to register repos and manage applications via CLI.
- **Expected result:** successful login message or token stored for CLI.

```bash
argocd login localhost:8080 --username admin --password <PASSWORD> --insecure
```

### 2.6 Add GitOps repository

- **What it does:** registers the Git repo with Argo CD so it can sync manifests.
- **Why required:** Argo CD must be able to fetch your desired-state manifests from Git.
- **Expected result:** repo added successfully.

```bash
argocd repo add  https://github.com/jainambarbhaya1509/gitops.git 
```

### 2.7 Apply ApplicationSet (GitOps bootstrap)

- **What it does:** applies an Argo CD ApplicationSet definition.
- **Why required:** ApplicationSet can generate Applications for environments/services automatically.
- **Expected result:** ApplicationSet resource created/updated in `argocd` namespace.

```bash
kubectl apply -f argocd/applicationset.yaml -n argocd 
```

### ✅ Completion checklist (Argo CD)

- [ ]  `argocd` namespace exists
- [ ]  Argo CD pods are running
- [ ]  Admin password retrieved
- [ ]  Port-forward works on `localhost:8080`
- [ ]  Repo is added to Argo CD
- [ ]  ApplicationSet applied

---

## 3) 🔐 Add Docker registry pull secret (shared)

<aside>
⚠️

**Security note:** Avoid long-lived Docker tokens in shell history and environment variables. Prefer a single source of truth such as Vault (as noted below).

</aside>

### 3.1 Export Docker token

- **What it does:** sets an environment variable with your Docker token for reuse in `kubectl create secret`.
- **Why required:** referenced by subsequent commands.
- **Expected result:** variable is available in the current shell session.

```bash
export DOCKER_TOKEN='<your-docker-token>'
```

### 3.2 Create the secret in `dev` and `prod` namespaces

- **What it does:** creates an `imagePullSecret` of type `docker-registry`.
- **Why required:** allows Kubernetes workloads to pull images from Docker Hub using credentials.
- **Expected result:** secret `dockerhub-secret-shared` created in each namespace.

```bash
kubectl create secret docker-registry dockerhub-secret-shared \<br>-n dev \<br>--docker-server=https://docker.io \<br>--docker-username=jainambarbhaya15 \<br>--docker-password="$DOCKER_TOKEN" \<br>--docker-email=jainam.bharbhaya-i@transbnk.co.in
kubectl create secret docker-registry dockerhub-secret-shared \<br>-n prod \<br>--docker-server=https://docker.io \<br>--docker-username=jainambarbhaya15 \<br>--docker-password="$DOCKER_TOKEN" \<br>--docker-email=jainam.bharbhaya-i@transbnk.co.in
```

### 3.3 Verify the secret exists

- **What it does:** lists the created secret in each namespace.
- **Why required:** quick check that secret creation succeeded.
- **Expected result:** secret appears in output.

```bash
kubectl get secret dockerhub-secret-shared -n dev<br>kubectl get secret dockerhub-secret-shared -n prod
```

### 3.4 Alternate Docker server endpoint (v1)

- **What it does:** creates the same secret but points to the Docker index v1 endpoint.
- **Why required:** can be useful for compatibility if `docker.io` endpoint causes auth/pull issues.
- **Expected result:** secret created (or command fails if secret already exists).

```bash
kubectl create secret docker-registry dockerhub-secret-shared \<br>-n dev \<br>--docker-server=https://index.docker.io/v1/ \<br>--docker-username=jainambarbhaya15 \<br>--docker-password="$DOCKER_TOKEN" \<br>--docker-email=jainam.bharbhaya-i@transbnk.co.in
```

### ✅ Completion checklist (Docker secret)

- [ ]  `DOCKER_TOKEN` exported in current shell
- [ ]  `dockerhub-secret-shared` exists in `dev`
- [ ]  `dockerhub-secret-shared` exists in `prod`

---

## Configuration

### Quick configuration summary

| Component | Key config | Where |
| --- | --- | --- |
| Jenkins | Ports: 8081→8080, 50000; volume: jenkins_home; Docker socket mount | docker run flags |
| Argo CD | Namespace: argocd; Port-forward: 8080→443 | kubectl apply + port-forward |
| GitOps repo | https://github.com/jainambarbhaya1509/gitops.git | argocd repo add |
| Docker pull secret | Name: dockerhub-secret-shared; Namespaces: dev, prod | kubectl create secret docker-registry |

---

## Verification commands

Use these to validate health after installation.

- Argo CD (pods + services)
    - **What it does:** checks Argo CD pods and service status.
    - **Why required:** verifies the installation is running.
    - **Expected result:** pods in `Running` state; services present.
    
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret \<br>-o jsonpath="\{.data.password\}" \| base64 -d
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
    
- Kubernetes secrets (dev/prod)
    - **What it does:** verifies the Docker registry secret exists.
    - **Why required:** ensures deployments can pull private images.
    - **Expected result:** secret listed in both namespaces.
    
    ```bash
    kubectl get secret dockerhub-secret-shared -n dev<br>kubectl get secret dockerhub-secret-shared -n prod
    ```
    

---

## Common issues / Troubleshooting

### Jenkins

- **Port conflict on 8081**
    - **Symptom:** container fails to start or port bind error.
    - **Fix:** choose a different host port in `-p 8081:8080`.
- **Docker builds fail inside Jenkins**
    - **Cause:** missing Docker socket mount or permissions.
    - **Check:** `-v /var/run/docker.sock:/var/run/docker.sock` is present.

### Argo CD

- **Cannot login via CLI**
    - **Cause:** port-forward not running, wrong password, or CLI not installed.
    - **Check:** port-forward terminal is active and using `localhost:8080`.
- **Repo add fails**
    - **Cause:** network/DNS restrictions or missing repo credentials (if private).
    - **Check:** repo URL is reachable from Argo CD and credentials are configured if needed.

### Docker secret

- **Image pull errors (401/403)**
    - **Cause:** bad token, wrong registry server, or secret not referenced by service account/workload.
    - **Try:** alternate server endpoint:
        - `--docker-server=https://index.docker.io/v1/`

---

## Production recommendations / Best practices

- **🔐 Secret management:** use a single source of truth like **Vault** (as noted) instead of manually exporting tokens.
- **Expose Argo CD properly:** replace port-forward with Ingress/LoadBalancer + TLS for team access.
- **RBAC:** restrict Argo CD admin usage; create roles for environments.
- **Namespace isolation:** keep `dev` and `prod` separated and enforce policies (NetworkPolicies, resource quotas).
- **Jenkins hardening:**
    - Avoid running Jenkins with excessive host privileges.
    - Review plugins and keep them updated.
    - Secure credentials via Jenkins credentials store (and/or an external secret manager).

---

## Cleanup commands (if applicable)

No cleanup commands were provided in the original notes.

<aside>
⚠️

If you later add cleanup steps, include:

- How to stop/remove Jenkins container and volume
- How to uninstall Argo CD manifests from `argocd`
- How to delete the `dockerhub-secret-shared` from namespaces
</aside>

k8s + argocd + helm + jenkins + github pipeline for ci/cd
