# Choosing Your GitOps Delivery Model with Red Hat Advanced Cluster Management (RHACM) 🚀

Red Hat Advanced Cluster Management for Kubernetes (RHACM) offers powerful and flexible ways to manage your application deployments across a fleet of Kubernetes clusters. A key aspect of this is the **ApplicationSet**, which allows you to automate deploying applications to multiple clusters. However, deciding on the right architectural model for your ApplicationSet deployments is crucial for efficiency, security, and scalability. This guide will walk you through the three primary models for ApplicationSet delivery with RHACM, helping you choose the best fit for your organization's needs.

***

## Central ApplicationSet Push-Model: Default ApplicationSet on the Hub

In this model, the RHACM Hub cluster takes an active role in pushing application configurations to your managed (spoke) clusters. The ApplicationSet controller resides on the Hub and directly connects to the API server of each spoke cluster to deploy and manage applications.

### Use it when:
- Your **Hub can connect to Spoke** (ManagedCluster).
- **No GitOps agent** should be installed on the Spoke.
- **Only the Hub can connect to Git**, centralizing credentials and enhancing security.
- It works for **Non-OpenShift-Cluster** integration.

This approach offers a centralized and straightforward management experience, especially for smaller environments where direct connectivity isn't a challenge.

***

## Central ApplicationSet Pull-Model

The pull model inverts the control flow. Instead of the Hub pushing configurations, an agent on the spoke cluster **pulls** the desired application manifests from the Hub. The ApplicationSet on the Hub generates the application definitions, which are then made available for the spoke clusters to retrieve and apply.

### Use it when:
- The **Hub cannot or should NOT connect to the Spoke**. This is common in environments with strict network segmentation.
- You're managing a **large fleet** of clusters, as it aligns better with RHACM's scalable architecture.
- You have **edge scenarios** with intermittent or restricted connectivity.

The pull model is the recommended approach for large-scale and distributed environments, offering enhanced security and scalability.

***

## Decentral Pull-Model (using RHACM-Policies)

This advanced model leverages RHACM's powerful **policy engine** to distribute and enforce the presence of a decentralized GitOps agent, typically Argo CD, on each managed cluster. Here, the focus shifts from the ApplicationSet UI to policy-driven governance.

### Use it when:
- There's **no need for ApplicationSet UI support**, but you want to use RHACM-Policies for GitOps.
- **Argo CD shall be used decentrally** on every Managed-Cluster, providing high autonomy.
- You want to benefit from **Policy-templating** to customize configurations at scale.

This decentralized approach offers the ultimate flexibility and GitOps-native integration, perfect for organizations that have fully embraced a policy-driven, decentralized operational model.
