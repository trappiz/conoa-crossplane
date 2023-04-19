# crossplane-conoa

Crossplane is an open source Kubernetes extension that transforms your Kubernetes cluster into a universal control plane.

Crossplane lets you manage anything, anywhere, all through standard Kubernetes APIs. Crossplane can even let you order a pizza directly from Kubernetes. If it has an API, Crossplane can connect to it.

With Crossplane, platform teams can create new abstractions and custom APIs with the full power of Kubernetes policies, namespaces, role-based access controls and more. Crossplane brings all your non-Kubernetes resources under one roof.

Custom APIs, created by platform teams, allow security and compliance enforcement across resources or clouds, without exposing any complexity to the developers. A single API call can create multiple resources, in multiple clouds and use Kubernetes as the control plane for everything.


## Crossplane Architecture

What makes Crossplane so special? First, it builds on Kubernetes and capitalizes on the fact that the real power of Kubernetes is its powerful API model and control plane logic (control loops). It also moves away from Infrastructure as Code to Infrastructure as Data. The difference is that IaC means writing code to describe how the provisioning should happen, whereas IaD means writing pure data files (in the case of Kubernetes YAML) and submitting them to the control component (in the case of Kubernetes an operator) to encapsulate and execute the provisioning logic.

![Image](https://lh5.googleusercontent.com/xyOQeU1pf3gEtLf0CcllWqUEyamGwbO6rgk2r-vpmbT3IWhk7rpc7Y_ZUXPh7XEjwmOJFnUwcbjAEagklpKfaYn6lyl7h0EivGUsHEhEEZmyaPzPAR6N6Fk7fER3NI1bJodunJC_eVpGUH6F-1AaV_bGbozkHGyIlYvfZU9X40H1QM3n3GnAUUETjw)

## Crossplane Terms

### Composition
Think of it as a terraform module, HCL code that describes how to take input variables and use them to create resources. Seen with Helm eyes this would translate to the template files in a helm chart


### Composite Resource (XR)
Think of it as tfvars for terraform and values.yaml for helm. In short inputs values to variables

### Composite Resource Claim (XRC)
Same as above but think more team/namespace scoped 

### Composite Resource Definition (XRD)
Think of it as Terraform module that define which variables exist, whether those variables are strings or integers, whether they’re required or optional, etc.


### Package
Packages extend Crossplane, either with support for new kinds of composite resources and claims, or support for new kinds of managed resources. There are two types of Crossplane package; configurations and providers.


### Configuration
A configuration extends Crossplane by installing conceptually related groups of XRDs and Compositions, as well as dependencies like providers or further configurations.

### Provider
Extends Crossplane by installing controllers for new kinds of managed resources. For example the AWS provider installs support for AWS managed resources like RDSInstance and S3Bucket.

A Provider is a distinct type in the Crossplane API. Note that each Provider package has its own configuration type, called a ProviderConfig. Don’t confuse the two, the former installs the provider while the latter specifies configuration that is relevant to all of its managed resources. A Crossplane provider is the same a provider in Terraform
§



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

https://marketplace.upbound.io/
https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws/v0.38.0

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

### Create S3 bucket with dynamic name
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

### Create S3 bucket with static name
```bash
cat <<EOF > s3-bucket.yaml
apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: demo-crossplane-bucket-01
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

kubectl apply -f s3-bucket.yaml
```


### Create a RDS PostgreSQL

```bash
cat <<EOF > rds/rds-instance.yaml
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Subnet
metadata:
  name: subnet-a
spec:
  forProvider:
    cidrBlock: 172.31.1.0/24
    vpcId: vpc-002c8d8307a98f7f5
    availabilityZone: eu-north-1a
    region: eu-north-1
  providerRef:
    name: aws-provider
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Subnet
metadata:
  name: subnet-b
spec:
  forProvider:
    cidrBlock: 172.31.2.0/24
    vpcId: vpc-002c8d8307a98f7f5
    availabilityZone: eu-north-1b
    region: eu-north-1
  providerRef:
    name: aws-provider
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Subnet
metadata:
  name: subnet-c
spec:
  forProvider:
    cidrBlock: 172.31.3.0/24
    vpcId: vpc-002c8d8307a98f7f5
    availabilityZone: eu-north-1c
    region: eu-north-1
  providerRef:
    name: aws-provider
---
apiVersion: database.aws.crossplane.io/v1beta1
kind: DBSubnetGroup
metadata:
  name: postgres-subnet-group
spec:
  forProvider:
    description: sample group
    region: eu-north-1
    subnetIdRefs:
      - name: subnet-a
      - name: subnet-b
      - name: subnet-c
  providerRef:
    name: aws-provider
---
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: rdspostgresql
spec:
  forProvider:
    region: eu-north-1
    dbInstanceClass: db.t3.micro
    masterUsername: masteruser
    allocatedStorage: 20
    engine: postgres
    engineVersion: "13"
    skipFinalSnapshotBeforeDeletion: true
    dbSubnetGroupName: postgres-subnet-group
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: aws-rdspostgresql-conn
EOF


kubectl create -f rds/rds-instance.yaml
```

### Deploy an EKS cluster

Deploying EKS cluster with automation is not a trivial task, there are a lot of components that need to be created along, like VPCs, subnets, IAM Roles, node pools, route tables, gateways…
This is not the experience developers want, but this complexity must go somewhere, there is no magic “Remove complexity” button!

Instead the process starts with building an XRD (Composite Resource Definition) where we can specify:

- schema of the XR (Composite Resource)
- schema of the XRC (Composite Resource Claim)

https://docs.crossplane.io/v1.11/concepts/composition/
https://docs.crossplane.io/v1.11/concepts/terminology/


```bash
# Create composition and definition
kubectl apply -f eks/


# Create EKS cluster! :)
kubectl apply -f eks-cluster-claim.yaml
```


### Get kubeconfig for EKS cluster

```bash
kubectl get secrets --namespace devops-team cluster \
     --output jsonpath="{.data.kubeconfig}" \
     | base64 --decode | tee eks-config.yaml
     export KUBECONFIG=$PWD/eks-config.yaml
     ```
