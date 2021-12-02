# Sample Dotnet Core application

Very simple web application with the least amount of dependencies.


## Pre-requisites

### Build

* [docker](https://docs.docker.com/get-docker/) CLI
* [pack](https://buildpacks.io/docs/tools/pack/#install) CLI
* [kp](https://github.com/vmware-tanzu/kpack-cli/releases) CLI
  * Target Kubernetes cluster 1.18 or better with
    * a container image registry (e.g., Harbor) installed
      * may or may not be on same cluster as Tanzu Build Service
    * Tanzu Build Service installed and appropriately integrated with container image registry
  * ~/.kube/config
    * Target context set to the cluster where Tanzu Build Service is installed

### Deploy

* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [tanzu](https://network.pivotal.io/products/tanzu-application-platform/#/releases/992949/file_groups/5804) CLI (beta 3)
  * Assumes you have access to a cluster with Tanzu Application Platform installed
* cluster no.2 with [kpack-controller](https://carvel.dev/kapp-controller/docs/latest/install/) installed
* cluster no.2 with cert-manager and external-dns installed
* a _registry-credentials_ secret installed in the namespace(s) where application will be deployed


## Clone

```
git clone https://github.com/pacphi/AltPackageRepository
```

You can switch between two branches:

* _main_
  * `git checkout main`
  * Builds application sourcing packages from publicly hosted Nuget repositories
* _jfrog-saas-artifactory_
  * `git checkout jfrog-saas-artifactory`
  * Builds application sourcing packages from private Artifactory repository
    * You may need to talk to your friendly enterprise operator to help you get this set up
    * JFrog makes this fairly straight-forward to [setup](https://jfrog.com/start-free/) a cloud-hosted instance yourself


## Build

### with locally cloned source using pack

First time use of pack, you'll need to run

```
pack config default-builder paketobuildpacks/builder:base
```

If you're on _main_ branch then

```
pack build dotnet-core-sample-nuget --path . --wait
```

If you're on _jfrog-saas-artifactory_ branch then

```
pack build dotnet-core-sample-nuget --path . --env USERNAME="{artifactory-username}" --env PASSWORD="{artifactory-password}" --env ARTIFACTORY_DOMAIN="{artifactory-domain}" --env REPOSITORY_KEY="{artifactory-repository-key}" --wait
```
> Replace all occurrences of `{artifactory}-*` placeholders with appropriate values


### direct from Github using kp and Tanzu Build Service

If you want to build from the _main_ branch then

```
kp image save dotnet-core-sample-nuget --git https://github.com/pacphi/AltPackageRepository --tag {container-image-registry-domain}/{repository-prefix}/dotnet-core-sample --wait
```
> Replace `{}` placeholders with appropriate values

If you want to build from the _jfrog-saas-artifactory_ branch then

```
kp image save dotnet-core-sample-nuget --git https://github.com/pacphi/AltPackageRepository --git-revision jfrog-saas-artifactory --tag {container-image-registry-domain}/{repository-prefix}/dotnet-core-sample --env USERNAME="{artifactory-username}" --env PASSWORD="{artifactory-password}" --env ARTIFACTORY_DOMAIN="{artifactory-domain}" --env REPOSITORY_KEY="{artifactory-repository-key}" --wait
```
> Replace all occurrences of `{artifactory}-*` placeholders with appropriate values

For example

```
kp image save dotnet-core-sample-nuget --git https://github.com/pacphi/AltPackageRepository --git-revision jfrog-saas-artifactory --tag harbor.klu.zoolabs.me/platform/apps/dotnet-core-sample --env USERNAME="<redacted>" --env PASSWORD="<redacted>" --env ARTIFACTORY_DOMAIN="zoolabs.jfrog.io" --env REPOSITORY_KEY="public-nuget" --wait
```


## Deploy

### manually with Docker

If you previously built with `pack`

```
docker run --rm -d -p 8080:8080 dotnet-sample-nuget
```

Visit `http://localhost:8080`

To stop

```
docker ps
docker stop {pid}
```

### manually with Kubernetes manifests

Create a namespace

```
kubectl create namespace staging
```

Create deployment manifest

```
cat > dotnet-core-sample-deployment.yml <<EOF
apiVersion: v1
kind: Secret
type: kubernetes.io/dockerconfigjson
metadata:
  name: registry-credentials
data:
  .dockerconfigjson: eyJhdXRocyI6eyJoYXJib3Iua2x1Lnpvb2xhYnMubWUiOnsidXNlcm5hbWUiOiJhZG1pbiIsInBhc3N3b3JkIjoiamFja0JlUXVpY2skNTAifX19
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: dotnet-core-sample
  name: dotnet-core-sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dotnet-core-sample
  template:
    metadata:
      labels:
        app: dotnet-core-sample
    spec:
      imagePullSecrets:
      - name: registry-credentials
      containers:
      - image: harbor.klu.zoolabs.me/platform/apps/dotnet-sample@sha256:4e2783a175a037f055165321d425cbcc1a84599e27ce0b4ba0749db05c141df7
        name: dotnet-core-sample
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: dotnet-core-sample
spec:
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: "dotnet-core-sample"
EOF
```
> You will need to replace `harbor.klu.zoolabs.me`, the encrypted value next to `.dockerconfigjson:`, and the image tag `dotnet-sample@sha256:4e2783a175a037f055165321d425cbcc1a84599e27ce0b4ba0749db05c141df7` with whatever the output of `kp image save` was.

Deploy it!

```
kubectl apply -f dotnet-core-sample-deployment.yml -n staging
```
> Note the `staging` namespace must already exist

Port forward

```
kubectl port-forward $(kubectl get po -l app=dotnet-core-sample -n staging | tail -n -1 | grep -o '^\S*') 8080:8080 -n staging
```
> Again note the `staging` namespace must already exist


Visit `http://localhost:8080` in a web browser

Destroy it!

```
kubectl delete -f dotnet-core-sample-deployment.yml -n staging
```

### with App CR, a Github manifests repository, and kpack-controller

Setup `ServiceAccount`, `Role` and `RoleBinding`. (Assumes `staging` namespace already exists in target cluster).

```
cat > staging-ns-sa.yml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: staging-ns-sa
  namespace: staging
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: staging-ns-role
  namespace: staging
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: staging-ns-role-binding
  namespace: staging
subjects:
- kind: ServiceAccount
  name: staging-ns-sa
  namespace: staging
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: staging-ns-role
EOF
```

Apply

```
kubectl apply -f staging-ns-sa.yml
```

Then, apply this CR when you want to pull and deploy the latest image available updates from the container registry with no intermediary step

```
cat > dotnet-core-sample-cd-via-imagepull.yml <<EOF
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  name: dotnet-core-sample
  namespace: staging
spec:
  serviceAccountName: staging-ns-sa
  fetch:
  - image
      url: harbor.klu.zoolabs.me/platform/apps/dotnet-sample@sha256:4e2783a175a037f055165321d425cbcc1a84599e27ce0b4ba0749db05c141df7
      secretRef:
        name: registry-credentials
  template:
  - ytt: {}
  deploy:
  - kapp: {}
EOF
```
> You will need to replace `harbor.klu.zoolabs.me`, the encrypted value next to `.dockerconfigjson:`, and the image tag `dotnet-sample@sha256:4e2783a175a037f055165321d425cbcc1a84599e27ce0b4ba0749db05c141df7` with whatever the output of `kp image save` was


Accessing the deployed application is left as an exercise for the consumer.  You could `kubectl port-forward` or if the cluster had [cert-manager](https://cert-manager.io/docs/), an ingress controller (like [contour](https://projectcontour.io/getting-started/) or [traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)) and [external-dns](https://github.com/kubernetes-sigs/external-dns) installed and configured appropriately you may establish a public route to your application.


### with Tanzu Application Platform

A one-liner that combines build with deploy using an in-built (and operator-configurable) supply chain.  Think of it as choreographed, opinionated, yet open for extension "path to production".

```
tanzu apps workload create dotnet-core-sample --git-repo https://github.com/pacphi/AltPackageRepository --git-branch main --type web --wait --tail
```
