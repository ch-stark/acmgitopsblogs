---

### Prerequisites

* A **Red Hat Advanced Cluster Management (RHACM)** Hub Cluster (v2.4+ recommended).
* One or more **Managed Clusters** imported into RHACM and in a `Ready` state.
* The **OpenShift GitOps operator** installed on the **Hub Cluster** (typically in `openshift-gitops`).
* A **Git repository** accessible by Argo CD instances on both Hub and Managed Clusters.
* `oc` (OpenShift CLI) configured to access your Hub Cluster.
* Cluster-admin privileges on the Hub Cluster for initial setup and policy creation.

---

### Step 1: Configure the Hub Cluster

First, we will prepare the RHACM Hub Cluster by configuring its local Argo CD instance and creating the necessary project and placement resources.

#### 1.1. Enable "Applications in any Namespace" on the Hub

This allows the Hub's Argo CD to manage `Application` resources in namespaces beyond the default `openshift-gitops`, which is essential for our multi-team setup.

1.  **Define Allowed Namespaces:**
    ```bash
    oc patch configmap argocd-cmd-params-cm -n openshift-gitops --type merge -p \
    '{"data":{"application.namespaces":"openshift-gitops,devteam1,devteam2,appadmin"}}'
    ```

2.  **Restart Argo CD Components:**
    ```bash
    oc rollout restart deployment/openshift-gitops-server -n openshift-gitops
    oc rollout restart statefulset/openshift-gitops-application-controller -n openshift-gitops
    ```

3.  **Adapt RBAC for Argo CD Server:**
    This allows the Argo CD UI/CLI on the Hub to see the `Application` resources in the new namespaces. Create `argocd-server-any-ns-rbac.yaml`:
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
    Apply it: `oc apply -f argocd-server-any-ns-rbac.yaml`

#### 1.2. Create the Argo CD `AppProject`

The `AppProject` defines the permissions for our generated applications, including which source repositories they can use and which namespaces they are allowed to be created in.

Create `appproject-teams.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-applications
  namespace: openshift-gitops
spec:
  description: Project for applications deployed to team and admin namespaces
  sourceRepos:
  - '*' # Or restrict to your specific Git repository URL
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  # This is crucial: it allows Application CRs to be created in these namespaces on the Hub
  sourceNamespaces:
  - openshift-gitops
  - devteam1
  - devteam2
  - appadmin
Apply it: oc apply -f appproject-teams.yaml -n openshift-gitops

1.3. Create the Placement for the ApplicationSet
This Placement resource will be used by our ApplicationSet to dynamically discover which managed clusters to target for deployments.

Create placement-appset-targets.yaml:

YAML

apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: appset-target-all-managed
  namespace: openshift-gitops # The ApplicationSet will reference this Placement
spec:
  clusterSets:
    - global # This example targets all clusters in the 'global' ClusterSet
Apply it: oc apply -f placement-appset-targets.yaml -n openshift-gitops

Step 2: Prepare Managed Clusters with the gitops-addon
Next, we will prepare our managed clusters by deploying the OpenShift GitOps Operator and an Argo CD instance. We will use the modern, all-in-one gitops-addon, which handles this entire process automatically.

Create the ClusterManagementAddOn Resource:
This resource makes the gitops-addon available across the fleet. Create cma-gitops-addon.yaml:

YAML

apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: gitops-addon
spec:
  addOnMeta:
    displayName: GitOps Addon
    description: Installs and configures OpenShift GitOps for the pull model.
  installStrategy:
    type: Placements
    placements:
    - name: placement-enable-gitops-addon # The Placement we will create next
      namespace: open-cluster-management # A common namespace for shared placements
Create the Placement for the Addon:
This Placement selects the clusters where the gitops-addon will be enabled. Create placement-addons.yaml:

YAML

apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement-enable-gitops-addon
  namespace: open-cluster-management
spec:
  clusterSets:
    - global
Apply both files to the Hub cluster:

Bash

oc apply -f cma-gitops-addon.yaml
oc apply -f placement-addons.yaml
RHACM will now automatically install a pull-model-ready OpenShift GitOps instance on every cluster matched by the Placement.

Step 3: Define Applications and Targeting in Git
(This step is unchanged. Ensure your Git repo has this structure and content.)

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
File Contents:

common-apps/devteam1-app/params.json:
JSON

{
  "hub_app_namespace": "devteam1",
  "managed_cluster_deploy_namespace": "devteam1",
  "app_name_suffix": "dt1"
}
common-apps/devteam1-app/deployment.yaml:
YAML

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
    spec:
      containers:
      - name: hello-openshift
        image: openshift/hello-openshift
common-apps/devteam2-app/params.json:
JSON

{
  "hub_app_namespace": "devteam2",
  "managed_cluster_deploy_namespace": "devteam2",
  "app_name_suffix": "dt2"
}
common-apps/devteam2-app/deployment.yaml:
YAML

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
    spec:
      containers:
      - name: hello-world
        image: gcr.io/google-samples/hello-app:1.0
common-apps/appadmin-tool/params.json:
JSON

{
  "hub_app_namespace": "appadmin",
  "managed_cluster_deploy_namespace": "appadmin",
  "app_name_suffix": "adm"
}
common-apps/appadmin-tool/deployment.yaml:
YAML

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
    spec:
      containers:
      - name: nginx
        image: nginx:latest
Commit and push this structure to your Git repository.

Step 4: Create the Master Pull-Model ApplicationSet on the Hub
This ApplicationSet is the engine of our workflow. It reads our Git repository, discovers the applications and their target namespaces, finds the managed clusters from our Placement, and generates the Application CRs on the Hub with the correct annotations for the pull model.

Create master-appset-pull-model.yaml. Remember to replace <YOUR_GIT_REPO_URL>.

YAML

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-namespace-pull-set
  namespace: openshift-gitops
spec:
  generators:
  - matrix:
      generators:
        - clusterDecisionResource: # Uses RHACM Placement
            configMapRef: acm-placement
            name: appset-target-all-managed # Our Placement for AppSet
            requeueAfterSeconds: 180
        - git: # Gets app configs
            repoURL: <YOUR_GIT_REPO_URL> # !!! REPLACE THIS !!!
            revision: main
            files:
            - path: "common-apps/**/params.json"
  template:
    # This Application CR is created on the HUB
    metadata:
      name: '{{path.basename}}-{{values.app_name_suffix}}-{{name}}'
      namespace: '{{values.hub_app_namespace}}' # CRUCIAL: Creates App CR in devteam1, etc. on Hub
      labels:
        team: '{{values.managed_cluster_deploy_namespace}}'
        rhacm-appset-pull-model: "true"
      # Annotations for RHACM Pull Model:
      annotations:
        # Tells Hub Argo CD to NOT reconcile this Application CR directly
        argocd.argoproj.io/skip-reconcile: "true"
        # Identifies the target managed cluster for RHACM
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
        # Explicitly enables pull model processing by RHACM
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
    spec:
      project: team-applications
      source:
        repoURL: <YOUR_GIT_REPO_URL> # !!! REPLACE THIS !!!
        targetRevision: main
        path: '{{path}}'
      # Destination for the PULL MODEL Application CR on the managed cluster
      destination:
        server: [https://kubernetes.default.svc](https://kubernetes.default.svc) # Standard for local cluster reconciliation
        namespace: '{{values.managed_cluster_deploy_namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
Apply it: oc apply -f master-appset-pull-model.yaml -n openshift-gitops

Step 5: Verification
On the Hub Cluster:

Check Addon Status: Verify that the gitops-addon was successfully enabled for your managed clusters.
Bash

# Replace 'cluster1' with your managed cluster's namespace on the hub
oc get managedclusteraddon gitops-addon -n cluster1 -o yaml
Look for a condition with type: Available and status: "True".
Check Generated Application CRs: Confirm that the Application resources were created in the correct team namespaces on the Hub.
Bash

oc get applications -n devteam1
oc get applications -n devteam2
oc get applications -n appadmin
Check ManifestWork: Verify that RHACM has created ManifestWork to transport the Application definitions to the managed clusters.
Bash

# Replace <managed_cluster_hub_namespace> e.g., cluster1
oc get manifestwork -n <managed_cluster_hub_namespace> -l rhacm-appset-pull-model=true
On the Managed Clusters:

Check GitOps Installation: Switch your oc context to a managed cluster. Verify that the gitops-addon successfully installed OpenShift GitOps.
Bash

# Check for the addon controller pod
oc get pods -n open-cluster-management-agent-addon | grep gitops-addon

# Check for the GitOps Operator pods
oc get pods -n openshift-gitops
You should see the argocd-application-controller, argocd-repo-server, and argocd-redis pods running.
Check Deployed Applications: Finally, verify that your sample applications are running in their designated namespaces.
Bash

oc get pods -n devteam1
oc get pods -n devteam2
oc get pods -n appadmin
You should see the hello-app-devteam1-*, hello-app-devteam2-*, and admin-tool-deployment-* pods running successfully.
