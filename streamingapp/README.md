# StreamingApp Helm Chart

This directory contains the Helm chart for deploying StreamingApp to Kubernetes. The chart deploys the application services, MongoDB, configuration, secrets, services, and ingress resources.

## Prerequisites

* Kubernetes cluster with `kubectl` configured
* Helm 3+
* Docker images available in a registry accessible by the cluster
* An AWS S3 bucket containing the video objects used by StreamingApp
* AWS credentials supplied through a Kubernetes Secret or an attached workload identity

## Chart layout

```text
streamingapp/
├── Chart.yaml
├── values.yaml
├── values-k3d.yaml              # local environment overlay; do not commit secrets
├── values-secrets.example.yaml  # safe template
├── values-secrets.yaml          # local secrets; do not commit
└── templates/
    ├── app.yaml
    ├── configmap.yaml
    ├── ingress.yaml
    ├── mongodb.yaml
    ├── secret.yaml
    └── serviceaccount.yaml
```

## Configure the chart

Start with the version-controlled defaults:

```bash
cd streamingapp
cp values-secrets.example.yaml values-secrets.yaml
```

Edit `values-secrets.yaml` and provide the runtime values required by the application:

```yaml
secrets:
  jwtSecret: "replace-with-a-long-random-value"
  awsAccessKeyId: "replace-with-your-access-key"
  awsSecretAccessKey: "replace-with-your-secret-key"
```

For environments using an EC2 instance role, IRSA, or another workload identity, leave the AWS key fields empty and configure the identity outside the chart.

Set the S3 and application values in an environment overlay. The important values are:

```yaml
config:
  awsRegion: "ap-south-1"
  awsS3Bucket: "your-streamingapp-bucket"
  mongoUri: "mongodb://streamingapp-mongodb:27017/streamingapp"

ingress:
  enabled: true
  className: traefik
  host: streamingapp.local
```

Do not commit `values-secrets.yaml`, AWS keys, JWT secrets, `.env` files, or generated environment overlays.

## Lint and render

Run these checks before installation:

```bash
helm lint .
helm template streamingapp . \
  -f values.yaml \
  -f values-k3d.yaml \
  -f values-secrets.yaml
```

Review the rendered output for correct image repositories, image tags, service ports, probe paths, S3 configuration, and secret references.

## Install or upgrade

Create the namespace once:

```bash
kubectl create namespace streaming --dry-run=client -o yaml | kubectl apply -f -
```

Install the chart:

```bash
helm upgrade --install streamingapp . \
  --namespace streaming \
  --create-namespace \
  -f values.yaml \
  -f values-k3d.yaml \
  -f values-secrets.yaml
```

Use the same command for later upgrades after changing chart templates or values.

## Verify the deployment

```bash
helm status streamingapp -n streaming
kubectl get pods,svc,deployments,ingress -n streaming
kubectl rollout status deployment -n streaming --all --timeout=5m
```

Inspect application logs when a pod is not ready:

```bash
kubectl logs -n streaming deploy/streamingapp-authService
kubectl logs -n streaming deploy/streamingapp-streamingService
kubectl logs -n streaming deploy/streamingapp-adminService
kubectl logs -n streaming deploy/streamingapp-chatService
kubectl logs -n streaming deploy/streamingapp-frontend
```

The exact deployment names are defined by the chart values and templates. List them with:

```bash
kubectl get deployments -n streaming
```

## Access through Ingress

For a local k3d deployment, map the configured host and port as required by the cluster. When using `streamingapp.local`, add a hosts entry if DNS is not configured:

```bash
echo "127.0.0.1 streamingapp.local" | sudo tee -a /etc/hosts
```

Then open:

```text
http://streamingapp.local:8080
```

Confirm the actual host and port in `values-k3d.yaml` and the rendered Ingress manifest before testing.

## Scaling and rolling updates

Scale a service temporarily with Kubernetes:

```bash
kubectl scale deployment <deployment-name> \
  --namespace streaming \
  --replicas=3
```

For a persistent setting, update the service replica count in the values file and run `helm upgrade` again. Monitor the rollout:

```bash
kubectl rollout status deployment/<deployment-name> -n streaming
kubectl get pods -n streaming -w
```

To roll back the last Helm revision:

```bash
helm history streamingapp -n streaming
helm rollback streamingapp <revision> -n streaming
```

## S3 playback troubleshooting

The API can be healthy while the catalogue is empty or playback fails. Check all of the following:

* `AWS_S3_BUCKET` points to the bucket containing the video and thumbnail objects.
* `AWS_REGION` matches the bucket region.
* The runtime identity can read the required S3 objects.
* MongoDB metadata contains a video with `status: ready`.
* The generated configuration contains the expected public streaming URL.
* The frontend was built with the correct API origin, such as `$PUBLIC_APP_ORIGIN/api` when that value is used by the image build.

Check configuration and recent events without printing Secret values:

```bash
kubectl describe pod -n streaming <pod-name>
kubectl get events -n streaming --sort-by=.lastTimestamp
kubectl get configmap -n streaming
```

## Uninstall

```bash
helm uninstall streamingapp -n streaming
```

Uninstalling the release does not necessarily delete persistent storage. Review PVCs before removing them:

```bash
kubectl get pvc -n streaming
```

## Source repository

[StreamingApp fork](https://github.com/mithlesh008/StreamingApp)
