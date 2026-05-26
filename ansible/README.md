# IBM BAW — Ansible Automation

Fully automated deployment of IBM Business Automation Workflow (BAW) 25.0.1
on OpenShift, driven by Ansible and optionally run from **AWX** (Ansible Tower).

---

## Architecture

```
AWX / Ansible EE (baw-ee container)
    │
    │  kubernetes.core modules + oc CLI
    │
    ├── infra namespace
    │     ├── PostgreSQL 16 (RHEL image)  — fn_gcd_db, fn_docs_db, fn_dos_db, fn_tos_db,
    │     │                                  icn_db, baw_authoring_db, [baw_runtime_db, ae_db]
    │     └── ISVD LDAP 10.0.4            — 8 users, 10 groups
    │
    └── baw namespace
          ├── CPE 5.7          — FileNet Content Platform Engine
          ├── ICN 3.2          — IBM Content Navigator (optional)
          ├── GraphQL          — FileNet GraphQL API (optional)
          ├── UMS 25.0.1       — User Management Service (OIDC provider)
          ├── BAStudio 25.0.1  — Business Automation Studio (workflow authoring)
          ├── BAW Runtime      — Standalone workflow runtime (optional)
          └── OpenShift Routes — passthrough TLS for all services
```

**Hostname:** `baw.apps.<cluster-ingress-domain>` (auto-detected)

---

## Prerequisites

| Requirement | Notes |
|---|---|
| OpenShift 4.x | Cluster at `api.homelab.home.nl:6443` |
| `oc` CLI | In `PATH`, already logged in |
| Python 3.8+ | For local runs |
| `ansible-core` 2.15+ | `pip install ansible-core` |
| `kubernetes.core` collection | `ansible-galaxy collection install kubernetes.core` |
| IBM Entitlement Key | From https://myibm.ibm.com/products-services/containerlibrary |
| RWX StorageClass | `nfs-homelab` — used for all PVCs |
| RHEL pull secret | `registry.redhat.io` (for PostgreSQL 16 image) |

Install all collections locally:

```bash
pip install ansible-core kubernetes jinja2 cryptography
ansible-galaxy collection install -r ansible/collections/requirements.yml
```

---

## Quick Start (local)

```bash
# 1. Edit variables
notepad ansible\group_vars\all.yml   # Windows
nano ansible/group_vars/all.yml      # Linux/macOS

# 2. Log in to OpenShift
oc login --server=https://api.homelab.home.nl:6443

# 3. Full deployment (all 12 steps)
ansible-playbook -i ansible/inventory.yml ansible/site.yml

# 4. Access URLs
#   BAStudio   : https://baw.apps.<domain>/bas/CaseManager
#   Navigator  : https://baw.apps.<domain>/navigator/
#   CPE ACCE   : https://baw.apps.<domain>/acce/
#   GraphQL    : https://baw.apps.<domain>/content-services-graphql/
#   UMS        : https://baw.apps.<domain>/ums/
```

### Run individual steps / tags

```bash
# Infrastructure only (namespaces, certs, PostgreSQL, LDAP)
ansible-playbook -i ansible/inventory.yml ansible/site.yml --tags prereqs

# Resume from step 5 (CPE) — infra already running
ansible-playbook -i ansible/inventory.yml ansible/site.yml --skip-tags prereqs

# Single role (e.g., just routes)
ansible-playbook -i ansible/inventory.yml ansible/site.yml --tags routes

# Dry run
ansible-playbook -i ansible/inventory.yml ansible/site.yml --check
```

### Available tags

| Tag | Steps |
|---|---|
| `prereqs` | steps 1–4 (namespaces, certs, postgresql, ldap) |
| `step1`, `namespaces` | namespaces role |
| `step2`, `certificates` | certificates role |
| `step3`, `postgresql` | postgresql role |
| `step4`, `ldap` | ldap role |
| `step5`, `cpe`, `filenet` | CPE role |
| `step6`, `icn`, `filenet` | ICN role |
| `step7`, `graphql` | GraphQL role |
| `step8`, `ums` | UMS role |
| `step9`, `routes` | routes role |
| `step10`, `bastudio` | BAStudio role |
| `step11`, `baw_runtime` | BAW Runtime role |
| `step12`, `argocd` | ArgoCD role |

---

## Configuration Reference

All variables are in [`group_vars/all.yml`](group_vars/all.yml).

### Cluster

| Variable | Default | Description |
|---|---|---|
| `ocp_api_url` | `https://api.homelab.home.nl:6443` | OpenShift API |
| `ocp_token` | `""` | Bearer token — leave empty to use current `oc login` session |
| `cluster_ingress_domain` | `""` | Auto-detected if empty |
| `baw_subdomain` | `baw` | Resulting BAW hostname: `baw.apps.<domain>` |

### Namespaces

| Variable | Default |
|---|---|
| `baw_namespace` | `baw` |
| `infra_namespace` | `infra` |

### Storage

| Variable | Default |
|---|---|
| `storage_class_name` | `nfs-homelab` |

### LDAP (ISVD)

| Variable | Default | Description |
|---|---|---|
| `ldap_host` | `ldap-svc.infra.svc.cluster.local` | In-cluster LDAP address |
| `ldap_port` | `9389` | LDAP port |
| `ldap_base_dn` | `dc=demodom,dc=nl` | Base DN |
| `ldap_bind_dn` | `cn=bindldap,ou=dev,dc=demodom,dc=nl` | Bind user |
| `ldap_bind_password` | `Password1` | Bind password |
| `ldap_admin_user` | `bawadmin` | Default admin user |
| `ldap_admin_password` | `Password1` | Password for all LDAP users |

**Default users:** `bindldap`, `bawadmin`, `p8admin`, `bawuser`, `p8user`, `test`, `admin`, `odmadmin`

**Default groups:** `bawadmins`, `tw_admins`, `tw_authors`, `p8admins`, `bawusers`, `p8users`, `resAdministrators`, `resDeployers`, `resMonitors`, `resExecutors`

### PostgreSQL

| Variable | Default |
|---|---|
| `postgres_host` | `postgres-svc.infra.svc.cluster.local` |
| `postgres_port` | `5432` |
| `postgres_admin_user` | `admin` |
| `postgres_admin_password` | `Password1` |

### BAW Databases

| Component | DB name | User |
|---|---|---|
| GCD | `fn_gcd_db` | `fn_gcd_user` |
| DOCS object store | `fn_docs_db` | `fn_docs_user` |
| DOS object store | `fn_dos_db` | `fn_dos_user` |
| TOS object store | `fn_tos_db` | `fn_tos_user` |
| ICN | `icn_db` | `icn_user` |
| BAStudio | `baw_authoring_db` | `baw_auth_user` |
| BAW Runtime (optional) | `baw_runtime_db` | `baw_runtime_user` |
| App Engine (optional) | `ae_db` | `ae_user` |

### TLS Certificates

| Variable | Default | Description |
|---|---|---|
| `cert_work_dir` | `/tmp/baw-certs` | Directory inside EE for generated certs |
| `keystore_password` | `Password1` | Password for PKCS12 server.p12 + trusts.p12 |

The `certificates` role generates a Root CA, signs a shared pod cert, and:
- Creates the `baw-ssl-certs` Secret in the `baw` namespace
- Creates the `shared-ssl-config` ConfigMap mounted at `/shared/tls/pkcs12/` on all pods
- Registers the Root CA in the OpenShift cluster trust bundle

### Optional Components

| Variable | Default | Component |
|---|---|---|
| `deploy_icn` | `true` | IBM Content Navigator |
| `deploy_graphql` | `true` | FileNet GraphQL API |
| `deploy_ums` | `true` | User Management Service (required for BAStudio) |
| `deploy_bastudio` | `true` | BAStudio authoring environment |
| `deploy_baw_runtime` | `false` | BAW standalone runtime |
| `deploy_app_engine` | `false` | App Engine (Playback Server) |
| `deploy_odm` | `false` | Operational Decision Manager |

### Timeouts

| Variable | Default | Purpose |
|---|---|---|
| `pod_ready_timeout` | `600` | General pod readiness |
| `cpe_config_timeout` | `900` | CPE WSI endpoint + startup |
| `bastudio_job_timeout` | `1800` | BAStudio bootstrap / init jobs |

---

## Role Reference

### `namespaces` (step 1)
- Creates `baw` Project with UID range `1000140000/1000` and `anyuid` SCC
- Creates `infra` Project with `privileged` SCC
- Creates `ibm-entitlement-key` pull secrets in both namespaces

### `certificates` (step 2)
- Generates Root CA (2048-bit RSA, 10-year validity)
- Generates shared pod cert (3-year, signed by Root CA)
  - SANs: `baw.<domain>`, `*.baw.svc.cluster.local`, `*.infra.svc.cluster.local`, `*.apps.<domain>`
- Builds PKCS12 keystores: `server.p12` (keystore) + `trusts.p12` (truststore)
- Creates `baw-ssl-certs` Secret and `shared-ssl-config` ConfigMap
- Registers CA in OpenShift cluster proxy trust bundle

### `postgresql` (step 3)
- Deploys RHEL PostgreSQL 16 in the `infra` namespace
- Creates all BAW databases with tablespaces (idempotent)
- 20 Gi RWX PVC, `POSTGRESQL_MAX_PREPARED_TRANSACTIONS=100`

### `ldap` (step 4)
- Deploys IBM ISVD 10.0.4 in the `infra` namespace
- Loads base DIT, 8 users, and 10 groups via `ldapadd` (idempotent)
- Service: `ldap-svc:9389`

### `cpe` (step 5)
- Deploys CPE 5.7 in the `baw` namespace
- Liberty server overrides mounted from ConfigMap: JDBC datasources, LDAP registry, SSL
- Waits for WSI health endpoint before running GCD initialization
- Service: `cpe-svc:9443`

### `icn` (step 6)
- Deploys IBM Content Navigator 3.2 in the `baw` namespace
- Liberty server overrides: `ECMClientDS` JDBC, LDAP, SSL
- Service: `icn-svc:9443`

### `graphql` (step 7)
- Deploys FileNet GraphQL API in the `baw` namespace
- Connects to CPE WSI endpoint
- Service: `graphql-svc:9443`

### `ums` (step 8)
- Deploys UMS 25.0.1 as the OIDC identity provider
- Registers OIDC clients: `bastudio`, `psliberty`
- Service: `ums-svc:9443`

### `routes` (step 9)
- Creates OpenShift passthrough TLS Routes on `baw.<domain>`:
  - `/acce/` → cpe-svc:9443
  - `/navigator/` → icn-svc:9443
  - `/content-services-graphql/` → graphql-svc:9443
  - `/ums/` → ums-svc:9443
  - `/bas/` → bastudio-svc:9443

### `bastudio` (step 10)
Five-stage deployment (each stage is a Kubernetes Job):
1. **bootstrap** — initializes BAStudio database + UMS integration
2. ICN restart (picks up new desktop configuration)
3. **content-init** — creates FileNet object stores for BAStudio
4. BAStudio Deployment
5. **workplace-init** — configures Workplace XT
6. **case-init** — initializes IBM Case Manager

### `baw_runtime` (step 11, optional)
- Deploys BAW standalone runtime engine
- Uses `baw_runtime_db` and `baw_pdw_db` databases
- Service: `baw-runtime-svc:9443`, Route: `/baw/`

### `argocd` (step 12, optional)
- Skipped unless `configure_argocd: true`
- Verifies OpenShift GitOps instance is Available
- Grants ArgoCD controller access to `baw` and `infra` namespaces
- Creates ArgoCD Application pointing to `lloydngcobo/baw-gitops`
- Hard-refreshes Application after creation

---

## Teardown

```bash
# Full teardown (removes baw + infra namespaces)
ansible-playbook -i ansible/inventory.yml ansible/playbooks/teardown.yml

# Skip the confirmation pause (for AWX / CI)
ansible-playbook -i ansible/inventory.yml ansible/playbooks/teardown.yml -e confirm=yes
```

Teardown order: ArgoCD Application → BAStudio Jobs → `baw` namespace → `infra` namespace → baw-ca-bundle ConfigMap

---

## Custom Execution Environment

The EE is defined in [`execution-environment.yml`](execution-environment.yml).

**Contents:**
- Base: `quay.io/ansible/awx-ee:latest`
- Collections: `awx.awx`, `kubernetes.core`, `ansible.posix`, `community.crypto`
- Python: `kubernetes`, `jinja2`, `cryptography`
- Additional: `oc` + `kubectl` CLI, `openldap-clients` (ldapadd/ldapsearch)

**Build and push:**
```bash
pip install ansible-builder
ansible-builder build -t baw-ee:latest -f ansible/execution-environment.yml

docker tag baw-ee:latest ghcr.io/lloydngcobo/baw-ee:latest
docker push ghcr.io/lloydngcobo/baw-ee:latest
```

---

## AWX Setup

### Bootstrap AWX

```bash
pip install awxkit
ansible-galaxy collection install awx.awx

ansible-playbook ansible/playbooks/awx-bootstrap.yml \
  -e "awx_password=<admin-password>" \
  -e "ocp_api_token=$(oc whoami --show-token)"
```

**Resources created:**

| Resource | Name |
|---|---|
| Organization | BAW Automation |
| Execution Environment | baw-ee |
| Project | baw-awx |
| Credential | OpenShift API |
| Inventory | BAW Localhost |
| Job Template | BAW Full Install |
| Job Template | BAW Prereqs Only |
| Job Template | BAW Teardown |
| Job Template | BAW Configure ArgoCD |
| Workflow | BAW Full Deploy |

### Workflow

```
BAW Prereqs Only ──► [Approval Gate] ──► BAW Full Install ──► BAW Configure ArgoCD
```

### AWX Survey (BAW Full Install)

Key survey questions:
- IBM Entitlement Key (password, required)
- Storage Class (default: `nfs-homelab`)
- BAW Namespace (default: `baw`)
- PostgreSQL / LDAP / CPE / Keystore / UMS passwords
- BAW License (NonProd/Prod/UVU/CU)
- Deploy BAW Runtime? (false/true)
- Configure ArgoCD? (false/true)
- ArgoCD GitOps Repo URL

---

## Default Credentials

| Service | Username | Password |
|---|---|---|
| LDAP admin | `bawadmin` | `Password1` (set via `ldap_admin_password`) |
| CPE / ACCE | `p8admin` | `Password1` (set via `cpe_admin_password`) |
| BAStudio | `bawadmin` | `Password1` (set via `ums_admin_password`) |
| Navigator | `bawadmin` | `Password1` |
| PostgreSQL | `admin` | `Password1` (set via `postgres_admin_password`) |

Change all default passwords in `group_vars/all.yml` before deploying to production.
