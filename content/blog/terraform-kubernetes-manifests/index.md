---
title: Applying kubernetes manifests using the terraform provider
tags: ["Kubernetes", "Terraform", "Helm", "Karpenter"]
date: '2024-04-06'
---

There was a use case where I had to install [Karpenter](https://karpenter.sh/) before installing [ArgoCD](https://argoproj.github.io/cd/).
The EKS cluster was created using [terraform](https://www.terraform.io/), so it made sense to use terraform to manage all karpenter resources as well.
In the past I had used the [kubectl provider](https://registry.terraform.io/providers/gavinbunney/kubectl/latest), but this time I ran into an issue when running terraform apply:

```bash
Error: default failed to create kubernetes rest client for update of resource: Unauthorized
│
│   with kubectl_manifest.karpenter_node_class,
│   on karpenter.tf line 67, in resource "kubectl_manifest" "karpenter_node_class":
│   67: resource "kubectl_manifest" "karpenter_node_class" {
```

There is an open issue in the kubectl provider which talks about [this](https://github.com/gavinbunney/terraform-provider-kubectl/issues/229).
I also tried the newer (and more maintained) [kubectl provider](https://github.com/alekc/terraform-provider-kubectl), but still ran into the same issue.

I think a better alternative would be to use the official kubernetes provider to apply the kubernetes manifests.
This means we have one less community managed module, keeping our supply chain more secure (hopefully)

The [kubernetes_manifest](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/manifest) is the perfect resource for this use case as I am trying to create an EC2 NodeClass and a NodePool.
The Custom Resource Definitions (CRDs) were already installed in the cluster by the helm provider.

#### Implementation details.

First I created a provider.tf and configured the kubernetes provider

```bash
terraform {
  required_providers {
    kubernetes = {
      source = "hashicorp/kubernetes"
      version = ">= 2.27.0"
    }
  }
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    # This requires the awscli to be installed locally where Terraform is executed
    args = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}
```



After that, I created a karpenter.tf file with the following content:

```bash
# Install the helm chart for karpenter
resource "helm_release" "karpenter" {
  namespace           = "karpenter"
  create_namespace    = true
  name                = "karpenter"
  repository          = "oci://public.ecr.aws/karpenter"
  repository_username = data.aws_ecrpublic_authorization_token.token.user_name
  repository_password = data.aws_ecrpublic_authorization_token.token.password
  chart               = "karpenter"
  version             = "0.35.1"
  wait                = false

  values = [
    <<-EOT
    replicas: 1
    settings:
      clusterName: ${module.eks.cluster_name}
      clusterEndpoint: ${module.eks.cluster_endpoint}
      interruptionQueue: ${module.karpenter.queue_name}
    EOT
  ]
}

# Apply manifests for the karpenter node class
resource "kubernetes_manifest" "karpenter_node_class" {
  manifest = yamldecode(
    templatefile("${path.module}/templates/karpenter_node_class.yaml.tftpl", {
      role         = module.eks.eks_managed_node_groups["bottlerocket_custom"].iam_role_name
      cluster_name = module.eks.cluster_name
    })
  )
  depends_on = [
    helm_release.karpenter
  ]
}

# Apply manifests for the karpenter node pool
resource "kubernetes_manifest" "karpenter_node_pool" {
  manifest = yamldecode(
    file("${path.module}/templates/karpenter_node_pool.yaml")
  )
  depends_on = [
    kubernetes_manifest.karpenter_node_class
  ]
}
```

A separate templates folder was created with 2 files

- `karpenter_node_class.yaml.tftpl`

```bash
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: Bottlerocket
  role: ${role}
  userData: |
    [settings.host-containers.admin]
    enabled = false
    [settings.host-containers.control]
    enabled = true
    [settings.kubernetes.node-labels]
    'karpenter.sh/capacity-type' = 'spot'
    [settings.kubernetes.node-taints]
    "node.cilium.io/agent-not-ready" = "true:NoExecute"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${cluster_name}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${cluster_name}
  tags:
    karpenter.sh/discovery: ${cluster_name}

```

- `karpenter_node_pool.yaml`

```bash
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["t", "m"]
        - key: "karpenter.k8s.aws/instance-cpu"
          operator: In
          values: ["2"]
        - key: "karpenter.k8s.aws/instance-hypervisor"
          operator: In
          values: ["nitro"]
        - key: "karpenter.k8s.aws/instance-generation"
          operator: Gt
          values: ["2"]
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s

```

We can verify that these resources were created after executing a terraform apply

```bash
➜  kubectl get nodepool
NAME      NODECLASS
default   default
➜  kubectl get ec2nc
NAME      AGE
default   19h
```
