# crossplane-conoa

Crossplane is an open source Kubernetes extension that transforms your Kubernetes cluster into a universal control plane.

Crossplane lets you manage anything, anywhere, all through standard Kubernetes APIs. Crossplane can even let you order a pizza directly from Kubernetes. If it has an API, Crossplane can connect to it.

With Crossplane, platform teams can create new abstractions and custom APIs with the full power of Kubernetes policies, namespaces, role-based access controls and more. Crossplane brings all your non-Kubernetes resources under one roof.

Custom APIs, created by platform teams, allow security and compliance enforcement across resources or clouds, without exposing any complexity to the developers. A single API call can create multiple resources, in multiple clouds and use Kubernetes as the control plane for everything.


## Crossplane Architecture

What makes Crossplane so special? First, it builds on Kubernetes and capitalizes on the fact that the real power of Kubernetes is its powerful API model and control plane logic (control loops). It also moves away from Infrastructure as Code to Infrastructure as Data. The difference is that IaC means writing code to describe how the provisioning should happen, whereas IaD means writing pure data files (in the case of Kubernetes YAML) and submitting them to the control component (in the case of Kubernetes an operator) to encapsulate and execute the provisioning logic.



## Create minikube base cluster

minikube start --cpus 4 --memory 8192


### create namespaces
kubectl create ns crossplane-system
kubectl create ns devops-team

# Installation

Prepare aws credentils to be used by crossplane

#### Configure AWS secret
```bash
cat << EOF > aws-credentials.txt
[default]
aws_access_key_id = <YOUR aws_access_key_id>
aws_secret_access_key = <YOUR aws_secret_access_key>
EOF

kubectl create secret generic aws-secret \
  -n crossplane-system --from-file=creds=./aws-credentials.txt
```

#### Add crossplane helm repo
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable && helm repo update


### Dry-run helm install
```bash
helm install crossplane \
crossplane-stable/crossplane \
--dry-run --debug \
--namespace crossplane-system \
--create-namespace

### Do the actual deployment using helm
```bash
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace \
--wait
```

### View pods
```bash
kubectl get pods -n crossplane-system
```

### Get crossplane apis 
```bash
kubectl api-resources | grep crossplane
```

### Install AWS provider
```bash
cat <<EOF > provider-aws.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.38.0
EOF

kubectl apply -f provider-aws.yaml
```

### Verify provider installed (~1-2min to get healthy)
```bash
kubectl get providers
```

### Get more in depth info about provider
```bash
kubectl describe providers provider-aws
```

### List new crds
```bash
kubectl get crds
```

### Create providerconfig
```bash
cat <<EOF > aws-providerconfig.yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
kubectl apply -f aws-providerconfig.yaml
```

### Create S3 bucket as an example
```bash
cat <<EOF | kubectl create -f -
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  generateName: crossplane-bucket-
  labels:
    docs.crossplane.io/example: provider-aws
spec:
  deletionPolicy: Delete
  forProvider:
    acl: private
    locationConstraint: eu-north-1
  providerConfigRef:
    name: default
EOF
```

### Deploy an EKS cluster

Deploy EKS Cluster

Deploying EKS cluster with automation is not a trivial task, there are a lot of components that need to be created along, like VPCs, subnets, IAM Roles, node pools, route tables, gateways…
This is not the experience developers want, but this complexity must go somewhere, there is no magic “Remove complexity” button!

Instead the process starts with building an XRD (Composite Resource Definition) where we can specify:

- schema of the XR (Composite Resource)
- schema of the XRC (Composite Resource Claim)


```bash
kubectl apply -f eks/composition-eks.yaml 
kubectl apply -f eks-cluster-claim.yaml
```


### Get kubeconfig for EKS cluster

```bash
kubectl get secrets --namespace devops-team cluster \
     --output jsonpath="{.data.kubeconfig}" \
     | base64 --decode | tee eks-config.yaml
     export KUBECONFIG=$PWD/eks-config.yaml
     ```
