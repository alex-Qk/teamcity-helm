# TeamCity Server Kubernetes Manifests

This directory contains simplified Kubernetes manifests converted from the original Helm chart. The manifests are designed to deploy a high-availability TeamCity server setup with:

- 2 TeamCity server instances in a StatefulSet
- HAProxy load balancer for routing
- PostgreSQL database (via Bitnami Helm chart)
- Persistent storage for TeamCity data
- Ingress configuration for external access

## Prerequisites

### 1. Install PostgreSQL Database

Install PostgreSQL using the Bitnami Helm chart:

```bash
# Create namespace first
kubectl create namespace teamcity

# Install PostgreSQL
helm install postgresql bitnami/postgresql \
  --namespace teamcity \
  --set auth.username=teamcity \
  --set auth.database=teamcity \
  --set auth.postgresPassword=teamcity-admin \
  --set auth.password=teamcity-pass \
  --set primary.persistence.size=10Gi
```

**For production**, use Kubernetes Secrets for better security:
```bash
# Create secret for passwords
kubectl create secret generic postgresql-passwords \
  --namespace teamcity \
  --from-literal=postgres-password=teamcity-admin \
  --from-literal=password=teamcity-pass

# Install with secret reference
helm install postgresql bitnami/postgresql \
  --namespace teamcity \
  --set auth.username=teamcity \
  --set auth.database=teamcity \
  --set auth.existingSecret=postgresql-passwords \
  --set primary.persistence.size=10Gi
```

### 2. Deploy TeamCity

Apply the manifests in the following order:

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-pvc.yaml
kubectl apply -f 03-configmaps.yaml
kubectl apply -f 04-services.yaml
kubectl apply -f 05-statefulset.yaml
kubectl apply -f 06-proxy-deployment.yaml
kubectl apply -f 07-ingress.yaml
kubectl apply -f 08-pdb.yaml
# kubectl apply -f 09-serviceaccount-rbac.yaml  # Optional - only if using Kubernetes agents
```

Or apply all at once:
```bash
kubectl apply -f k8s-manifests/
```

**Note**: The ServiceAccount and RBAC resources in `09-serviceaccount-rbac.yaml` are commented out by default (matching the original Helm chart). Uncomment them only if you need TeamCity to manage Kubernetes agents.

## Configuration Changes

Key values that have been hardcoded from the original Helm chart:

- **Namespace**: `teamcity`
- **Replicas**: 2 TeamCity servers, 2 proxy instances
- **Storage**: Uses `efs` storage class with 16Gi
- **Database**: PostgreSQL connection to `postgresql.teamcity:5432`
- **Images**: 
  - TeamCity: `jetbrains/teamcity-server:latest`
  - HAProxy: `haproxy:3.0`
- **Ingress hosts**: 
  - `teamcity.example.com`
  - `teamcity.isolated.example.com`
  - `teamcity-main.example.com`

## Customization

To customize the deployment:

1. Update database connection in `03-configmaps.yaml`
2. Change ingress hostnames in `07-ingress.yaml`
3. Modify resource requests/limits in the StatefulSet and Deployment
4. Update storage class and size in `02-pvc.yaml`
5. Change replica counts in the StatefulSet and Deployment specs

## TeamCity Multinode Setup

This deployment features **automatic node configuration** that:

- **Automatically assigns JVM options** based on pod ordinal
- **Handles database initialization** intelligently 
- **Sets proper node responsibilities** for main vs secondary nodes
- **Prevents startup issues** with smart initialization logic

### Automated Features

#### **Main Node (teamcity-server-0)**:
- ✅ **Initial Setup**: Starts without multinode config during first run
- ✅ **Post-Setup**: Automatically enables full multinode configuration  
- ✅ **Responsibilities**: `MAIN_NODE,CAN_PROCESS_BUILD_TRIGGERS,CAN_PROCESS_USER_DATA_MODIFICATION_REQUESTS,CAN_CHECK_FOR_CHANGES,CAN_PROCESS_BUILD_MESSAGES`

#### **Secondary Node (teamcity-server-1)**:
- ✅ **Smart Startup**: Only starts after main node initializes database
- ✅ **Auto-Configuration**: Automatically joins as secondary node
- ✅ **Responsibilities**: `CAN_PROCESS_BUILD_TRIGGERS,CAN_PROCESS_USER_DATA_MODIFICATION_REQUESTS,CAN_CHECK_FOR_CHANGES,CAN_PROCESS_BUILD_MESSAGES`

### Simplified Setup Process

### 1. Deploy Both Nodes
```bash
# Deploy with both nodes - they'll coordinate automatically
kubectl apply -f k8s-manifests/
```

### 2. Initialize Main Node
1. Access TeamCity via your ingress host (e.g., `teamcity.example.com`)
2. Complete the initial setup wizard (main node starts in single-node mode)
3. Configure database connection (PostgreSQL details are pre-configured)
4. Set up your first admin user

### 3. Automatic Multinode Activation
1. **Restart the main node** to enable multinode configuration:
   ```bash
   kubectl rollout restart statefulset teamcity-server -n teamcity
   ```
2. **Secondary node automatically joins** with proper configuration
3. **No manual UI configuration needed!**

### 4. Verify Setup
1. Check **Administration > Diagnostics > Server Health** for both nodes
2. Verify load balancing through HAProxy stats at `/healthz`
3. Test failover by stopping one node

### Smart Startup Logic
The startup script automatically:
- ✅ **Detects database state** (`dataDirectoryInitialized` file)
- ✅ **Configures JVM options** based on hostname and initialization state
- ✅ **Assigns node responsibilities** automatically
- ✅ **Prevents secondary nodes** from starting before main node is ready
- ✅ **Handles config backup** for main node during initial setup

For detailed HA configuration, see: https://www.jetbrains.com/help/teamcity/multinode-setup.html

## Access

- Main TeamCity UI: Access via configured ingress hosts
- HAProxy stats: Available at `/healthz` endpoint on the main proxy service (port 80)
- Direct node access: Use `teamcity-main.example.com` for the main node
- Individual nodes:
  - Node 0: `teamcity-server-direct-0.teamcity:8111` (internal)
  - Node 1: `teamcity-server-direct-1.teamcity:8111` (internal)