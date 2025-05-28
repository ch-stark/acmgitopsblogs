# Building Resilient Applications: Automated Failover with OpenShift GitOps and RHACM

In the world of hybrid and multi-cloud deployments, ensuring your applications stay running, even when a cluster faces issues, is critical. Manually moving applications during an outage is slow, error-prone, and stressful. What if we could automate this process, ensuring a smooth transition from a primary site to a secondary one?

This is precisely the challenge we can tackle using **Red Hat Advanced Cluster Management for Kubernetes (RHACM)** alongside **OpenShift GitOps** (powered by Argo CD). By combining RHACM's powerful cluster selection capabilities with Argo CD's GitOps-driven deployment engine, we can create sophisticated, automated high-availability (HA) and disaster recovery (DR) strategies.

Let's dive into a common use case: setting up an **active-passive failover** for a RocketChat application. Our goal is to have it run on our primary `on-prem` cluster by default, but automatically switch to our secondary `rosa` (Red Hat OpenShift Service on AWS) cluster if the primary becomes unavailable.

---

### The Building Blocks: How It Works

To achieve this automated failover, we rely on a few key Kubernetes resources, working in concert:

1.  **`Placement`**: An RHACM custom resource that acts as the "brain." It evaluates your clusters based on defined rules, policies, and scores to decide *where* an application *should* run.
2.  **`AddOnPlacementScore`**: Another RHACM custom resource. It lets us assign specific scores to our clusters, giving us fine-grained control over the `Placement` resource's preferences – essential for defining primary and secondary roles.
3.  **`ApplicationSet`**: An Argo CD custom resource. We use its `clusterDecisionResource` generator, which cleverly *listens* to the decisions made by an RHACM `Placement` resource. When the `Placement` decision changes (e.g., during a failover), the `ApplicationSet` automatically generates or updates the Argo CD `Application` to deploy to the *newly selected* cluster.

Essentially, **RHACM decides**, and **Argo CD acts**.

---

### Step 1: Defining the Failover Logic (`Placement`)

First, we create a `Placement` resource. This defines *how* RHACM should select a cluster for our RocketChat workload.

```yaml
# rocketchat-placement.yaml
kind: Placement
metadata:
  name: rocketchat-placement
  namespace: openshift-gitops # Often placed here for Argo CD integration
spec:
  # --- Core Failover Settings ---
  numberOfClusters: 1 # <- IMPORTANT: Only deploy to ONE cluster at a time.
  clusterSets:
    - prod-clusters   # <- Target clusters within this group.
  tolerations:      # <- Prevents 'flapping' during brief issues.
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
      tolerationSeconds: 60
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
      tolerationSeconds: 60

  # --- Decision Logic ---
  prioritizerPolicy:
    mode: Exact
    configurations:
      # Prioritizer 1: Custom Score (Our Preference)
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: cluster-score # <- Links to our custom scores
            scoreName: clusterScore
        weight: 1                      # <- Lower weight
      # Prioritizer 2: Stability
      - scoreCoordinate:
          type: BuiltIn
          builtIn: Steady              # <- Prefers to NOT move workloads
        weight: 3                      # <- Higher weight = more 'sticky'
