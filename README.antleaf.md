# Antleaf Hyku Notes

This file contains Antleaf-specific setup and deployment notes for Hyku. Hyku should be checked out locally and these instructions applied there.

## Local Development

The local Stack Car hostname is driven by `APP_NAME`.

This can be configured in `.env.development` which has precedence over the default `.env` file.


### Eample `.env.development` file
```env
APP_NAME=antleaf-hyku
INITIAL_ADMIN_EMAIL=support@antleaf.com
INITIAL_ADMIN_PASSWORD=$ANTLEAF_HYKU_ADMIN_PASSWORD
```

```bash
sc up
```

Then open:

```text
https://admin-antleaf-hyku.localhost.direct/
```

To check the active admin host:

```bash
sc config | grep HYKU_ADMIN_HOST
```

## Kubernetes Deployment

### Create secret to hold Antleaf Robot Email Account for SMTP (uses Google application password)

```bash
kubectl create secret generic -n hyku hyku-secrets \
  --from-literal=SMTP_PASSWORD=$ANTLEAF_SUPPORT_SMTP_PASSWORD \
  --from-literal=INITIAL_ADMIN_PASSWORD=$ANTLEAF_HYKU_ADMIN_PASSWORD
```

Antleaf cluster deployment uses the local Helm wrapper with Antleaf image names and the Antleaf values file at `ops/deploy.yaml`.

For the command above, both the release name and namespace are `hyku`.

By default, `bin/helm_deploy` deploys image tag `latest`. To deploy a specific
tag, set `DEPLOY_TAG`; `WORKER_TAG` defaults to the same value.

### If just deploying with existing images:
```bash
export DEPLOY_TAG="v1.2.4" && \
  export DEPLOY_IMAGE="antleaf/antleaf-hyku-web" && \
  export WORKER_IMAGE="antleaf/antleaf-hyku-worker" && \
  export HELM_EXTRA_ARGS="--values ops/deploy.yaml" && \
  ./bin/helm_deploy hyku hyku
```

### If deploying with a new image tag:
```bash
export DEPLOY_TAG="v1.2.4" && \
  export DEPLOY_IMAGE="antleaf/antleaf-hyku-web" && \
  export WORKER_IMAGE="antleaf/antleaf-hyku-worker" && \
  docker buildx build -f Dockerfile --target hyku-web --platform linux/amd64,linux/arm64 --push -t $DEPLOY_IMAGE:$DEPLOY_TAG . && \
  docker buildx build -f Dockerfile --target hyku-worker --platform linux/amd64,linux/arm64 --push -t $WORKER_IMAGE:$DEPLOY_TAG . && \
  export HELM_EXTRA_ARGS="--values ops/deploy.yaml" && \
  ./bin/helm_deploy hyku hyku
```

## Uninstall

To uninstall the Antleaf release:

```bash
./bin/helm_delete hyku hyku
```

## Antleaf Values

Antleaf production-specific settings live in `ops/deploy.yaml`, including:

- ingress hosts for `antleaf-hyku.antleaf.com`
- NFS-backed storage classes and volume sizes
- resource requests and limits
- production environment variables
- initial admin email

Review `ops/deploy.yaml` before deploying changes to the cluster.