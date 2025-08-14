# Argo CD Pull Model for Red Hat Advanced Cluster Management

- Read more about the architecture [here]

[here]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.14/html/gitops/gitops-overview#arch-pull

- Pull model is an obvious choice for anyone looking to offload the work to child clusters while maintaining a central control plane

## Steps

1. Identify your cluster sets - start by deciding which clusters will be having a particular set of applications or operators across all of the similar type of clusters. For example, all lab clusters need to have the cost-management operator to take care of costs and ensure lab clusters are not over-used.

This can be created from the ACM UI the fastest, or also by creating a ManagedClusterSet object like below

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: argocd-child-clusters
spec:
  clusterSelector:
    selectorType: ExclusiveClusterSetLabel
```
If you are using the cli, you need to complete the following [steps] as well, by adding labels to identify the clusters to add to the clusterset

[steps]: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.3/html/clusters/managedclustersets#adding-clusters-to-a-managedclusterset-cli

2. You now create a Placement rule for your cluster set. Essentially how different applications will reference your Argo Application

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: gitops-placement
  namespace: openshift-gitops
spec:
  clusterSets:
    - argocd-child-clusters
```

3. You then create a `ManagedClusterSetBinding` to bind your placement rule to the `ManagedClusterSet`

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: argocd-child-clusterset-binding
  namespace: openshift-gitops
spec:
  clusterSet: argocd-child-clusters
```

4. You then create your `GitopsCluster` resource for every cluster type. This specifies where to search for the argocd server. Notice the argoServer section which mentions the location of the argoServer
```yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
spec:
  argoServer:
    argoNamespace: openshift-gitops
    cluster: local-cluster
  placementRef:
    apiVersion: cluster.open-cluster-management.io/v1beta1
    kind: Placement
    name: gitops-placement
    namespace: openshift-gitops
```
That's all!

4. You now create your ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: helloworld-allclusters-app-set
  namespace: openshift-gitops
spec:
  generators:
  - clusterDecisionResource:
      configMapRef: acm-placement
      labelSelector:
        matchLabels:
          cluster.open-cluster-management.io/placement: gitops-placement
      requeueAfterSeconds: 30
  template:
    metadata:
      annotations:
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
        apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: openshift-gitops
        argocd.argoproj.io/skip-reconcile: "true"
      labels:
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
      name: '{{name}}-helloworld-app'
    spec:
      destination:
        namespace: helloworld
        server: https://kubernetes.default.svc
      project: default
      source:
        path: helloworld
        repoURL: https://github.com/stolostron/application-samples.git
      syncPolicy:
        automated: {}
        syncOptions:
        - CreateNamespace=true
```