# Keycloak with CloudNativePG and Traefik – Install via Helm

This repository installs **Keycloak** with its own PostgreSQL database
(**CloudNativePG**) and a **Traefik** `IngressRoute` with an automatic TLS
certificate.

---

## Prerequisites

* A Kubernetes cluster (e.g. k3s) with **Traefik v3** and an ACME certificate
  resolver (`IngressRoute`, `traefik.io/v1alpha1`)
* The **CloudNativePG operator** (`Cluster`, `postgresql.cnpg.io/v1`)
* Helm v3.9+ and `kubectl` pointed at the cluster
* A DNS record for your `host` pointing at the node that runs Traefik
* The cluster can pull `keycloak/keycloak` (Docker Hub) and
  `ghcr.io/cloudnative-pg/postgresql`

If Traefik or CloudNativePG is missing, `helm install` stops with
`no matches for kind "IngressRoute"` (or `"Cluster"`) `... ensure CRDs are installed first`.

---

## 1) Clone the repository

## 2) Create and edit `values.yaml`

```bash
cp values.sample.yaml values.yaml && vim values.yaml
```

Only `host` is required. Every other key has a default in `chart/values.yaml`
(see the table), so add a key only when this environment needs a different
value. Pin `image.tag` and `db.image` anyway: both upgrades change the database
and cannot be undone, so a newer chart must never upgrade an environment on its
own.

| Key | Default | Description |
|---|---|---|
| `host` | required | Public hostname of Keycloak |
| `exposeAdmin` | `false` | `true`: the whole host is public, admin console included. `false`: only `/realms/` and `/resources/` are public, the `master` realm is blocked, and the admin console is reached through a port-forward (see [step 4](#4-open-the-admin-console)) |
| `adminHostname` | `http://localhost:8080` | With `exposeAdmin: false`: the URL of the admin console through the port-forward. Keep port `8080`. Give each server its own `*.localhost` name (e.g. `http://keycloak-prod.localhost:8080`), so the browser saves a separate password per server |
| `exposeAdminApi` | `false` | With `exposeAdmin: false`: `true` also makes the admin REST API of every realm except `master` public (`/admin/realms/<realm>/`), for a backend that manages users with a service-account token. The admin console stays closed |
| `certResolver` | `le` | Traefik certificate resolver |
| `image.repository` | `keycloak/keycloak` | Keycloak image, or your own image built on it |
| `image.tag` | `26.8.1` | Keycloak version |
| `args` | `[start]` | Use `[start, --optimized]` with a pre-built image |
| `replicas` | `1` | Keycloak pods; more than one form a cluster through the database |
| `extraEnv` | `[]` | Extra Keycloak options as env vars, e.g. `KC_FEATURES`, `KC_LOG_LEVEL` |
| `resources` | 250m–1 CPU, 1Gi–1536Mi | Keycloak container; the JVM heap is ~70% of the memory limit |
| `db.image` | `ghcr.io/cloudnative-pg/postgresql:18.4-system-trixie` | PostgreSQL image |
| `db.instances` | `1` | PostgreSQL instances; 2+ adds streaming replicas with automatic failover |
| `db.storageSize` | `2Gi` | Database volume size |
| `db.storageClass` | cluster default | StorageClass of the database volume |
| `db.sharedBuffers` | `64MB` | PostgreSQL `shared_buffers`, ~25% of the DB memory limit |
| `db.priorityClassName` | none | An existing PriorityClass for the DB pods |
| `db.resources` | 100m–500m CPU, 256Mi–512Mi | Database container |
| `nodeSelector`, `tolerations` | none | Applied to the Keycloak and DB pods |

Example for a dedicated 8 vCPU / 16GB node:

```yaml
host: auth.example.com
image:
  tag: "26.8.1"
resources:
  requests:
    cpu: "4"
    memory: 7Gi
  limits:
    cpu: "4"
    memory: 7Gi
db:
  image: ghcr.io/cloudnative-pg/postgresql:18.4-system-trixie
  storageSize: 10Gi
  sharedBuffers: 768MB
  resources:
    requests:
      cpu: 1500m
      memory: 3Gi
    limits:
      cpu: 1500m
      memory: 3Gi
```

---

## 3) Install / upgrade

```bash
helm upgrade --install keycloak chart --create-namespace -n keycloak --values values.yaml
kubectl -n keycloak rollout status deploy/keycloak --timeout=10m
```

On a k3s server, run `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` first.
k3s's `kubectl` finds that file by itself, but `helm` does not: it falls back
to `http://localhost:8080` and fails with `Kubernetes cluster unreachable`.

The first start takes about 2 minutes: Keycloak builds itself, then creates
its schema. `Startup probe failed` events during that time are expected.

The commands in this README assume the release name `keycloak`; resource names
follow the release name (`<release>`, `<release>-db`, `<release>-db-auth`, ...).

---

## 4) Open the admin console

With `exposeAdmin: true`, open `https://<host>/admin/`.

With `exposeAdmin: false`, the console runs at `adminHostname` through a
port-forward. Browsers send every `*.localhost` name to `127.0.0.1`, so no
`/etc/hosts` entry is needed.

1. Once after install, set the `master` realm's Frontend URL to
   `adminHostname`. Without it, the console's login and its third-party
   cookie check go to `https://<host>/realms/master/`, which is blocked, and
   the console shows "Timeout when waiting for 3rd party check iframe
   message".

   ```bash
   ADMIN_HOSTNAME=http://localhost:8080
   kubectl -n keycloak exec deploy/keycloak -- /opt/keycloak/bin/kcadm.sh config credentials \
     --config /tmp/kcadm.config --server http://localhost:8080 --realm master --user temp-admin \
     --password "$(kubectl -n keycloak get secret keycloak-bootstrap-admin -o jsonpath='{.data.password}' | base64 -d)"
   kubectl -n keycloak exec deploy/keycloak -- /opt/keycloak/bin/kcadm.sh update realms/master \
     --config /tmp/kcadm.config -s "attributes.frontendUrl=${ADMIN_HOSTNAME}"
   kubectl -n keycloak exec deploy/keycloak -- rm -f /tmp/kcadm.config
   ```

   Set `ADMIN_HOSTNAME` to your `adminHostname`. The change makes all
   earlier `master` tokens invalid, so log in again afterwards. To undo it,
   run the same commands with `attributes.frontendUrl=`. kcadm runs inside
   the pod, so a wrong value cannot lock you out.

   After `temp-admin` is replaced (step 5), log in with your own `master`
   admin user instead; kcadm then asks for the password:

   ```bash
   read -rp 'Master admin username: ' KC_ADMIN_USER
   kubectl -n keycloak exec -it deploy/keycloak -- /opt/keycloak/bin/kcadm.sh config credentials \
     --config /tmp/kcadm.config --server http://localhost:8080 --realm master --user "$KC_ADMIN_USER"
   ```

   A service-account client with only the `create-realm` role (for example
   one for Terraform) cannot change `master` and gets `HTTP 403 Forbidden`.

2. Forward the port:

   ```bash
   kubectl -n keycloak port-forward svc/keycloak 8080:8080
   ```

   From your own machine, without a kubeconfig, forward through SSH to the
   server that runs `kubectl`:

   ```bash
   ssh -L 8080:localhost:18080 <server> 'kubectl -n keycloak port-forward svc/keycloak 18080:8080'
   ```

3. Open `<adminHostname>/admin/`.

Tools that need the `master` realm, such as the Terraform Keycloak provider,
can use `http://localhost:8080` through the same port-forward.

---

## 5) Replace the bootstrap admin

On its **first** start Keycloak creates `temp-admin` from the
`keycloak-bootstrap-admin` secret and flags it as temporary. Later starts ignore
the secret. Replace it once:

1. Get the bootstrap password:

   ```bash
   kubectl -n keycloak get secret keycloak-bootstrap-admin -o jsonpath='{.data.password}' | base64 -d && echo
   ```

2. Open the admin console and log in as `temp-admin`.
3. In the `master` realm: **Users → Add user**, username `admin`, then **Create**.
4. On the new user: **Credentials → Set password**, with *Temporary* off.
5. **Role mapping → Assign role → Filter by realm roles**, pick `admin`, **Assign**.
6. Sign out and log in as `admin`.
7. **Users → temp-admin → Action → Delete**.

---

## Data persistence

| Event | Users, realms, sessions |
|---|---|
| Pod restart, node reboot | Kept (stored in the database volume) |
| `helm upgrade` | Kept; generated passwords are reused |
| `helm uninstall` | Kept: the `keycloak-db` cluster and `keycloak-db-auth` secret stay (`helm.sh/resource-policy: keep`), and installing again with the same release name and namespace reuses them |
| `kubectl delete namespace keycloak` | **Deleted** |

The database has no backups configured. It holds every user and credential,
so back it up before production use, e.g. with the CloudNativePG Barman Cloud
plugin to object storage.

---

## Updates

**Keycloak version.** Set `image.tag` in `values.yaml` and run the install
command again. Keycloak stops before the new version starts, so login is down
for 1–2 minutes, and the new version migrates the database schema on start. A
migrated database cannot go back to the older version, so back up the database
first and read the Keycloak upgrade notes for every version you skip.

**Keycloak options.** Add them to `extraEnv` instead of changing the chart:

```yaml
extraEnv:
  - name: KC_LOG_LEVEL
    value: INFO
```

**PostgreSQL.** For a minor update, change the tag in `db.image`.
CloudNativePG restarts the database with the new image. For a major update
(e.g. 18 → 19), change `db.image` to the new major version: CloudNativePG runs
an offline in-place upgrade, so the database is down for its duration. Back up
first.

**Disk size.** Increase `db.storageSize`. This only takes effect if the
StorageClass allows volume expansion.

**Scaling.** Raise `db.instances` for database replicas with automatic
failover. Raise `replicas` for more Keycloak pods.

---

## Uninstall

```bash
helm uninstall keycloak -n keycloak
```

This removes Keycloak but keeps the database (see *Data persistence*).

⚠️ To delete everything, users included (cannot be undone):

```bash
kubectl delete namespace keycloak
```
