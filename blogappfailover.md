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

Step 2: Assigning Priority Scores (AddOnPlacementScore)
The Placement needs scores to know which cluster we prefer. We create AddOnPlacementScore resources (one per cluster namespace on the hub) and then patch them with our desired values.

Create the resources:

YAML

# cluster-scores.yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster-score
  namespace: on-prem # <- Namespace for your on-prem cluster
---
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster-score
  namespace: rosa   # <- Namespace for your ROSA cluster
Bash

$ oc apply -f cluster-scores.yaml
Now, set the scores – high for primary (99), low for secondary (1):

Bash

# Primary Cluster (High Score)
$ oc patch addonplacementscore cluster-score --namespace on-prem \
  --subresource=status --type=merge -p \
  '{"status":{"scores":[{"name":"clusterScore","value":99}]}}'

# Secondary Cluster (Low Score)
$ oc patch addonplacementscore cluster-score --namespace rosa \
  --subresource=status --type=merge -p \
  '{"status":{"scores":[{"name":"clusterScore","value":1}]}}'
Now, RHACM knows on-prem is our first choice.

Step 3: Deploying with GitOps (ApplicationSet)
Finally, we use an ApplicationSet to link our Git repository (containing the RocketChat manifests) to the Placement decision.

YAML

# rocketchat-appset.yaml
kind: ApplicationSet
metadata:
  name: rocketchat
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement # Default ACM integration point
        labelSelector:
          matchLabels:
            # --- This is the MAGIC LINK! ---
            cluster.open-cluster-management.io/placement: rocketchat-placement
        requeueAfterSeconds: 30 # Check for placement changes every 30s
  template: # <- Standard Argo CD Application definition
    metadata:
      name: rocketchat-{{name}} # {{name}} comes from the generator (cluster name)
      labels:
        velero.io/exclude-from-backup: "true"
    spec:
      destination:
        namespace: default
        server: "{{server}}" # {{server}} comes from the generator (API URL)
      project: default
      sources:
        - path: rocketchat
          repoURL: [https://github.com/levenhagen/rocketchat-acm.git](https://github.com/levenhagen/rocketchat-acm.git)
          targetRevision: main
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
The Key: The clusterDecisionResource generator watches our rocketchat-placement. It sees which cluster RHACM picked and uses its name ({{name}}) and server URL ({{server}}) to create an Argo CD Application targeted at that specific cluster.

Apply it:

Bash

$ oc apply -f rocketchat-appset.yaml
Watching the Failover in Action 🚀
Initial State: Argo CD will deploy RocketChat to the on-prem cluster because it has the highest score (99).
Failure: Imagine the on-prem cluster goes offline.
Detection: After 60 seconds (our tolerationSeconds), RHACM marks on-prem as unavailable.
Re-evaluation: RHACM re-runs the Placement logic. on-prem is out. The next best (and only) option is rosa (score 1).
Decision Change: The Placement now decides on rosa.
Argo CD Reacts: Within 30 seconds (requeueAfterSeconds), the ApplicationSet generator sees the change. It creates (or updates) an Argo CD Application for rocketchat-rosa.
Deployment: Argo CD deploys RocketChat to the rosa cluster.
Your application is now running on the secondary cluster, all without manual intervention! You can monitor this entire process through the RHACM console's application topology view or the OpenShift GitOps dashboard.

Why This Matters
This approach provides a robust, automated, and GitOps-native way to handle application high availability and disaster recovery. By leveraging RHACM's intelligent placement and Argo CD's declarative deployments, you can:

Increase application resilience across hybrid environments.
Reduce manual effort and human error during outages.
Implement complex HA strategies (like active-passive, active-active, or score-based distribution) with declarative YAML.
Maintain a clear audit trail through Git and Kubernetes resources.
Combining these powerful tools allows you to build sophisticated, self-healing application infrastructures, ready to withstand the challenges of modern distributed systems.





this is not md yet



What this means:



numberOfClusters: 1: Ensures we only run on on-prem or rosa, never both (active-passive).

clusterSets: We focus our search on clusters within the prod-clusters set (you'd need to create this set and add your on-prem and rosa clusters to it).

tolerations: We give RHACM a 60-second grace period before reacting to a cluster becoming unreachable. This avoids failing over due to a minor network hiccup.

prioritizerPolicy: This is the critical part:We use two prioritizers: Our clusterScore (to define primary/secondary) and Steady (to control failback).

Steady has a higher weight (3) than clusterScore (1). This means RHACM will try hard not to move the application once placed. If it fails over to rosa, it will stay on rosa even if on-prem recovers. If you wanted automatic failback, you'd give clusterScore the higher weight.

Apply it:



Bash



$ oc apply -f rocketchat-placement.yaml

Step 2: Assigning Priority Scores (AddOnPlacementScore)

The Placement needs scores to know which cluster we prefer. We create AddOnPlacementScore resources (one per cluster namespace on the hub) and then patch them with our desired values.

Create the resources:



YAML



# cluster-scores.yamlapiVersion: cluster.open-cluster-management.io/v1alpha1kind: AddOnPlacementScoremetadata:

name: cluster-score

namespace: on-prem # <- Namespace for your on-prem cluster---apiVersion: cluster.open-cluster-management.io/v1alpha1kind: AddOnPlacementScoremetadata:

name: cluster-score

namespace: rosa # <- Namespace for your ROSA cluster

Bash



$ oc apply -f cluster-scores.yaml

Now, set the scores – high for primary (99), low for secondary (1):



Bash



# Primary Cluster (High Score)

$ oc patch addonplacementscore cluster-score --namespace on-prem \

--subresource=status --type=merge -p \

'{"status":{"scores":[{"name":"clusterScore","value":99}]}}'# Secondary Cluster (Low Score)

$ oc patch addonplacementscore cluster-score --namespace rosa \

--subresource=status --type=merge -p \

'{"status":{"scores":[{"name":"clusterScore","value":1}]}}'

Now, RHACM knows on-prem is our first choice.

Step 3: Deploying with GitOps (ApplicationSet)

Finally, we use an ApplicationSet to link our Git repository (containing the RocketChat manifests) to the Placement decision.



YAML



# rocketchat-appset.yamlkind: ApplicationSetmetadata:

name: rocketchat

namespace: openshift-gitopsspec:

generators:

- clusterDecisionResource:

configMapRef: acm-placement # Default ACM integration point

labelSelector:

matchLabels:

# --- This is the MAGIC LINK! ---

cluster.open-cluster-management.io/placement: rocketchat-placement

requeueAfterSeconds: 30 # Check for placement changes every 30s

template: # <- Standard Argo CD Application definition

metadata:

name: rocketchat-{{name}} # {{name}} comes from the generator (cluster name)

labels:

velero.io/exclude-from-backup: "true"

spec:

destination:

namespace: default

server: "{{server}}" # {{server}} comes from the generator (API URL)

project: default

sources:

- path: rocketchat

repoURL: [https://github.com/levenhagen/rocketchat-acm.git](https://github.com/levenhagen/rocketchat-acm.git)

targetRevision: main

syncPolicy:

automated:

prune: true

selfHeal: true

syncOptions:

- CreateNamespace=true

- PruneLast=true

The Key: The clusterDecisionResource generator watches our rocketchat-placement. It sees which cluster RHACM picked and uses its name ({{name}}) and server URL ({{server}}) to create an Argo CD Application targeted at that specific cluster.

Apply it:



Bash



$ oc apply -f rocketchat-appset.yaml

Watching the Failover in Action 🚀

Initial State: Argo CD will deploy RocketChat to the on-prem cluster because it has the highest score (99).

Failure: Imagine the on-prem cluster goes offline.

Detection: After 60 seconds (our tolerationSeconds), RHACM marks on-prem as unavailable.

Re-evaluation: RHACM re-runs the Placement logic. on-prem is out. The next best (and only) option is rosa (score 1).

Decision Change: The Placement now decides on rosa.

Argo CD Reacts: Within 30 seconds (requeueAfterSeconds), the ApplicationSet generator sees the change. It creates (or updates) an Argo CD Application for rocketchat-rosa.

Deployment: Argo CD deploys RocketChat to the rosa cluster.

Your application is now running on the secondary cluster, all without manual intervention! You can monitor this entire process through the RHACM console's application topology view or the OpenShift GitOps dashboard.

Why This Matters

This approach provides a robust, automated, and GitOps-native way to handle application high availability and disaster recovery. By leveraging RHACM's intelligent placement and Argo CD's declarative deployments, you can:



Increase application resilience across hybrid environments.

Reduce manual effort and human error during outages.

Implement complex HA strategies (like active-passive, active-active, or score-based distribution) with declarative YAML.

Maintain a clear audit trail through Git and Kubernetes resources.

Combining these powerful tools allows you to build sophisticated, self-healing application infrastructures, ready to withstand the challenges of modern distributed systems.




Show thinking
It seems there might have been an issue with how the Markdown (MD) was displayed or copied in the previous response. Markdown is a lightweight markup language that uses plain-text formatting. The content I provided was written in Markdown.

Here is the blog post again, presented directly as Markdown. If you are viewing this in an environment that renders Markdown, it should appear formatted with headings, lists, code blocks, and bold text. If you are viewing it as plain text, you will see the Markdown syntax (like ##, *, ```, **).

Building Resilient Applications: Automated Failover with OpenShift GitOps and RHACM
In the world of hybrid and multi-cloud deployments, ensuring your applications stay running, even when a cluster faces issues, is critical. Manually moving applications during an outage is slow, error-prone, and stressful. What if we could automate this process, ensuring a smooth transition from a primary site to a secondary one?

This is precisely the challenge we can tackle using Red Hat Advanced Cluster Management for Kubernetes (RHACM) alongside OpenShift GitOps (powered by Argo CD). By combining RHACM's powerful cluster selection capabilities with Argo CD's GitOps-driven deployment engine, we can create sophisticated, automated high-availability (HA) and disaster recovery (DR) strategies.

Let's dive into a common use case: setting up an active-passive failover for a RocketChat application. Our goal is to have it run on our primary on-prem cluster by default, but automatically switch to our secondary rosa (Red Hat OpenShift Service on AWS) cluster if the primary becomes unavailable.

The Building Blocks: How It Works
To achieve this automated failover, we rely on a few key Kubernetes resources, working in concert:

Placement: An RHACM custom resource that acts as the "brain." It evaluates your clusters based on defined rules, policies, and scores to decide where an application should run.
AddOnPlacementScore: Another RHACM custom resource. It lets us assign specific scores to our clusters, giving us fine-grained control over the Placement resource's preferences – essential for defining primary and secondary roles.
ApplicationSet: An Argo CD custom resource. We use its clusterDecisionResource generator, which cleverly listens to the decisions made by an RHACM Placement resource. When the Placement decision changes (e.g., during a failover), the ApplicationSet automatically generates or updates the Argo CD Application to deploy to the newly selected cluster.
Essentially, RHACM decides, and Argo CD acts.

Step 1: Defining the Failover Logic (Placement)
First, we create a Placement resource. This defines how RHACM should select a cluster for our RocketChat workload.

YAML

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
What this means:

numberOfClusters: 1: Ensures we only run on on-prem or rosa, never both (active-passive).
clusterSets: We focus our search on clusters within the prod-clusters set (you'd need to create this set and add your on-prem and rosa clusters to it).
tolerations: We give RHACM a 60-second grace period before reacting to a cluster becoming unreachable. This avoids failing over due to a minor network hiccup.
prioritizerPolicy: This is the critical part:
We use two prioritizers: Our clusterScore (to define primary/secondary) and Steady (to control failback).
Steady has a higher weight (3) than clusterScore (1). This means RHACM will try hard not to move the application once placed. If it fails over to rosa, it will stay on rosa even if on-prem recovers. If you wanted automatic failback, you'd give clusterScore the higher weight.
Apply it:

Bash

$ oc apply -f rocketchat-placement.yaml
Step 2: Assigning Priority Scores (AddOnPlacementScore)
The Placement needs scores to know which cluster we prefer. We create AddOnPlacementScore resources (one per cluster namespace on the hub) and then patch them with our desired values.

Create the resources:

YAML

# cluster-scores.yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster-score
  namespace: on-prem # <- Namespace for your on-prem cluster
---
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster-score
  namespace: rosa   # <- Namespace for your ROSA cluster
Bash

$ oc apply -f cluster-scores.yaml
Now, set the scores – high for primary (99), low for secondary (1):

Bash

# Primary Cluster (High Score)
$ oc patch addonplacementscore cluster-score --namespace on-prem \
  --subresource=status --type=merge -p \
  '{"status":{"scores":[{"name":"clusterScore","value":99}]}}'

# Secondary Cluster (Low Score)
$ oc patch addonplacementscore cluster-score --namespace rosa \
  --subresource=status --type=merge -p \
  '{"status":{"scores":[{"name":"clusterScore","value":1}]}}'
Now, RHACM knows on-prem is our first choice.

Step 3: Deploying with GitOps (ApplicationSet)
Finally, we use an ApplicationSet to link our Git repository (containing the RocketChat manifests) to the Placement decision.

YAML

# rocketchat-appset.yaml
kind: ApplicationSet
metadata:
  name: rocketchat
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement # Default ACM integration point
        labelSelector:
          matchLabels:
            # --- This is the MAGIC LINK! ---
            cluster.open-cluster-management.io/placement: rocketchat-placement
        requeueAfterSeconds: 30 # Check for placement changes every 30s
  template: # <- Standard Argo CD Application definition
    metadata:
      name: rocketchat-{{name}} # {{name}} comes from the generator (cluster name)
      labels:
        velero.io/exclude-from-backup: "true"
    spec:
      destination:
        namespace: default
        server: "{{server}}" # {{server}} comes from the generator (API URL)
      project: default
      sources:
        - path: rocketchat
          repoURL: [https://github.com/levenhagen/rocketchat-acm.git](https://github.com/levenhagen/rocketchat-acm.git)
          targetRevision: main
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
The Key: The clusterDecisionResource generator watches our rocketchat-placement. It sees which cluster RHACM picked and uses its name ({{name}}) and server URL ({{server}}) to create an Argo CD Application targeted at that specific cluster.

Apply it:

Bash

$ oc apply -f rocketchat-appset.yaml
Watching the Failover in Action 🚀
Initial State: Argo CD will deploy RocketChat to the on-prem cluster because it has the highest score (99).
Failure: Imagine the on-prem cluster goes offline.
Detection: After 60 seconds (our tolerationSeconds), RHACM marks on-prem as unavailable.
Re-evaluation: RHACM re-runs the Placement logic. on-prem is out. The next best (and only) option is rosa (score 1).
Decision Change: The Placement now decides on rosa.
Argo CD Reacts: Within 30 seconds (requeueAfterSeconds), the ApplicationSet generator sees the change. It creates (or updates) an Argo CD Application for rocketchat-rosa.
Deployment: Argo CD deploys RocketChat to the rosa cluster.
Your application is now running on the secondary cluster, all without manual intervention! You can monitor this entire process through the RHACM console's application topology view or the OpenShift GitOps dashboard.

Why This Matters
This approach provides a robust, automated, and GitOps-native way to handle application high availability and disaster recovery. By leveraging RHACM's intelligent placement and Argo CD's declarative deployments, you can:

Increase application resilience across hybrid environments.
Reduce manual effort and human error during outages.
Implement complex HA strategies (like active-passive, active-active, or score-based distribution) with declarative YAML.
Maintain a clear audit trail through Git and Kubernetes resources.
Combining these powerful tools allows you to build sophisticated, self-healing application infrastructures, ready to withstand the challenges of modern distributed systems.






can you send the second part of blog as markdown?



Show thinking
It seems there might still be some confusion or display issues. The previous response contained the entire blog post formatted in Markdown.

However, if you'd like a specific portion, here is the text starting from "Step 2: Assigning Priority Scores" through to the end, presented again in Markdown format:

Step 2: Assigning Priority Scores (AddOnPlacementScore)
The Placement needs scores to know which cluster we prefer. We create AddOnPlacementScore resources (one per cluster namespace on the hub) and then patch them with our desired values.

Create the resources:

YAML

# cluster-scores.yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster-score
  namespace: on-prem # <- Namespace for your on-prem cluster
---
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster-score
  namespace: rosa   # <- Namespace for your ROSA cluster
Bash

$ oc apply -f cluster-scores.yaml
Now, set the scores – high for primary (99), low for secondary (1):

Bash

# Primary Cluster (High Score)
$ oc patch addonplacementscore cluster-score --namespace on-prem \
  --subresource=status --type=merge -p \
  '{"status":{"scores":[{"name":"clusterScore","value":99}]}}'

# Secondary Cluster (Low Score)
$ oc patch addonplacementscore cluster-score --namespace rosa \
  --subresource=status --type=merge -p \
  '{"status":{"scores":[{"name":"clusterScore","value":1}]}}'
Now, RHACM knows on-prem is our first choice.

Step 3: Deploying with GitOps (ApplicationSet)
Finally, we use an ApplicationSet to link our Git repository (containing the RocketChat manifests) to the Placement decision.

YAML

# rocketchat-appset.yaml
kind: ApplicationSet
metadata:
  name: rocketchat
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement # Default ACM integration point
        labelSelector:
          matchLabels:
            # --- This is the MAGIC LINK! ---
            cluster.open-cluster-management.io/placement: rocketchat-placement
        requeueAfterSeconds: 30 # Check for placement changes every 30s
  template: # <- Standard Argo CD Application definition
    metadata:
      name: rocketchat-{{name}} # {{name}} comes from the generator (cluster name)
      labels:
        velero.io/exclude-from-backup: "true"
    spec:
      destination:
        namespace: default
        server: "{{server}}" # {{server}} comes from the generator (API URL)
      project: default
      sources:
        - path: rocketchat
          repoURL: https://github.com/levenhagen/rocketchat-acm.git
          targetRevision: main
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
The Key: The clusterDecisionResource generator watches our rocketchat-placement. It sees which cluster RHACM picked and uses its name ({{name}}) and server URL ({{server}}) to create an Argo CD Application targeted at that specific cluster.

Apply it:

Bash

$ oc apply -f rocketchat-appset.yaml
Watching the Failover in Action 🚀
Initial State: Argo CD will deploy RocketChat to the on-prem cluster because it has the highest score (99).
Failure: Imagine the on-prem cluster goes offline.
Detection: After 60 seconds (our tolerationSeconds), RHACM marks on-prem as unavailable.
Re-evaluation: RHACM re-runs the Placement logic. on-prem is out. The next best (and only) option is rosa (score 1).
Decision Change: The Placement now decides on rosa.
Argo CD Reacts: Within 30 seconds (requeueAfterSeconds), the ApplicationSet generator sees the change. It creates (or updates) an Argo CD Application for rocketchat-rosa.
Deployment: Argo CD deploys RocketChat to the rosa cluster.
Your application is now running on the secondary cluster, all without manual intervention! You can monitor this entire process through the RHACM console's application topology view or the OpenShift GitOps dashboard.

Why This Matters
This approach provides a robust, automated, and GitOps-native way to handle application high availability and disaster recovery. By leveraging RHACM's intelligent placement and Argo CD's declarative deployments, you can:

Increase application resilience across hybrid environments.
Reduce manual effort and human error during outages.
Implement complex HA strategies (like active-passive, active-active, or score-based distribution) with declarative YAML.
Maintain a clear audit trail through Git and Kubernetes resources.
Combining these powerful tools allows you to build sophisticated, self-healing application infrastructures, ready to withstand the challenges of modern distributed systems.








