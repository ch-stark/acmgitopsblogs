Before you begin, ensure you have the following:

* A **Red Hat Advanced Cluster Management (RHACM)** Hub Cluster (version 2.13 or higher recommended).
* One or more **Managed Clusters** imported into RHACM and in a `Ready` state.
* The **OpenShift GitOps operator** (which provides Argo CD) installed on the Hub Cluster, typically in the `openshift-gitops` namespace. This will be our Argo CD control plane.
* A **Git repository** accessible by the Hub Cluster's Argo CD instance. This repository will store your application manifests and namespace configuration.
* The `oc` (OpenShift CLI) command-line tool configured to access your Hub Cluster.
* Cluster-admin privileges on the Hub Cluster for the initial Argo CD and RBAC setup.

---

## Step 1: Configure Argo CD for "Applications in any Namespace"

To allow Argo CD on the Hub to manage `Application` resources in namespaces other than `openshift-gitops`, you need to configure it.

1.  **Define Allowed Namespaces in Argo CD:**
    Patch the `argocd-cmd-params-cm` ConfigMap in the `openshift-gitops` namespace on your Hub Cluster. This tells the Argo CD controllers which namespaces they are allowed to watch for `Application` resources.

    ```bash
    oc patch configmap argocd-cmd-params-cm -n openshift-gitops --type merge -p \
    '{"data":{"application.namespaces":"openshift-gitops,devteam1,devteam2,appadmin"}}'
    ```

2.  **Restart Argo CD Components:**
    For the changes to take effect, restart the relevant Argo CD deployments on the Hub Cluster.

    ```bash
    oc rollout restart deployment/openshift-gitops-server -n openshift-gitops
    oc rollout restart statefulset/openshift-gitops-application-controller -n openshift-gitops
    ```

3.  **Adapt Kubernetes RBAC for Argo CD Server:**
    The `openshift-gitops-argocd-server` service account needs permissions to list, get, and watch `Application` resources in the namespaces you've enabled.

    Create a file named `argocd-server-any-ns-rbac.yaml`:
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: argocd-server-view-apps-any-ns
    rules:
    - apiGroups: ["argoproj.io"]
      resources: ["applications"]
      verbs: ["get", "list", "watch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: argocd-server-view-apps-any-ns
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: argocd-server-view-apps-any-ns
    subjects:
    - kind: ServiceAccount
      name: openshift-gitops-argocd-server
      namespace: openshift-gitops
    ```
    Apply this RBAC configuration:
    ```bash
    oc apply -f argocd-server-any-ns-rbac.yaml
    ```
    *Note: For full CRUD operations from the UI/CLI on Applications in other namespaces, you might need to grant more verbs like `create`, `update`, `patch`, `delete` to this ClusterRole, or use a more specific Role/RoleBinding per namespace if finer-grained control is desired.*

---

## Step 2: Structure Your Git Repository

Organize your Git repository to hold application manifests and the configuration that dictates their target namespaces.

Create the following directory structure:

├── common-apps
│   ├── devteam1-app
│   │   ├── deployment.yaml
│   │   └── params.json
│   ├── devteam2-app
│   │   ├── deployment.yaml
│   │   └── params.json
│   └── appadmin-tool
│       ├── deployment.yaml
│       └── params.json


**File Contents:**

* **`common-apps/devteam1-app/params.json`**:
    This file defines parameters for the `devteam1-app`, including its target namespace on both the Hub (for the `Application` CR) and the Managed Cluster (for the workload).
    ```json
    {
      "hub_app_namespace": "devteam1",
      "managed_cluster_deploy_namespace": "devteam1",
      "app_name_suffix": "dt1"
    }
    ```

* **`common-apps/devteam1-app/deployment.yaml`**:
    A simple deployment for testing.
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-app-devteam1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hello-devteam1
      template:
        metadata:
          labels:
            app: hello-devteam1
            team: devteam1
        spec:
          containers:
          - name: hello-openshift
            image: openshift/hello-openshift
            ports:
            - containerPort: 8080
    ```

* **`common-apps/devteam2-app/params.json`**:
    ```json
    {
      "hub_app_namespace": "devteam2",
      "managed_cluster_deploy_namespace": "devteam2",
      "app_name_suffix": "dt2"
    }
    ```

* **`common-apps/devteam2-app/deployment.yaml`**:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-app-devteam2
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hello-devteam2
      template:
        metadata:
          labels:
            app: hello-devteam2
            team: devteam2
        spec:
          containers:
          - name: hello-world
            image: gcr.io/google-samples/hello-app:1.0
            ports:
            - containerPort: 8080
    ```

* **`common-apps/appadmin-tool/params.json`**:
    ```json
    {
      "hub_app_namespace": "appadmin",
      "managed_cluster_deploy_namespace": "appadmin",
      "app_name_suffix": "adm"
    }
    ```

* **`common-apps/appadmin-tool/deployment.yaml`**:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: admin-tool-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: admin-tool
      template:
        metadata:
          labels:
            app: admin-tool
            role: admin
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
    ```

Commit and push these files to your Git repository.

---

## Step 3: Create RHACM Placement and Argo CD AppProject

These resources on the Hub Cluster will define where applications can be deployed and what permissions they have.

1.  **Create the RHACM `Placement`:**
    This selects the Managed Clusters. Create `placement-all-managed.yaml`:
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1beta1
    kind: Placement
    metadata:
      name: deploy-to-all-managed
      namespace: openshift-gitops # ApplicationSet will reference this
    spec:
      # Example: Selects all clusters. Customize with clusterSets or predicates as needed.
      predicates:
      - requiredClusterSelector:
          labelSelector:
            matchExpressions: [] # Empty matches all
    ```
    Apply it: `oc apply -f placement-all-managed.yaml -n openshift-gitops`

2.  **Create the Argo CD `AppProject`:**
    This project will authorize the creation of `Application` resources in the `devteam1`, `devteam2`, and `appadmin` namespaces on the Hub. Create `appproject-teams.yaml`:
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
      name: team-applications
      namespace: openshift-gitops # AppProject must be in Argo CD's namespace
    spec:
      description: Project for applications deployed to team and admin namespaces
      sourceRepos:
      - '*' # Allow all Git repos, or restrict to yours
      destinations:
      - namespace: '*' # Allows deployment to any namespace on managed clusters
        server: '*'   # Allows deployment to any server (managed cluster)
      clusterResourceWhitelist:
      - group: '*'
        kind: '*'
      # This is crucial: allows Application CRs to be created in these namespaces on the Hub
      sourceNamespaces:
      - openshift-gitops # For the ApplicationSet itself, if needed, or other central apps
      - devteam1
      - devteam2
      - appadmin
    ```
    Apply it: `oc apply -f appproject-teams.yaml -n openshift-gitops`

---

## Step 4: Create the Master ApplicationSet

This `ApplicationSet` will reside in `openshift-gitops` on the Hub Cluster. It will discover applications from your Git repository and generate `Application` CRs in the appropriate Hub namespaces (`devteam1`, `devteam2`, `appadmin`).

Create `master-appset.yaml`. **Replace `<YOUR_GIT_REPO_URL>` with your actual Git repository URL.**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-namespace-deployment-set
  namespace: openshift-gitops # ApplicationSet itself is in the Argo CD namespace
spec:
  generators:
  - matrix:
      generators:
        # Generator 1: Get managed clusters from RHACM Placement
        - clusterDecisionResource:
            configMapRef: acm-placement # This is a default, ensure it points to OCM/RHACM
            name: deploy-to-all-managed # Matches the Placement resource name
            requeueAfterSeconds: 180
        # Generator 2: Get app configurations from Git
        - git:
            repoURL: <YOUR_GIT_REPO_URL> # !!! REPLACE THIS !!!
            revision: main # Or your target branch/tag
            files:
            - path: "common-apps/**/params.json"
  template:
    # Defines the Application CR that will be created on the HUB CLUSTER
    metadata:
      name: '{{path.basename}}-{{values.app_name_suffix}}-{{name}}' # e.g., devteam1-app-dt1-cluster1
      namespace: '{{values.hub_app_namespace}}' # CRUCIAL: Creates App CR in devteam1, devteam2, or appadmin on Hub
      labels:
        team: '{{values.managed_cluster_deploy_namespace}}' # For easier filtering
        rhacm-appset-generated: "true"
    spec:
      project: team-applications # Must match the AppProject name
      source:
        repoURL: <YOUR_GIT_REPO_URL> # !!! REPLACE THIS !!!
        targetRevision: main
        path: '{{path}}' # Path to the app's manifests (e.g., common-apps/devteam1-app)
      destination:
        # Defines where the app's resources are deployed ON THE MANAGED CLUSTER
        server: '{{server}}' # From clusterDecisionResource (managed cluster URL)
        namespace: '{{values.managed_cluster_deploy_namespace}}' # Target namespace on managed cluster
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true # Argo CD will create the namespace on the managed cluster if it doesn't exist
Apply it: oc apply -f master-appset.yaml -n openshift-gitops

Step 5: Verification
On the Hub Cluster:

Check ApplicationSet Status:
Bash

oc get applicationset multi-namespace-deployment-set -n openshift-gitops -o yaml
Look for conditions indicating successful generation of Application resources.
Check Generated Application CRs: Verify that Application custom resources have been created in the devteam1, devteam2, and appadmin namespaces on the Hub.
Bash

oc get applications -n devteam1
oc get applications -n devteam2
oc get applications -n appadmin
You should see applications like devteam1-app-dt1-clusterXYZ, etc.
Check Argo CD UI: Open the Argo CD UI on your Hub cluster. You should see the generated applications. If you filter by project (team-applications), they should appear. You might need to adjust RBAC for your user to see applications in these non-default namespaces in the UI.
On the Managed Clusters (RHACM Pull Model in Action):

RHACM's application-manager on the Hub detects the Application CRs and creates corresponding ManifestWork resources targeting the managed clusters.
The Klusterlet agent on each managed cluster applies these ManifestWork resources, which creates the Application CR on the managed cluster.
The Argo CD instance/agent on the managed cluster reconciles this Application CR.
Verify Namespaces: Switch your oc context to a managed cluster.
Bash

oc get ns | grep -E "devteam1|devteam2|appadmin"
The namespaces should have been created by Argo CD due to CreateNamespace=true.
Verify Deployments: Check if the pods are running in their respective namespaces on the managed cluster.
Bash

# On a managed cluster
oc get pods -n devteam1
oc get pods -n devteam2
oc get pods -n appadmin
You should see hello-app-devteam1-*, hello-app-devteam2-*, and admin-tool-deployment-* pods running.
Check ManifestWork (Advanced): On the Hub cluster, you can inspect the ManifestWork resources created for each managed cluster. The namespace for ManifestWork is the managed cluster's import namespace on the hub (e.g., cluster1).
Bash

# Replace <managed_cluster_namespace_on_hub> with the actual namespace, e.g., cluster1, cluster2
oc get manifestwork -n <managed_cluster_namespace_on_hub>
This tutorial provides a comprehensive setup for deploying applications to distinct namespaces on your managed clusters using a centralized ApplicationSet on the RHACM Hub, leveraging the "Applications in any Namespace" feature of Argo CD and the RHACM pull model. Remember to adapt repository URLs, placement rules, and application details to your specific environment.
