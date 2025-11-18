# OpenShift AI 3.0 App-of-Apps

This repository implements a GitOps-based deployment strategy for OpenShift AI (OCPAI) 3.0 platform using Argo CD's app-of-apps pattern. It provides a declarative, version-controlled approach to managing operators, platform components, and applications across OpenShift clusters.

## Project Overview

This project deploys a complete OpenShift AI 3.0 platform stack, including:

- **Operators**: Core operators required for the AI/ML platform (NVIDIA GPU Operator, OpenShift AI Operator, Node Feature Discovery, Kueue, Cert Manager, Authorino, Service Mesh, Serverless, and more)
- **Platform Components**: Essential infrastructure services including Milvus (vector database), MinIO (object storage), ODH Tools, and NVIDIA GPU cluster policies
- **Applications**: Sample applications like the shakeout-app for testing and validation

All components are managed through Argo CD using Kustomize for configuration management, enabling environment-specific overlays and consistent deployments.

## Repository Structure

ocpai3-app-of-apps/  
├── clusters/                      # Argo CD Application definitions (app-of-apps)  
│ ├── root-application.yaml        # Root application entry point  
│ └── ocpai3/                      # OCPAI3 platform application definitions  
│ ├── -operator-application.yaml   # Operator subscriptions  
│ ├── -application.yaml            # Platform component applications  
│ └── tenants/                     # Tenant-specific configurations  
│  
├── components/                    # Kubernetes manifests and configurations  
│ ├── operators/                   # Operator subscriptions (OLM)  
│ │ ├── nvidia/                    # NVIDIA GPU Operator  
│ │ ├── ocpai/                     # OpenShift AI Operator  
│ │ ├── nfd/                       # Node Feature Discovery  
│ │ ├── kueue/                     # Kueue Operator  
│ │ └── ...                        # Additional operators  
│ │  
│ └── platform/                    # Platform component deployments  
│ ├── milvus/                      # Milvus vector database  
│ ├── minio/                       # MinIO object storage  
│ ├── odh-tools/                   # Open Data Hub tools  
│ ├── nvidia-instance/             # NVIDIA GPU cluster policies  
│ ├── nfd-instance/                # Node Feature Discovery instance  
│ └── ocpai/                       # Data Science Cluster (DSC) instance  
│  
├── shakeout-app/                  # Test application for validation  
│ ├── base/                        # Base configuration  
│ └── overlays/                    # Environment-specific overlays (aws, rosa, lvm)  
│  
└── tools/                         # Bootstrap and cluster management scripts  
├── bootstrap-cluster.sh           # Initial cluster GitOps setup  
└── rosa/                          # ROSA-specific utilities  
├── create-machinepool.sh          # Machine pool creation helper  
└── grant-user-cluster-admin.sh    # User permission management  


### Key Directories

- **`clusters/`**: Contains Argo CD `Application` custom resources that define what to deploy and where. The root application points to the `ocpai3/` directory, which contains all platform-specific applications.
- **`components/`**: Contains the actual Kubernetes manifests organized by type:
  - **`operators/`**: Operator subscriptions managed via OLM (Operator Lifecycle Manager)
  - **`platform/`**: Application deployments, custom resources, and platform services
- **`shakeout-app/`**: A test application used to validate storage classes, routes, and network connectivity
- **`tools/`**: Utility scripts for cluster bootstrapping and management

## Deployment with Argo CD

This project uses Argo CD's **app-of-apps pattern** to manage deployments. Here's how it works:

### 1. Root Application

The entry point is `clusters/root-application.yaml`, which defines an Argo CD Application that:
- Points to the `clusters/ocpai3/` directory in this repository
- Deploys to the `openshift-gitops` namespace
- Uses sync wave `0` to ensure it deploys first

### 2. App-of-Apps Pattern

The `clusters/ocpai3/` directory contains multiple Argo CD Application definitions, each referencing components in the `components/` directory:

- **Operator Applications**: Deploy operator subscriptions (e.g., `nvidia-operator-application.yaml`, `ocpai-operator.yaml`)
- **Platform Applications**: Deploy platform components (e.g., `milvus-application.yaml`, `minio-application.yaml`)
- **Instance Applications**: Deploy operator instances and custom resources (e.g., `nvidia-cluster-policy-application.yaml`, `ocpai-dcs-instance.yaml`)

### 3. Sync Waves

Applications use `argocd.argoproj.io/sync-wave` annotations to control deployment order:
- **Wave 0**: Root application
- **Wave 10-11**: Core operators (NVIDIA, OCPAI)
- **Wave 20-30**: Platform components and instances

### 4. Deployment Steps

1. **Bootstrap Argo CD**: Run `tools/bootstrap-cluster.sh` to install OpenShift GitOps and configure permissions (see Tools section below)

2. **Create Root Application**: Apply the root application to Argo CD:
   ```bash
   oc apply -f clusters/root-application.yaml
   ```

3. **Sync Applications**: Argo CD will automatically discover and deploy all applications defined in `clusters/ocpai3/`. You can:
   - Monitor via Argo CD UI (route available after bootstrap)
   - Use Argo CD CLI: `argocd app sync <app-name>`
   - Sync all apps: `argocd app sync -l app.kubernetes.io/part-of=platform`

4. **Manual Sync**: Applications are configured with manual sync by default. Changes require manual sync from the Argo CD UI or CLI.

### 5. Configuration Management

- Components use **Kustomize** for configuration management
- Environment-specific overlays are available (e.g., `minio/overlays/rosa/` for ROSA-specific storage configurations)
- All manifests are version-controlled and declarative

## Tools Directory - Cluster Bootstrapping

The `tools/` directory contains scripts for initial cluster setup and management:

### `bootstrap-cluster.sh`

This is the primary bootstrapping script for setting up a new OpenShift cluster with GitOps capabilities. It:

1. **Installs OpenShift GitOps Operator**: Subscribes to the OpenShift GitOps operator from the Red Hat Operator catalog
2. **Configures Permissions**: Grants `cluster-admin` role to the Argo CD application controller service account, enabling it to:
   - Create namespaces automatically (`CreateNamespace=true`)
   - Deploy resources across all namespaces
   - Manage cluster-scoped resources
3. **Verifies Installation**: Checks that the operator is installed and permissions are correctly configured
4. **Provides Troubleshooting**: Includes helpful commands for diagnosing permission issues

**Prerequisites**:
- Authenticated to OpenShift cluster with `cluster-admin` privileges
- `oc` CLI installed and configured

**Usage**:
```bash
./tools/bootstrap-cluster.sh
```

### ROSA Utilities (`tools/rosa/`)

Additional scripts for managing Red Hat OpenShift Service on AWS (ROSA) clusters:

- **`create-machinepool.sh`**: Interactive script to add bare metal or GPU-enabled worker nodes to a ROSA cluster. It:
  - Prompts for AWS profile selection
  - Allows selection of instance types (bare metal or GPU-enabled)
  - Creates machine pools with configurable replica counts
  - Supports both AWS CLI v2 and ROSA CLI

- **`grant-user-cluster-admin.sh`**: Utility to grant cluster-admin privileges to a specific user on a ROSA cluster

## Additional Notes

- All applications use manual sync policies for safety
- Applications include `ApplyOutOfSyncOnly=true` sync option for efficient updates
- Namespace creation is automated via `CreateNamespace=true` sync option
- The repository structure supports multiple cluster configurations (see `clusters/finsight-app-project/` for an example)
- Platform components include overlays for different deployment environments (AWS, ROSA, LVM storage)

## Contributing

When adding new components:
1. Create manifests in `components/operators/` or `components/platform/`
2. Add corresponding Application definition in `clusters/ocpai3/`
3. Set appropriate sync wave annotations for deployment ordering
4. Test with the shakeout-app before deploying to production

## References

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai/)
- [Kustomize Documentation](https://kustomize.io/)

