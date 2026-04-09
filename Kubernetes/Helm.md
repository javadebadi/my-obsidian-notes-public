Helm is the package management system for Kubernetes. Helm to Kubernetes is similar to pip to Python.

Helm use cases:
- Install and manage 3rd party applications
- Create and store you own Helm charts (ArtifactHub)
- Charts are pre-installed in managed clusters.

## Installation
In MacOs run:
```shell
brew install helm
```
Then check version of the helm:
```shell
helm version --short
```
![[helm-version.png]]

The `helm list` will show list of installed Helm charts:
![[helm-list.png]]
And `helm env` will shot list of environment variables for Helm:
![[helm-env.png]]


## Install and Configure a Chart for the Artifact Hub

Spend some time exploring [ArtifactHub](https://artifacthub.io/).


For example, here I want to install [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack). Based on installation guide on that page

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 83.0.0
```
![[Pasted image 20260406200914.png]]
Basically the pattern for installation of a chart is `helm install <release_name> <repo/chart_name>`.

Now if I run `helm list`:
![[Pasted image 20260406201029.png]]

Helm chart version and the app version are two different things:

|Field|What it is|Used by Helm?|Key point|
|---|---|---|---|
|`version`|Helm chart version|✅ Yes|Controls upgrades & releases|
|`appVersion`|Application (container) version|❌ No|Informational only (for humans)|


And if I run `kubectl get all` I will see many K8 resource are created:
![[Pasted image 20260406201347.png]]

To version control 3rd party helm chart here are some advices:

|Practice|What you do|Problem it solves|
|---|---|---|
|🔒 Pin chart version|`--version x.x.x` or in `Chart.yaml`|Prevents unexpected breaking changes|
|📝 Store `values.yaml` in Git|Keep all overrides in repo|Ensures reproducible configuration|
|🔗 Use dependencies + lock file|`helm dependency update` → commit `Chart.lock`|Locks exact chart versions (like npm lock)|
|📦 Vendor chart (optional)|`helm pull --untar` and commit|Full control, no external dependency risk|
For example for vendoring the chart:
```shell
helm pull bitnami/postgresql --version 12.1.3 --untar
```
Then locally you will have `/charts/postgresql`.

## Custom Chart

To create your own helm chart run:
```shell
helm create sample-chart
```
and it will create a boilerplate helm chart for you which you can edit to have your own chart.
![[Pasted image 20260408223715.png]]

For simplicity, delete the content of `templates` directory and create a new file `configmap.yaml`:
```shell
kubectl create cm app-config --from-literal=username=Admin --dry-run=client -o yaml > sample-chart/templates/configmap.yaml
```
![[Pasted image 20260408224353.png]]

Now to install the helm package run:
```shell
helm install my-helm-app sample-chart
```
![[Pasted image 20260408224643.png]]

As we see in above screenshot the ConfigMap is created after installing the helm package.

Now let's add a secret in `templates` directory and upgrade the helm package.
```shell
kubectl create secret generic app-secret --from-literal=password=password --dry-run=client -o yaml > sample-chart/templates/secrets.yaml
```
![[Pasted image 20260408225626.png]]

To update the package run `helm upgrade <helm-release> <helm-chart>`
```shell
helm upgrade my-helm-app sample-chart
```

![[Pasted image 20260408225727.png]]

![[Pasted image 20260408230042.png]]


to rollback to another version run `helm rollback <helm-realease> <revision-number>`
```shell
helm rollback my-helm-app <revision-number>
```
![[Pasted image 20260408230736.png]]


### Templates directive - Charts.yaml

There is a file `Charts.yaml`:
![[Pasted image 20260408232304.png]]

and we want to use the `version` field of this file in naming of config map using `{{.Chart.Version}}` you should use capital letters based on Go convention:
![[Pasted image 20260408232702.png]]

Now we can see the version is appended to the name of the configmap:
![[Pasted image 20260408232753.png]]

### Helm Conditional Statements
The Helm conditional statements has if/else statements that let you render values dynamically based on whether or not a specific conditions is met.

Update the `configmap.yaml` by adding:
```yaml
{{ if eq .Values.env "dev" }}
  debug: 'true'
{{ else }}
  debug: 'false'
{{ end }}
```

![[Pasted image 20260408235511.png]]

Then update the `values.yaml` file by adding:
```yaml
env: "dev"
```
![[Pasted image 20260408235338.png]]

and run `helm template sample-chart` to see the yaml config:
```shell
helm template sample-chart
```
![[Pasted image 20260408235639.png]]

and now change the `values.yaml` to `env: prod` and re-run:
![[Pasted image 20260408235721.png]]

![[Pasted image 20260408235748.png]]

you can see the `debug` field is different based on the `env` value.

Applying the changes to K8 object by running the `helm upgrade my-app-chart sample-chart` we have:
![[Pasted image 20260409000029.png]]
