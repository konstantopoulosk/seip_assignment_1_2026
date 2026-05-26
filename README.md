# echo-api — SEIP Assignment 1 (2026)

**Author:** Konstantinos Konstantopoulos (`konstantopoulosk`)  
**Repository:** https://github.com/konstantopoulosk/seip_assignment_1_2026  
**Container image:** `ghcr.io/konstantopoulosk/echo-api:latest`

A Node.js Express API (`echo-api`) packaged with Docker, delivered via GitHub Actions to GHCR, and orchestrated on a local Minikube cluster using ConfigMaps and Secrets for external configuration.

| Metadata | Details |
| :--- | :--- |
| **Course** | Software Engineering in Practice |
| **Deadline** | 3rd June |
| **Total Points** | 100 |

---

## Prerequisites

Install and verify the following on your machine:

- [Git](https://git-scm.com/) and a GitHub account
- [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) (running)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

> **Note:** Do not modify `server.js`. Configuration is injected at runtime via environment variables (Docker `-e`, Kubernetes ConfigMap/Secret).

---

## Repository structure

```
.
├── Dockerfile
├── server.js
├── package.json
├── .github/workflows/ci-cd.yaml
└── k8s/
    ├── configmap.yaml
    ├── secret.yaml
    ├── deployment.yaml
    └── service.yaml
```

---

## 1. Clone the repository

```powershell
git clone https://github.com/konstantopoulosk/seip_assignment_1_2026.git
cd seip_assignment_1_2026
```

---

## 2. Run locally with Docker (Task 1)

Build the image:

```powershell
docker build -t echo-api:local .
```

Run with sample configuration:

```powershell
docker run --rm -p 3000:3000 `
  -e WELCOME_MESSAGE="Welcome from Docker!" `
  -e NODE_ENV=production `
  -e API_SECRET_KEY="my-assignment-secret-2026" `
  echo-api:local
```

Verify:

| Endpoint | Command / URL |
|----------|----------------|
| Root | http://localhost:3000/ |
| Secure config | http://localhost:3000/secure-config |
| Health | http://localhost:3000/health |

---

## 3. CI/CD pipeline (Task 2)

Pushing to the `main` branch triggers `.github/workflows/ci-cd.yaml`, which:

1. Checks out the repository
2. Authenticates to GHCR with `${{ secrets.GITHUB_TOKEN }}`
3. Builds and pushes `ghcr.io/konstantopoulosk/echo-api:latest`

View runs: https://github.com/konstantopoulosk/seip_assignment_1_2026/actions

If Minikube cannot pull the image, set the `echo-api` package visibility to **Public** under GitHub Packages settings.

---

## 4. Deploy to Minikube (Task 3)

### Start the cluster

```powershell
minikube start
```

### Apply all manifests

From the repository root:

```powershell
kubectl apply -f k8s/
```

Expected resources:

| Resource | Name |
|----------|------|
| ConfigMap | `echo-api-config` |
| Secret | `echo-api-secret` |
| Deployment | `echo-api` (3 replicas) |
| Service | `echo-api-service` (ClusterIP) |

### Wait for rollout

```powershell
kubectl rollout status deployment/echo-api
kubectl get pods
```

You should see **3 pods** in `Running` state and deployment **3/3** ready:

```powershell
kubectl get all -n default
kubectl get configmap,secret
```

---

## 5. Access the application (Task 4 — port-forward)

The Service is `ClusterIP` (internal only). Forward it to your machine:

```powershell
kubectl port-forward service/echo-api-service 8080:80
```

Keep this terminal open. In a browser or second terminal:

| Endpoint | URL | Expected response |
|----------|-----|-------------------|
| Greeting | http://localhost:8080/ | Custom `WELCOME_MESSAGE` and `"environment":"production"` |
| Secure config | http://localhost:8080/secure-config | `"status":"Authorized"` and masked secret suffix |
| Health | http://localhost:8080/health | `{"status":"Healthy"}` |

**curl examples:**

```powershell
curl http://localhost:8080/
curl http://localhost:8080/secure-config
curl http://localhost:8080/health
```

---

## Configuration reference

### ConfigMap (`k8s/configmap.yaml`)

| Key | Value |
|-----|--------|
| `WELCOME_MESSAGE` | `Welcome to the Software Engineering in Practice Assignment Cluster!` |
| `NODE_ENV` | `production` |

### Secret (`k8s/secret.yaml`)

| Key | Plain value (before Base64) |
|-----|---------------------------|
| `API_SECRET_KEY` | `my-assignment-secret-2026` |

To encode a new secret without a trailing newline (PowerShell):

```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("your-secret-here"))
```

---

## API endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Returns welcome message and environment |
| GET | `/secure-config` | Returns authorization status and masked secret suffix |
| GET | `/health` | Liveness/readiness probe target (port 3000) |

---

## AI usage & future engineering report (Task 4)

*Copy this section into your submission PDF and adjust wording if needed to reflect your personal experience.*

### AI integration

I used **Cursor (AI-assisted IDE)** as the primary generative AI tool during this assignment. It helped scaffold the `Dockerfile`, the GitHub Actions workflow for GHCR, and all four Kubernetes manifests (`configmap`, `secret`, `deployment`, `service`). I also used it to explain assignment requirements, verify command sequences, and troubleshoot when local tools were not yet installed (Docker after PC restart, Minikube missing from PATH).

I reviewed every generated file against the assignment rubric and official documentation before committing. I ran builds, `kubectl apply`, and browser tests myself to confirm behavior.

### Utility analysis

The most useful AI assistance was:

- **Kubernetes YAML structure** — correct `valueFrom` / `configMapKeyRef` / `secretKeyRef` wiring, probe definitions, and resource requests/limits
- **Dockerfile layer caching** — ordering `package.json` + `npm install` before copying `server.js`
- **CI/CD workflow syntax** — GHCR login with `GITHUB_TOKEN` and `packages: write` permissions
- **Operational checklists** — which `kubectl` commands to run and what output to expect for submission screenshots
- **Base64 encoding** — reminding me to avoid trailing newlines when encoding `API_SECRET_KEY`

### Friction points

AI was less reliable when:

- **Assuming tools were installed** — early on, Minikube was not on my PATH; the agent could not start the cluster until I installed it manually
- **Environment-specific paths** — OneDrive folder paths and PowerShell syntax sometimes differed from bash examples in generic AI answers
- **Screenshot / submission workflow** — I had to clarify that proof images belong in the university PDF, not in the GitHub repo or GHCR

I resolved issues by reading terminal errors directly (`ImagePullBackOff`, `command not found`), checking GitHub Actions logs, and validating endpoints in the browser after `kubectl port-forward`.

### Future architectural outlook

With an extra week or in a production setting, I would:

1. **Ingress** — Replace `kubectl port-forward` with an NGINX Ingress Controller and TLS (e.g. cert-manager)
2. **GitOps** — Use Argo CD or Flux so cluster state tracks Git automatically
3. **Observability** — Add Prometheus metrics and Grafana dashboards for pod health and latency
4. **CI security** — Integrate Trivy or Snyk image scanning in the pipeline before push to GHCR
5. **Secrets management** — Move from static Base64 in YAML to External Secrets Operator or a vault
6. **Environments** — Separate `dev` / `staging` / `prod` namespaces with promoted image tags instead of only `:latest`

---

## Submission checklist (PDF)

| # | Evidence |
|---|----------|
| 1 | Public GitHub repo link |
| 2 | Screenshot: successful GitHub Actions run |
| 3 | Screenshot: `kubectl get all -n default` + `kubectl get configmap,secret` |
| 4 | Screenshots: `http://localhost:8080/` and `http://localhost:8080/secure-config` (with port-forward running) |
| 5 | AI reflection section (above) |

---

## Assignment specification (summary)

| Task | Points | Deliverable |
|------|--------|-------------|
| 1 — Containerization | 15 | `Dockerfile` |
| 2 — CI/CD | 25 | `.github/workflows/ci-cd.yaml` → GHCR |
| 3 — Orchestration | 40 | `k8s/*.yaml` on Minikube |
| 4 — Documentation | 20 | This README + port-forward validation + AI report |

---

## Contact

For assignment questions: gtheodorou@aueb.gr
