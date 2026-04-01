# Docker and Kubernetes Related Files

This document provides a list of Docker and Kubernetes related files within the project, including their paths and descriptions.

## 🐳 Docker Related Files
| File Name | Path | Description |
| :--- | :--- | :--- |
| `Dockerfile` | [nitrocloud/infra-engine/Dockerfile](file:///Users/admin/Desktop/fresh_projects/nitrocloud/infra-engine/Dockerfile) | Main Dockerfile for the infra-engine. |
| `.dockerignore` | [nitrocloud/infra-engine/.dockerignore](file:///Users/admin/Desktop/fresh_projects/nitrocloud/infra-engine/.dockerignore) | Docker ignore rules for the infra-engine. |
| `Dockerfile` | [nitrocloud/infra-engine/docker_files/mcp-server/Dockerfile](file:///Users/admin/Desktop/fresh_projects/nitrocloud/infra-engine/docker_files/mcp-server/Dockerfile) | Dockerfile for the MCP server. |
| `.dockerignore` | [nitrocloud/infra-engine/docker_files/mcp-server/.dockerignore](file:///Users/admin/Desktop/fresh_projects/nitrocloud/infra-engine/docker_files/mcp-server/.dockerignore) | Docker ignore rules for the MCP server. |
| `Dockerfile` | [nitrocloud/scheduler-service/Dockerfile](file:///Users/admin/Desktop/fresh_projects/nitrocloud/scheduler-service/Dockerfile) | Dockerfile for the scheduler-service. |
| `Dockerfile.linux` | [nitrostudio/studio/Dockerfile.linux](file:///Users/admin/Desktop/fresh_projects/nitrostudio/studio/Dockerfile.linux) | Linux specific Dockerfile for the studio. |
| `Dockerfile` | [nitrostudio/gateway/Dockerfile](file:///Users/admin/Desktop/fresh_projects/nitrostudio/gateway/Dockerfile) | Dockerfile for the gateway. |
| `docker-compose.yml` | [nitrostudio/gateway/docker-compose.yml](file:///Users/admin/Desktop/fresh_projects/nitrostudio/gateway/docker-compose.yml) | Docker Compose configuration for the gateway. |

## ☸️ Kubernetes Related Files
| File Name | Path | Description |
| :--- | :--- | :--- |
| `kubeconfig-nitrocloud.yaml` | [nitrocloud/base-architecture/kubeconfig-nitrocloud.yaml](file:///Users/admin/Desktop/fresh_projects/nitrocloud/base-architecture/kubeconfig-nitrocloud.yaml) | Kubernetes configuration file for the NitroCloud cluster. |
| `updated-prometheus-rule.yaml` | [nitrocloud/base-architecture/updated-prometheus-rule.yaml](file:///Users/admin/Desktop/fresh_projects/nitrocloud/base-architecture/updated-prometheus-rule.yaml) | Kubernetes Prometheus monitoring rules. |
| `Pulumi.yaml` | [nitrocloud/base-architecture/Pulumi.yaml](file:///Users/admin/Desktop/fresh_projects/nitrocloud/base-architecture/Pulumi.yaml) | Infrastructure as code, likely managing Kubernetes resources. |
| `Pulumi.dev.yaml` | [nitrocloud/base-architecture/Pulumi.dev.yaml](file:///Users/admin/Desktop/fresh_projects/nitrocloud/base-architecture/Pulumi.dev.yaml) | Development environment configuration for Pulumi. |

## 🧪 Workflows & Other Configuration (YAML)
- `nitrostudio/gateway/buildspec.yml`: AWS CodeBuild build specification.
- `nitrostudio/gateway/docs/swagger.yaml`: API documentation for the gateway.
- `nitrostudio/.github/workflows/`: Contains several CI/CD pipeline definitions (e.g., `release.yml`, `build-linux.yml`, `studio-release.yml`).
