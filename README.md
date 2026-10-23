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
| `exposeAdmin` | `false` | `true`: the whole host is public, admin console included. `false`: only `/realms/`, `/resources/` and `/robots.txt` are public, the `master` realm is blocked, and the admin console is reached through a port-forward |
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

The first start takes about 2 minutes: Keycloak builds itself, then creates
its schema. `Startup probe failed` events during that time are expected.

The commands in this README assume the release name `keycloak`; resource names
follow the release name (`<release>`, `<release>-db`, `<release>-db-auth`, ...).

---

## 4) Open the admin console

With `exposeAdmin: true`, open `https://<host>/admin/`.

With `exposeAdmin: false`:

```bash
kubectl -n keycloak port-forward svc/keycloak 8080:8080
```

and open http://localhost:8080/admin/ (the local port must be 8080).

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
