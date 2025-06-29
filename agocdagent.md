# Supercharge Your GitOps: Managing the argocd-agent with Red Hat Advanced Cluster Management

Red Hat Advanced Cluster Management (ACM) for Kubernetes provides a powerful, single control plane to orchestrate and manage multiple Kubernetes clusters. Built on the foundation of the open-source project Open Cluster Management (OCM), ACM excels at simplifying complex multi-cluster and multicloud deployments.

A key feature of ACM is its extensible add-on framework, which allows you to seamlessly integrate and manage essential tools across your entire fleet. One such powerful integration is the argocd-agent, which can be installed and managed as an ACM add-on, bringing the power of Argo CD's GitOps capabilities under the centralized governance of Red Hat ACM.

## Benefits of Using the ACM argocd-agent Add-On
Integrating the argocd-agent via the ACM add-on framework offers significant advantages for platform engineering and operations teams.

🎯 Centralized Deployment
With your managed clusters (or spokes) registered to the ACM hub, you can deploy the argocd-agent across your entire landscape from a single, centralized point of control. Newly registered spokes can also be configured to automatically receive the argocd-agent, completely eliminating manual, cluster-by-cluster deployments and ensuring consistent tooling.

🚀 Advanced Placement and Rollout
Leverage the sophisticated Placement API within Red Hat ACM to define precise and advanced strategies for deploying the argocd-agent. This allows for controlled rollouts, targeting specific cluster sets based on labels, regions, or other criteria, giving you granular control over your fleet.

🔧 Centralized Lifecycle Management & Maintenance
Manage the complete lifecycle of all argocd-agent instances directly from the ACM hub cluster. This centralized approach simplifies critical operations such as upgrades, rollbacks, and, most importantly, the ability to swiftly revoke access for any agent that is compromised or behaving maliciously.

🩺 Fleet-wide Health Visibility
Gain a unified, real-time view into the health and operational status of every argocd-agent instance across your cluster fleet. ACM's console provides a single pane of glass to monitor connectivity and ensure your GitOps agents are running smoothly everywhere.

🔒 Secure Communication
Security is paramount. The ACM add-on framework can leverage a CustomSigner registration type to automatically sign and manage client certificates for the argocd-agent on managed clusters. This ensures all communication between the agent and the Argo CD server is secured with mTLS. The framework also handles automatic certificate rotation, preventing downtime and security lapses due to expired certificates.

⚙️ Flexible Configuration
Customize the argocd-agent deployment to meet your specific needs using the ACM AddOnTemplate. This powerful feature enables you to apply custom, template-based modifications to the agent's configuration and Kubernetes manifests, ensuring it fits perfectly within your environment's requirements.

## Architecture
The architecture for managing the argocd-agent with ACM is streamlined and declarative, relying on a few key Kubernetes-native resources on the hub cluster. The diagram below shows a simplified overview of this architecture with one managed cluster connected to the ACM hub.

### ClusterManagementAddOn
The central pillar for add-on registration. This hub-level resource formally introduces the argocd-agent to ACM, linking its deployment blueprint (AddOnTemplate) with the targeting logic (Placement) that dictates which managed clusters will host the agent.

### AddOnTemplate
Your agent's deployment manifest. This resource provides the precise instructions—the Kubernetes manifests—for installing and configuring the argocd-agent on each designated managed cluster. It also defines the use of the CustomSigner for automated, secure certificate management.

### Placement
The intelligent scheduler for your add-on. This resource specifies the rules and criteria (e.g., cluster labels, regions) for selecting the managed clusters where the argocd-agent will be deployed, ensuring it reaches the right destinations within your fleet.

## Setup
Ready to get started? If you want to use Red Hat Advanced Cluster Management to bootstrap and manage your argocd-agent environment, follow the comprehensive guide provided in the official documentation to walk through the setup process.
