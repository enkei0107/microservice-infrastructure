# 🛠️ Infrastructure Documentation - Microservices Architecture

## 📁 Overview

This document outlines the infrastructure setup used to manage and deploy a scalable, reliable microservices system using gRPC, Docker, Kubernetes, and GitOps strategies. It details how shared resources are structured, how services are deployed, and how routing and CI/CD pipelines are managed.

---

## 🧬 Architecture Overview

* **Protocol Buffers (Protobuf):**

    * Shared gRPC definitions are maintained in a central repository.
    * All service contracts are defined using `.proto` files.
    * Buf is used to lint, break-check, and maintain version control of protobufs.

* **Microservice Consumption:**

    * Each microservice repository includes the shared proto repo as a `git submodule`.
    * Services regenerate gRPC and gRPC-Gateway code from the submodule on build.

* **Containerization with Docker:**

    * All microservices are built into Docker images.
    * Base images are language-specific (e.g. `golang:1.22`, `node:20`, `python:3.11`).
    * Multi-stage builds are used for production images.

* **Orchestration with Kubernetes:**

    * A Kubernetes cluster with multiple nodes distributes service workloads.
    * Each environment (`dev`, `staging`, `production`) is deployed in its own namespace.

* **Routing with gRPC-Gateway and Kong:**

    * gRPC-Gateway is used for HTTP/REST clients to access gRPC services.
    * Kong API Gateway handles routing, load balancing, authentication, and rate limiting.

* **Container Registry:**

    * Docker images are pushed to GitHub Container Registry (`ghcr.io`) or GitLab Container Registry.
    * Pull secrets are set up in Kubernetes to access private images.

* **Git Branch Environment Mapping:**

    * `dev`: Development (in-progress features)
    * `staging`: QA/Testing environment
    * `production`: Stable live deployment

---

## 🗃️ Shared Protobuf Repository

* Maintained in a central repo with Buf.
* Folder structure:

  ```sh
  /proto
    /auth
    /user
    /order
  buf.yaml
  buf.lock
  ```
* Validated via `buf lint` and breaking changes detected using `buf breaking`.

## 📦 Microservice Repositories

Each microservice contains:

```sh
/api (git submodule)
/internal
/cmd
/pkg
Dockerfile
```

* Use cli to regenerate code:

  ```sh
   buf generate
  ```
* Communicates with other services via gRPC clients.
* Implements REST via gRPC-Gateway.

---

## 🐳 Docker and Registry

Each service has a `Dockerfile` like:

```Dockerfile
FROM golang:1.22 as builder
WORKDIR /app
COPY . .
RUN make build

FROM alpine
COPY --from=builder /app/bin/service /bin/service
ENTRYPOINT ["/bin/service"]
```

### Registry:

* GitHub: `ghcr.io/org/service:tag`
* GitLab: `registry.gitlab.com/org/project/service:tag`

### Tags:

* `dev-<commit-hash>`
* `staging-v1.2.3`
* `prod-v1.2.3`

---

## ☸️ Kubernetes Infrastructure

### Cluster Setup:

* Managed via cloud provider or self-hosted.
* Uses kubelet, kube-proxy, CoreDNS, etc.
* Auto-scaling node pools enabled.

### Deployment:

Each environment is represented by a directory:

```sh
/k8s/dev/
/k8s/staging/
/k8s/production/
```

With files like:

```yaml
- deployment.yaml
- service.yaml
- ingress.yaml
- configmap.yaml
```

### Namespaces:

* `dev`, `staging`, and `prod` for isolation.

### Observability:

* Integrated with Prometheus + Grafana
* Loki for logs
* Kiali for service mesh

---

## 🌐 API Gateway Setup

### gRPC-Gateway

* Exposes REST/JSON endpoints by translating to gRPC calls.
* Generates Swagger/OpenAPI spec from proto files.

### Kong Gateway

* Runs in Kubernetes with Helm chart
* Features used:

    * Load balancing
    * JWT/OAuth2 plugins
    * Rate limiting & CORS
    * TLS termination

Example route:

```
/api/v1/users -> grpc-gateway -> user-service
```

---

## 🧪 CI/CD Pipelines

### CI Pipeline (on push):

* Lint, Test, Build Docker image
* Push to registry
* Generate proto code (if needed)

### CD Pipeline (per branch):

* `dev` ➜ Deploy to dev namespace
* `staging` ➜ Deploy to staging namespace
* `production` ➜ Deploy to prod namespace

Example (GitHub Actions):

```yaml
jobs:
  deploy:
    steps:
      - uses: actions/checkout@v3
      - run: make docker-build && make docker-push
      - run: kubectl apply -f k8s/$ENV/
```

---

## 🔐 Secrets and Configuration

* Managed using Kubernetes Secrets and ConfigMaps
* Use SealedSecrets for encrypted GitOps
* Env-specific values:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: host
```

---

## 📋 Environments Overview

| Branch       | Namespace | Domain                  | Purpose                  |
| ------------ | --------- | ----------------------- | ------------------------ |
| `dev`        | `dev`     | dev.api.example.com     | Developer testing        |
| `staging`    | `staging` | staging.api.example.com | QA & UAT                 |
| `production` | `prod`    | api.example.com         | Public facing production |

Each has a dedicated config, image tag policy, and resource quotas.

---

## 🧰 Tooling Summary

| Tool          | Purpose                                    |
| ------------- | ------------------------------------------ |
| Buf           | Protobuf linting and breaking change guard |
| Docker        | Container build and packaging              |
| Kubernetes    | Orchestration and deployment               |
| gRPC          | Service-to-service communication           |
| gRPC-Gateway  | REST ↔ gRPC translation                    |
| Kong          | API Gateway & routing                      |
| GitHub/GitLab | Version control and CI/CD                  |
| Prometheus    | Metrics collection                         |
| Grafana       | Metrics dashboard                          |
| Loki          | Log aggregation                            |
| SealedSecrets | Encrypted Kubernetes secrets in GitOps     |

---

## ✅ Best Practices

* Maintain backward compatibility in proto definitions.
* Use `make gen-proto` and keep generated files committed when needed.
* Store secrets securely and avoid hardcoding.
* Use namespace and RBAC separation for each environment.
* Monitor and log all services with alerts.
* Perform automated integration tests in CI pipelines.

---

Feel free to customize this infrastructure to fit specific scaling, security, or compliance needs!
