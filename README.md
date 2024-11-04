# Crossplane & GitOps in Action

## Prerequisites
* kubectl
* kind
* argocd
* kustomize
* up

## Setup

### Kubernetes Cluster (Control Plane Cluster)

Create a local Kubernetes cluster using [kind](https://kind.sigs.k8s.io/) to run Crosplane and manage our infrastructure on GCP.

```bash
kind create cluster --wait 5m
```

### ArgoCD

You need to apply the kustomization under `gitops/argocd` directory:

```bash
kubectl apply -k ./gitops/argocd

# Check the argocd namespace
kubectl get pods -n argocd

# Wait until argo is ready
kubectl wait pod --all --for=condition=Ready --namespace=argocd --timeout=600s
```

To access argocd, you can use the CLI tool or the Web UI:

```bash
# First, port forward to the argocd server
kubectl port-forward svc/argocd-server 8080:443 -n argocd

# Get the initial admin password
ARGO_PASS=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d)

# Connect using CLI
# If you want to connect to the UI, you can use https://localhost:8080
argocd login localhost:8080 --username admin --password ${ARGO_PASS} --insecure
```

### Argo Applications

We are ready to use GitOps. Let's start with Crossplane deploymenet:

First, create the app of the apps:

```bash
cat <<EOF > ./gitops/crossplane/crossplane.yaml
##########################################################
## The ArgoCD App of Apps for all Crossplane components ##
##########################################################
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-bootstrap
  namespace: argocd
  labels:
    crossplane.jonashackt.io: bootstrap
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://github.com/developer-guy/crossplane-gitops-in-action-devfest-2024.git
    targetRevision: HEAD
    path: gitops/crossplane/bootstrap
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true    
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 5s 
        factor: 2 
        maxDuration: 1m
EOF
```

Then, create the Crossplane installation:

```bash
cat <<EOF > ./gitops/crossplane/bootstrap/crossplane-install.yaml
# The ArgoCD Application for crossplane core components themselves
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    chart: crossplane
    repoURL: https://charts.crossplane.io/stable
    targetRevision: 1.17.2 # this was the latest version of the Helm Chart at the we were working on this.
    helm:
      releaseName: crossplane
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true    
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 5s 
        factor: 2 
        maxDuration: 1m
EOF
```

We need to push our changes to our repo. Then, when we create the `crossplane-bootstrap` app, ArgoCD will take care of deploying Crossplane Helm Chart for us.

```bash
# Push changes
git add .
git commit -am "Deploy crossplane"
git push

# Create the crossplane-bootstrap app
kubectl apply -f ./gitops/crossplane/crossplane.yaml

# You can manually sync argocd, and check the app status
argocd app sync argocd/crossplane-bootstrap

# List the apps
argocd app list

# Now, check the pods in the crossplane-system namespace
kubectl get pods -n crossplane-system
```

## Manage GCP

First, add GCP Provider application, and it's configuration:

```bash
cat <<EOF > ./gitops/crossplane/bootstrap/provider.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: provider-gcp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: https://github.com/developer-guy/crossplane-gitops-in-action-devfest-2024.git
    targetRevision: HEAD
    path: gitops/crossplane/provider-gcp
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true    
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 5s 
        factor: 2 
        maxDuration: 1m
EOF
```

Create the provider configuration:

```bash
cat <<EOF > ./gitops/crossplane/provider-gcp/storage-provider.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v1.8.3
EOF
```

We need to push those changes, then create the `provider-gcp` application:

```bash
# Push changes
git add .
git commit -am "Deploy GCP provider for storage"
git push

# Create the crossplane-bootstrap app
kubectl apply -f ./gitops/crossplane/bootstrap/provider.yaml
```

### Resources

To be able to create resources on GCP side, you need to create a Service Account and put it's key to a Kubernetes Secret:

```bash
# Step 1: Set up variables for project ID and service account name
PROJECT_ID=$(gcloud config get-value project)  # Replace with your actual project ID
SERVICE_ACCOUNT_NAME="crossplane-gcp-sa"
SERVICE_ACCOUNT_KEY="gcp-crossplane-key.json"
SECRET_NAME="gcp-credentials"
SECRET_KEY="creds.json"

# Step 2: Create the service account
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
    --display-name="Crossplane GCP Service Account"

# Step 3: Assign the roles to the service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/cloudsql.admin"
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.admin"
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/servicenetworking.networksAdmin"

# Step 4: Generate a JSON key for the service account
gcloud iam service-accounts keys create $PWD/$SERVICE_ACCOUNT_KEY \
    --iam-account="$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com"

# Create the secret
kubectl create secret generic $SECRET_NAME \
  -n crossplane-system --from-file=$SECRET_KEY=$PWD/$SERVICE_ACCOUNT_KEY
```

Create the provider config with this information:

```bash
cat <<EOF > ./gitops/crossplane/provider-gcp/providerconfig.yaml
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  credentials:
    secretRef:
      key: $SECRET_KEY
      name: $SECRET_NAME
      namespace: crossplane-system
    source: Secret
  projectID: $PROJECT_ID
EOF
```

Push the changes to take effect:

```bash
# Push changes
git add .
git commit -am "Create GCP provider configuration"
git push
```

## GCP Managed Resources

Add managed resources application to argocd:

```bash
cat <<EOF > ./gitops/crossplane/bootstrap/managed-resources.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: managed-resources
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: https://github.com/developer-guy/crossplane-gitops-in-action-devfest-2024.git
    targetRevision: HEAD
    path: gitops/crossplane/managed-resources
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 5s 
        factor: 2 
        maxDuration: 1m
EOF

# Random ID for the bucket name:
BUCKET_ID=$(uuidgen | cut -c -8 | tr '[:upper:]' '[:lower:]')

cat <<EOF > ./gitops/crossplane/managed-resources/bucket.yaml
apiVersion: storage.gcp.upbound.io/v1beta2
kind: Bucket
metadata:
  annotations:
    meta.upbound.io/example-id: storage/v1beta1/bucketobject
  labels:
    testing.upbound.io/example-name: crossplane-example-bucket-$BUCKET_ID
  name: crossplane-example-bucket-$BUCKET_ID
spec:
  forProvider:
    location: EU
    storageClass: MULTI_REGIONAL
EOF
```

Push the code and apply the `managed-resources` application, and check the bucket being created!

```bash
# Push changes
git add .
git commit -am "Create a bucket"
git push

# Create the crossplane-bootstrap app
kubectl apply -f ./gitops/crossplane/bootstrap/managed-resources.yaml
```


## Composition 

> TODO!
