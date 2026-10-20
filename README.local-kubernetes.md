# Local Kubernetes Deployment

This guide deploys Antleaf Hyku to the Kubernetes cluster bundled with Docker
Desktop on macOS.

## Prerequisites

Make sure `kubectl` is targeting Docker Desktop rather than the prod cluster.

```bash
kubectl config use-context docker-desktop
kubectl config current-context
kubectl --context docker-desktop cluster-info
```

`kubectl config current-context` must print `docker-desktop`. Do not continue if
it names the prod cluster.

Install Docker Desktop with Kubernetes enabled, Helm 4, kubectl, and Python 3.
Set `ANTLEAF_HYKU_DB_PASSWORD` and `ANTLEAF_HYKU_ADMIN_PASSWORD` to local
passwords. The commands below use Helm 4's `--rollback-on-failure` option.
Chart versions are pinned to the versions used to validate this setup.

## Choose your public hostnames

Choose a DNS zone managed in your Cloudflare account and a root subdomain for
this deployment. Replace the example zone and contact email below with your
own values before running the commands.

```bash
export HYKU_ZONE="example.com"
export HYKU_SUBDOMAIN="hyku-dev"
export HYKU_HOST="${HYKU_SUBDOMAIN}.${HYKU_ZONE}"
export HYKU_TENANT="test"
export HYKU_TENANT_HOST="${HYKU_TENANT}.${HYKU_HOST}"

export LETSENCRYPT_EMAIL="you@example.com"
```

Edit `ops/deploy-local.yaml` before deploying:

- Replace every `hyku-dev.example.com` with your chosen `$HYKU_HOST`, including
  the root and wildcard entries under `ingress.hosts` and `ingress.tls.hosts`,
  `HYKU_ADMIN_HOST`, `HYKU_ROOT_HOST`, and `HYKU_DEFAULT_HOST`.
  Preserve the `*.` and `%{tenant}.` prefixes where present.
- Set `INITIAL_ADMIN_EMAIL` and `HYKU_CONTACT_EMAIL` to your own addresses.

Enter literal values in the YAML; Helm does not expand the shell variables.
Use this file directly for every local Helm install or upgrade. Keep personal
hostname and email substitutions out of commits so Git retains reusable example
values.

Run commands from the repository root and keep the variables above set in the
terminal used for the remaining steps. In portal instructions, enter the values
of the variables; Cloudflare does not expand shell variables.

## Install the PostgreSQL operator

Install CloudNativePG once per Docker Desktop cluster:

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts --force-update
helm repo update cnpg
helm --kube-context docker-desktop upgrade --install cnpg cnpg/cloudnative-pg \
  --version 0.29.1 \
  --namespace cnpg-system \
  --create-namespace \
  --timeout 10m \
  --wait
```

## Create the standalone database

Set `ANTLEAF_HYKU_DB_PASSWORD` to a local password. Use the same value for the
database Secret and the Hyku Helm release. CloudNativePG requires a
`kubernetes.io/basic-auth` Secret with `username` and `password` keys in the
same namespace as its Cluster:

```bash
kubectl --context docker-desktop create namespace postgres
kubectl --context docker-desktop create secret generic hyku-db-credentials \
  --namespace postgres \
  --type kubernetes.io/basic-auth \
  --from-literal=username=hyrax \
  --from-literal="password=${ANTLEAF_HYKU_DB_PASSWORD}"
kubectl --context docker-desktop apply -f ops/postgres-local.yaml
kubectl --context docker-desktop --namespace postgres wait \
  --for=condition=Ready cluster/ppostgres-antleaf --timeout=10m
kubectl --context docker-desktop --namespace postgres get cluster,pods,services
```

The database endpoint is
`ppostgres-antleaf-rw.postgres.svc.cluster.local:5432`. On first bootstrap,
CloudNativePG creates the `hyrax` database and application role, grants the role
`CREATEDB` for Hyku's setup and tenant databases, then installs the PostgreSQL
extensions Hyku requires. The database is stored on its own
2 Gi persistent volume and remains available when the Hyku Helm release is
upgraded or removed.

## Create the Hyku namespace and secret

Set `ANTLEAF_HYKU_ADMIN_PASSWORD` to a local admin password, then create the
application namespace and Secret:

```bash
kubectl --context docker-desktop create namespace hyku
kubectl --context docker-desktop create secret generic hyku-secrets \
  --namespace=hyku \
  --from-literal="INITIAL_ADMIN_PASSWORD=${ANTLEAF_HYKU_ADMIN_PASSWORD}" \
  --from-literal="DB_PASSWORD=${ANTLEAF_HYKU_DB_PASSWORD}"
```

## Install NGINX and cert-manager

```bash
helm --kube-context docker-desktop upgrade --install nginx-ingress nginx-ingress \
  --repo https://helm.nginx.com/stable --version 2.8.0 \
  --namespace ingress-system --create-namespace \
  --values ops/nginx-ingress-local.yaml --skip-crds --wait --timeout 5m

helm --kube-context docker-desktop upgrade --install cert-manager cert-manager \
  --repo https://charts.jetstack.io --version v1.21.2 \
  --namespace cert-manager --create-namespace --set crds.enabled=true \
  --wait --timeout 5m
```

NGINX uses an internal ClusterIP Service. Cloudflare Tunnel is the public entry
point; no inbound port, host `cloudflared` installation, or port-forward is needed.

## Request the origin certificate

Use HTTP-01 validation for the root hostname. Every tunnel route uses this same
Origin Server Name, so tenant routes do not require wildcard origin certificates
or a Cloudflare DNS API token. The incoming HTTP Host header still selects the
Hyku tenant.

```bash
kubectl --context docker-desktop apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: hyku-letsencrypt
  namespace: hyku
spec:
  acme:
    email: ${LETSENCRYPT_EMAIL}
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: hyku-letsencrypt-account
    solvers:
      - http01:
          ingress:
            name: hyku-hyrax
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hyku-letsencrypt
  namespace: hyku
  annotations:
    cert-manager.io/issue-temporary-certificate: "true"
spec:
  secretName: hyku-letsencrypt-tls
  issuerRef:
    name: hyku-letsencrypt
    kind: Issuer
  privateKey:
    rotationPolicy: Always
  dnsNames:
    - ${HYKU_HOST}
EOF
```

The temporary certificate lets NGINX start before issuance. It is not publicly
trusted. Issuance will wait until the application ingress and tunnel are running.
The HTTP-01 solver edits the existing ingress to avoid F5 NGINX host conflicts.
The local values include `acme.cert-manager.io/http01-edit-in-place: "true"`
to exempt the challenge path from NGINX redirects.

## Deploy

Download the Hyrax chart if `hyrax/` is not already present:

```bash
helm pull --untar oci://ghcr.io/samvera/charts/hyrax --version 3.7.3
```

Apply the patch below once to that chart. Skip it if
`hyrax/templates/_helpers.tpl` already contains the
`if .Values.sharedPvcAccessModes` override. Docker Desktop's `standard`
storage class requires `ReadWriteOnce` here; the local values use pod affinity
to place web and worker on the same node so they can share those volumes.
If you download a fresh copy of the chart later, reapply this patch.

```bash
patch -d hyrax -p1 <<'PATCH'
--- a/templates/_helpers.tpl
+++ b/templates/_helpers.tpl
@@ -231,3 +231,8 @@
 {{- define "hyrax.sharedPvcAccessModes" -}}
-{{- if .Values.worker.enabled }}
+{{/* Override only when all PVC consumers are constrained to one node. */}}
+{{/* Otherwise, keep the worker-aware default access mode below. */}}
+{{- if .Values.sharedPvcAccessModes }}
+accessModes:
+{{- toYaml .Values.sharedPvcAccessModes | nindent 2 }}
+{{- else if .Values.worker.enabled }}
 accessModes:
PATCH
```

Hyku reads its database password from `hyku-secrets` at runtime. The same
value is supplied to Helm so the chart can generate Hyku's `DATABASE_URL`:

```bash
helm --kube-context docker-desktop upgrade --install hyku ./hyrax \
  --namespace hyku \
  --values ops/deploy-local.yaml \
  --set-string externalPostgresql.password="${ANTLEAF_HYKU_DB_PASSWORD}" \
  --timeout 20m \
  --rollback-on-failure \
  --wait
```

Check the application pods after Helm completes:

```bash
kubectl --context docker-desktop --namespace hyku get pods
```

Confirm the worker deployment is available:

```bash
kubectl --context docker-desktop --namespace hyku get deployment hyku-hyrax-worker
```

## Install the tunnel connector

Create a remotely managed tunnel in Cloudflare if you do not already have one.
Run the following in an interactive terminal to enter its token with hidden input.
The token is sent through stdin to Kubernetes, without a token file or command
argument. Skip token creation if `hyku/cloudflared-token` already exists.

```bash
python3 - <<'PYTOKEN'
import getpass, json, subprocess
token = getpass.getpass("Cloudflare tunnel token (hidden): ").strip()
if not token:
    raise SystemExit("A tunnel token is required")
secret = {"apiVersion": "v1", "kind": "Secret",
          "metadata": {"name": "cloudflared-token", "namespace": "hyku"},
          "type": "Opaque", "stringData": {"token": token}}
subprocess.run(["kubectl", "--context", "docker-desktop", "apply", "--server-side",
                "--field-manager=local-tunnel", "-f", "-"],
               input=json.dumps(secret), text=True, check=True)
PYTOKEN
kubectl --context docker-desktop apply -f ops/cloudflared-local.yaml
kubectl --context docker-desktop -n hyku rollout status deployment/cloudflared --timeout=120s
```

If rotating an existing token, restart `deployment/cloudflared` after replacing
the Secret. Do not paste tokens into chat or commit them to Git.

## Configure Cloudflare and finish issuance

1. In <https://dash.cloudflare.com>, select your account and open
   **Networking → Tunnels → your tunnel → Published application routes**.
2. Add or edit the root route: Subdomain `$HYKU_SUBDOMAIN`, Domain `$HYKU_ZONE`,
   Path empty. For initial bootstrap, use Service URL
   `http://nginx-ingress-controller.ingress-system.svc.cluster.local:80`.
   Leave HTTP Host Header empty. This is only the connector-to-NGINX hop;
   browser-to-Cloudflare and the tunnel itself remain encrypted.
3. Ensure the root proxied CNAME points to `<TUNNEL-UUID>.cfargotunnel.com`.
   The portal normally creates this record automatically.

The root route's `/.well-known/acme-challenge/` path must be publicly reachable
for issuance and renewal. If using Cloudflare Access or WAF challenges, exempt
that path.

Wait for the public certificate:

```bash
kubectl --context docker-desktop -n hyku wait --for=condition=Ready \
  certificate/hyku-letsencrypt --timeout=10m
```

Then edit the root route and use these permanent settings:

| Setting | Value |
| --- | --- |
| Service URL | `https://nginx-ingress-controller.ingress-system.svc.cluster.local:443` |
| Origin Server Name | Value of `$HYKU_HOST` |
| CA Pool | Empty |
| No TLS Verify / Disable TLS certificate verification | Off |
| HTTP Host Header | Empty |

If the portal separates Type and URL, choose HTTPS and omit `https://` from the
URL field. Never point the service at the public hostname; that creates a loop.


| Field | Value |
| --- | --- |
| Type | CNAME |
| Name | `${HYKU_TENANT}.${HYKU_SUBDOMAIN}` |
| Target | Copy the root record's `<TUNNEL-UUID>.cfargotunnel.com` target |
| Proxy status | Proxied (orange cloud) |
| TTL | Auto |

Cloudflare provides a separate browser-facing edge certificate. Ensure it covers
`$HYKU_TENANT_HOST`; nested names may require Cloudflare Advanced Certificate
Manager. An origin Let's Encrypt certificate does not extend edge coverage.

## Notes

A fresh database has no tenants. Create the test tenant through Hyku account
administration and set its domain to `$HYKU_TENANT_HOST`. Existing tenant domains
are not changed by DNS or `HYKU_DEFAULT_HOST` updates.
For each new tenant, complete [Cloudflare routing for each tenant](#cloudflare-routing-for-each-tenant)
before checking its public URL.

After creating the tenant, check `https://$HYKU_HOST` and
`https://$HYKU_TENANT_HOST`. cert-manager renews the origin certificate
automatically and NGINX loads the updated Secret. Keep the tunnel, root DNS
record, and ACME challenge path available for renewal.

Web and worker share ReadWriteOnce volumes and are constrained to the same node.
Local dependencies are standalone CloudNativePG, Redis, Solr, ZooKeeper and FITS;
production NFS and SMTP are not used. PostgreSQL extensions are installed only
when bootstrapping a fresh database volume.

### Cloudflare routing for each tenant

For every tenant, add **one published application route and one proxied CNAME**
to the existing tunnel. Repeat these steps whenever you add a tenant, using its
short name as `$HYKU_TENANT`.

1. Open **Networking → Tunnels → your tunnel → Published application routes**.
2. Select **Add a published application** and enter:

| Setting | Value |
| --- | --- |
| Subdomain | `${HYKU_TENANT}.${HYKU_SUBDOMAIN}` |
| Domain | `$HYKU_ZONE` |
| Path | Empty |
| Service URL | `https://nginx-ingress-controller.ingress-system.svc.cluster.local:443` |
| Additional application settings → TLS → Origin Server Name | Root hostname: `$HYKU_HOST` |
| CA Pool | Empty |
| No TLS Verify / Disable TLS certificate verification | Off |
| HTTP Host Header | Empty |

Keep Origin Server Name set to the root hostname even for tenant routes.

3. Save the route.
4. Under **your zone → DNS → Records**, check whether Cloudflare created the
   tenant's record automatically. If it is missing, select **Add record**:

## Remove the local deployment

To remove only Hyku and retain volumes, uninstall its Helm release. Deleting the
namespace removes its volumes too. For a complete reset, the following commands
permanently remove local application and database data. Securely retain the tunnel
token first if you want to reuse it. Delete certificate resources while
cert-manager is still running so its challenge finalizers can complete.

```bash
kubectl --context docker-desktop -n hyku delete certificate hyku-letsencrypt --ignore-not-found
helm --kube-context docker-desktop -n hyku uninstall hyku
kubectl --context docker-desktop delete namespace hyku --wait=true
kubectl --context docker-desktop delete namespace postgres --wait=true
helm --kube-context docker-desktop -n cnpg-system uninstall cnpg
kubectl --context docker-desktop delete namespace cnpg-system --wait=true
helm --kube-context docker-desktop -n ingress-system uninstall nginx-ingress
kubectl --context docker-desktop delete namespace ingress-system --wait=true
# Only if no other local applications use cert-manager:
helm --kube-context docker-desktop -n cert-manager uninstall cert-manager
kubectl --context docker-desktop delete namespace cert-manager --wait=true
```

Helm retains operator CRDs. Follow the guide from the beginning to rebuild.
Recreate the token Secret and bootstrap certificate issuance through the HTTP
origin route again.
