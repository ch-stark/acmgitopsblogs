The Goal: A Multi-Tenant Hub for Team-Managed Deployments
The primary goal of this tutorial is to construct a true multi-tenant GitOps Hub using RHACM. In this model, different teams (devteam1, devteam2, appadmin) are empowered to manage their own application deployments through their own ApplicationSet resources, which live in their dedicated namespaces on the Hub cluster.

This architecture provides superior isolation and autonomy compared to a single, central ApplicationSet.

Here is the workflow we will build:

Team-Scoped ApplicationSets: Each team will have an ApplicationSet defined in their own namespace on the Hub (e.g., devteam1/appset-devteam1). This ApplicationSet is responsible only for that team's applications.
Automated Application Generation: The team's ApplicationSet will automatically generate Application resources for each managed cluster. Crucially, these Application resources will be created in the same namespace as the ApplicationSet (e.g., devteam1/app-for-cluster-1).
Pull-Model Propagation: These Application resources on the Hub will be annotated for the RHACM pull model. RHACM will then propagate their definitions to the managed clusters via ManifestWork.
Managed Cluster Execution: A single, multi-tenant OpenShift GitOps instance (installed by the gitops-addon) on each managed cluster will reconcile all the incoming Application definitions, deploying each team's workloads into the correct corresponding namespaces (devteam1, devteam2, appadmin).
This tutorial will guide you through setting up this entire advanced, multi-tenant workflow from start to finish.

Prerequisites
Red Hat Advanced Cluster Management (RHACM) Hub Cluster (v2.8+ recommended for this feature).
OpenShift GitOps operator installed on the Hub Cluster (v2.8+ recommended).
One or more Managed Clusters imported into RHACM.
A Git repository accessible by Argo CD.
oc (OpenShift CLI) configured to access your Hub Cluster.
Cluster-admin privileges on the Hub Cluster.
Step 1: Configure the Hub Cluster for Multi-Tenant ApplicationSets
This is the most critical setup step. We must configure the Hub's Argo CD to allow both Application and ApplicationSet resources to be managed in team-specific namespaces.

Enable "Application" and "ApplicationSet" in Any Namespace:
Patch the argocd-cmd-params-cm ConfigMap to tell the Argo CD controllers which namespaces to watch. The applicationset.namespaces key enables the feature for ApplicationSets.

Bash

oc patch configmap argocd-cmd-params-cm -n openshift-gitops --type merge -p \
'{"data":{
  "application.namespaces":"openshift-gitops,devteam1,devteam2,appadmin",
  "applicationset.namespaces":"openshift-gitops,devteam1,devteam2,appadmin"
}}'
Restart Argo CD Components:
Apply the changes by restarting the controllers.

Bash

oc rollout restart deployment/openshift-gitops-server -n openshift-gitops
oc rollout restart statefulset/openshift-gitops-application-controller -n openshift-gitops
oc rollout restart deployment/openshift-gitops-applicationset-controller -n openshift-gitops
Create Team Namespaces on the Hub:
These namespaces will contain the ApplicationSet and generated Application resources.

Bash

oc create ns devteam1
oc create ns devteam2
oc create ns appadmin
Create the Shared AppProject:
The AppProject must list the team namespaces in sourceNamespaces. This acts as a security gate, permitting Application resources created in these namespaces to be associated with this project.

Create appproject-teams.yaml:

YAML

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: shared-team-projects
  namespace: openshift-gitops
spec:
  description: Shared project for all team-managed applications
  sourceRepos:
  - '*' # Or restrict to your specific Git repository URL
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  # This allows Application CRs from these namespaces to use this project
  sourceNamespaces:
  - devteam1
  - devteam2
  - appadmin
Apply it: oc apply -f appproject-teams.yaml

Step 2: Prepare Managed Clusters with the gitops-addon
We will use the gitops-addon to automatically install a single, shared OpenShift GitOps instance on each managed cluster. This single instance is capable of managing deployments into multiple different namespaces.

Create the ClusterManagementAddOn Resource:
This makes the gitops-addon available for installation. Create cma-gitops-addon.yaml:

YAML

apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: gitops-addon
spec:
  installStrategy:
    type: Placements
    placements:
    - name: placement-enable-gitops-addon
      namespace: open-cluster-management
Create the Placement for the Addon:
This Placement selects all managed clusters to install the addon on. Create placement-addons.yaml:

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
Step 3: Define Team-Specific ApplicationSets and Apps in Git
Your Git repository will now contain the definitions for each team's ApplicationSet as well as their application manifests.

Recommended Git Structure:

├── appsets/
│   ├── devteam1-appset.yaml
│   ├── devteam2-appset.yaml
│   └── appadmin-appset.yaml
│
└── apps/
    ├── devteam1-app/
    │   └── deployment.yaml
    ├── devteam2-app/
    │   └── deployment.yaml
    └── appadmin-tool/
        └── deployment.yaml
File Contents:

appsets/devteam1-appset.yaml:
Note that metadata.namespace is set to devteam1. The generated Application will also be in this namespace.

YAML

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: devteam1-appset
  namespace: devteam1 # <-- This AppSet lives in the team's namespace
spec:
  generators:
  - clusterDecisionResource:
      configMapRef: acm-placement
      name: appset-target-all-managed # A shared Placement defined on the Hub
      requeueAfterSeconds: 180
  template:
    metadata:
      name: 'devteam1-app-{{name}}'
      namespace: devteam1 # <-- Generated app is also in the team's namespace
      annotations:
        argocd.argoproj.io/skip-reconcile: "true"
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
    spec:
      project: shared-team-projects
      source:
        repoURL: <YOUR_GIT_REPO_URL>
        targetRevision: main
        path: apps/devteam1-app
      destination:
        server: https://kubernetes.default.svc
        namespace: devteam1
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true]
Create similar devteam2-appset.yaml and appadmin-appset.yaml files, changing the metadata.name, metadata.namespace, and template.metadata.namespace, template.spec.source.path, and template.spec.destination.namespace fields accordingly for devteam2 and appadmin.

The deployment.yaml files inside the apps/ directory can be the same simple examples from the previous tutorial.

Step 4: Bootstrap the Team ApplicationSets with an "App of AppSets"
To get the team ApplicationSets from Git onto the Hub, we use a single bootstrap application, often called an "App of AppSets".

Create a Shared Placement for the ApplicationSet Generator:
All the team ApplicationSets can share a single Placement resource to discover clusters. This Placement must exist on the Hub before the ApplicationSets are synced.

Create placement-appset-generator-target.yaml:

YAML

apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: appset-target-all-managed
  namespace: openshift-gitops # Place in a central, known location
spec:
  clusterSets:
    - global
Apply it: oc apply -f placement-appset-generator-target.yaml

Create the Bootstrap Application:
This Application lives in openshift-gitops and its only job is to sync the contents of the appsets/ directory from your Git repository.

Create bootstrap-app-of-appsets.yaml:

YAML

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-appsets
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: <YOUR_GIT_REPO_URL> # !!! REPLACE THIS !!!
    targetRevision: main
    path: appsets # This points to the directory containing all your AppSet YAMLs
  destination:
    server: https://kubernetes.default.svc # Deploys to the Hub itself
    namespace: default # The namespace here doesn't matter, as the AppSets have their own target namespaces
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
Apply it: oc apply -f bootstrap-app-of-appsets.yaml

Step 5: Verification
On the Hub Cluster:

Check Bootstrap App: In the openshift-gitops Argo CD UI, ensure the app-of-appsets application is Synced and Healthy.
Check ApplicationSets: Verify that the team ApplicationSets have been created in their respective namespaces.
Bash

oc get applicationset -n devteam1
oc get applicationset -n devteam2
oc get applicationset -n appadmin
Check Generated Applications: Verify that each ApplicationSet has generated Application CRs.
Bash

oc get application -n devteam1
oc get application -n devteam2
Check ManifestWork: Confirm that ManifestWork has been created to transport the Application definitions.
Bash

# Replace <managed_cluster_hub_namespace> e.g., cluster1
oc get manifestwork -n <managed_cluster_hub_namespace>
On the Managed Clusters:

Check GitOps Installation: Switch oc context to a managed cluster. Verify the single, shared GitOps instance is running.
Bash

oc get pods -n openshift-gitops
Check Deployed Applications: Verify that the workloads from all teams are running in their correct namespaces.
Bash

oc get pods --all-namespaces | grep -E "devteam1|devteam2|appadmin"
You should see pods from all three applications running in their respective namespaces on each managed cluster, all reconciled by the single GitOps instance.
