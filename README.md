# Cloud‑Init Bootstrap: Kind + ingress‑nginx + cert‑manager + PostgreSQL

> One‑shot cloud‑init that provisions a reproducible Kubernetes lab/edge cluster on **Ubuntu 24.04** with:
>
> - **Kind** (Kubernetes in Docker) pinned to a stable node image
> - **ingress‑nginx** via the upstream **Kind provider** manifest
> - **cert‑manager** (Let’s Encrypt HTTP‑01 ready) via Helm OCI
> - **PostgreSQL** tuned (`max_connections=200`, `superuser_reserved_connections=5`), open to any IP (for convenience)
> - A **DB restore inbox**: drop a `.tar.gz` with dumps into `/var/backups/pg/inbox/` and it restores automatically
>
> ⚠️ **Security note:** kubeconfig is shared system‑wide and PostgreSQL accepts connections from anywhere. This is deliberate (to match lab/PoC needs). Review and tighten for production.

---

## What this repo contains

- `cloud-config.yaml` — a cloud‑init file orchestrating everything with systemd stages:
    - **00-system**: Docker, sysctl, bash completion, Docker log rotation
    - **10-kind**: Kind cluster (2 nodes), kubeconfig rewrite to your public `SERVER_IP:API_PORT`
    - **20-ingress**: `ingress-nginx` installed *from the Kind provider manifest* and pinned to run on the control‑plane
    - **30-certmgr**: `cert-manager` installed from the official Helm **OCI** chart, staging & prod ClusterIssuers created
    - **40-postgres**: PostgreSQL hardening & tuning; listens on `*`; HBA allows `0.0.0.0/0` and `::0/0`; SCRAM‑SHA‑256
    - **50-dbrestore**: a watcher that processes new `.tar.gz` backups in `/var/backups/pg/inbox`

The file is idempotent; stages have completion markers in `/var/lib/bootstrap/complete.d/` so reboots won’t redo work.

---

## Requirements

- VM with **Ubuntu 24.04** and **cloud‑init** support
- Public IPv4 (or NAT with ports **80**, **443**, **6443** forwarded to the host)
- DNS A/AAAA records pointing to your IP for TLS issuance (optional but recommended)
- Outbound internet access to pull container images and manifests

---

## Quick start

1) **Set your variables** (these placeholders are referenced by `cloud-config.yaml`):
```yaml
SERVER_IP:        <your public IP>
KIND_VERSION:     v0.30.0
API_PORT:         6443
ACME_EMAIL:       you@example.com
PG_PASSWORD:      <strong password>
```

2) **Create the VM** and pass `cloud-config.yaml` as user‑data. Examples:
- On most clouds: paste into the “User data (cloud‑init)” field.
- With `cloud-localds` (for local/libvirt): `cloud-localds seed.iso cloud-config.yaml` and attach the ISO.

3) **Open ports** 80/443 (ingress) and 6443 (K8s API) on your firewall/security group.

4) **Watch bootstrap logs** (optional):
```bash
journalctl -u bootstrap-10-kind.service     -f
journalctl -u bootstrap-20-ingress.service  -f
journalctl -u bootstrap-30-certmgr.service  -f
journalctl -u bootstrap-40-postgres.service -f
```

5) **Use kubectl** (kubeconfig is shared system‑wide):
```bash
export KUBECONFIG=/etc/kubernetes/kubeconfig
kubectl get nodes -o wide
```

---

## Components & Versions

- **Kubernetes (Kind)**: `kindest/node:v1.33.4` (control‑plane and worker)
- **Kind CLI**: `${KIND_VERSION}` (downloaded with checksum verification)
- **ingress‑nginx**: upstream Kind provider manifest `controller‑v1.13.2`
- **cert‑manager**: `v1.18.2` from Helm **OCI** with CRDs enabled
- **PostgreSQL**: Ubuntu 24.04 package, configured for SCRAM‑SHA‑256

> To upgrade, bump the tags above and re‑apply `cloud-config.yaml` on a fresh VM.

---

## PostgreSQL “inbox” restores

**Path layout**
```
/var/backups/pg/
├── inbox/      # put *.tar.gz here
├── archive/    # processed tarballs are moved here
└── staged/     # temporary extraction area
```
A systemd path unit triggers `/opt/bootstrap/dbrestore-scan.sh` whenever a new `*.tar.gz` appears in `inbox/`. The script:
- Calculates a SHA256 and skips if already processed
- Extracts the archive into a timestamped directory
- Executes `globals.sql` if present
- Reads `db_meta.tsv` (TSV: `DB	OWNER	ENC	LC_COLLATE	LC_CTYPE`) and **creates databases** as needed
- Restores any of: `*.dumpdir`, `*.dump` and `*.sql` into corresponding databases
- Moves the tarball to `archive/`

**Example scp upload**
```bash
# From your laptop to the VM:
scp ./backups/2025-09-01_120001.tar.gz ubuntu@${SERVER_IP}:/var/backups/pg/inbox/
```
Once copied, watch the log:
```bash
journalctl -u dbrestore-inbox.service -n 200 --no-pager
```

**Expected archive contents**
```
globals.sql
db_meta.tsv
mydb.dumpdir/           # or mydb.dump
anotherdb.sql
postgres.dumpdir        # optional
...
```
> Connections:
> - IPv4: `psql "postgresql://postgres:${PG_PASSWORD}@${SERVER_IP}:5432/postgres"`
> - IPv6: `psql "postgresql://postgres:${PG_PASSWORD}@[${SERVER_IPv6}]:5432/postgres"`

---

## TLS with cert‑manager (Let’s Encrypt HTTP‑01)

The bootstrap creates two ClusterIssuers: `letsencrypt-staging` and `letsencrypt-prod`. To issue a cert for an app Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts: [ demo.example.com ]
      secretName: demo-tls
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-svc
                port:
                  number: 80
```
Ensure your DNS `A` record points to the VM, and port **80** is reachable from the internet (for ACME HTTP‑01).

---

## Ingress controller details

This project **uses the upstream Kind provider manifest** (not Helm) and pins the controller to the control‑plane node. The bootstrap:
1. Labels the control‑plane node: `ingress-ready=true`
2. Applies the Kind‑specific manifest for `ingress-nginx`
3. Waits for the admission jobs, verifies the webhook CA bundle, and **only if needed** removes a broken ValidatingWebhookConfiguration to avoid blocking Ingress creation (including ACME HTTP‑01)

If you want to restore the webhook later, simply re‑apply the manifest:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.2/deploy/static/provider/kind/deploy.yaml
```

---

## Security considerations (read me)

- **Kubeconfig exposure:** `/etc/kubernetes/kubeconfig` is world‑readable so any OS Login user can access the cluster.
- **PostgreSQL exposure:** `listen_addresses='*'` and HBA rules for `0.0.0.0/0` and `::0/0` are permissive by design.
- **Certificates:** Let’s Encrypt HTTP‑01 requires public reachability on port 80.

Harden for production by scoping kubeconfig/RBAC, narrowing HBA CIDRs, and using DNS‑01 for ACME if port 80 is not acceptable.

---

## Troubleshooting

- **ingress‑nginx webhook stuck** (empty CA bundle):
  ```bash
  kubectl get validatingwebhookconfiguration ingress-nginx-admission -o yaml | grep caBundle -n
  # if empty, re-run:
  kubectl -n ingress-nginx delete job/ingress-nginx-admission-patch
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.2/deploy/static/provider/kind/deploy.yaml
  ```
  As a last resort, delete the validating webhook:
  ```bash
  kubectl delete validatingwebhookconfiguration ingress-nginx-admission
  ```

- **cert-manager** not issuing:
  ```bash
  kubectl -n cert-manager get pods
  kubectl describe challenge -A
  kubectl describe order -A
  ```

- **PostgreSQL** connection limits:
    - `max_connections=200` and `superuser_reserved_connections=5` leave a few slots for `postgres` when saturated.

---

## License

This project is licensed under the **[MIT License](LICENSE)**.
