# ⚙️ DevSecOps Ansible Automation

An Ansible-native reimplementation of the [DevSecOps Pipeline Suite](https://github.com/nargiz5/devsecops-pipeline) — the same Vault → GitLab → DefectDojo → Grafana security stack, but provisioned through idempotent, taggable Ansible roles instead of a monolithic bash script.

Where the original bootstraps everything with `bootstrap.sh` and static `.env` values, this version treats **Vault as a live secret store**: each role fetches what it needs from Vault at runtime and writes new secrets (tokens, API keys, project IDs) back into it as the stack comes online.

---

## 📐 What Gets Provisioned

```
site.yml (targets: localhost)
   │
   ├─ cleanup   → stop/remove any previous stack (containers, volumes, networks)
   ├─ docker    → install Docker Engine, log in to Docker Hub, configure insecure registry
   ├─ vault     → run HashiCorp Vault (dev mode), seed initial secrets
   ├─ gitlab    → start GitLab CE + Runner, create root/admin/user accounts,
   │              import target project, register the runner, push tokens back to Vault
   ├─ dojo      → clone & start DefectDojo, create admin user, generate API key,
   │              create a Product + Engagement, push results back to Vault
   ├─ pipeline  → render .gitlab-ci.yml from a Jinja2 template using live Vault data,
   │              push it into the imported GitLab project via the API
   └─ grafana   → (re)generate provisioning files from templates, start Grafana,
                  wire up the DefectDojo Postgres datasource + dashboard
```

Every role after `vault` follows the same pattern: **pull secrets → do the work → write results back**, so the Vault entry at `secret/data/devsecops` ends up as a running record of the entire environment (tokens, project IDs, API keys) instead of a static seed file.

---

## 🧰 Stack

| Component | Role file | Notes |
|---|---|---|
| **Docker Engine** | `roles/docker` | Installed from the official Docker apt repo; configures an insecure local registry |
| **HashiCorp Vault** | `roles/vault` | Runs in dev mode; central store for every credential used downstream |
| **GitLab CE + Runner** | `roles/gitlab` | Full account/project/runner bootstrap via `gitlab-rails runner` and the REST API |
| **DefectDojo** | `roles/dojo` | Cloned from upstream, brought up via its own `docker compose`, configured via Django shell |
| **Pipeline generation** | `roles/pipeline` | Renders and pushes `.gitlab-ci.yml` dynamically from Vault-sourced values |
| **Grafana** | `roles/grafana` | Provisioning (datasource + dashboard) fully templated and regenerated on each run |
| **Cleanup** | `roles/cleanup` | Idempotency helper — tears down known containers/volumes before a fresh run |

---

## 📁 Project Structure

```
devsecops-ansible/
├── site.yml                  # Master playbook — orchestrates all roles against localhost
├── inventory.ini             # Single-host local inventory
├── group_vars/
│   └── all.yml                # Ansible Vault–encrypted variables (AES256) — see below
└── roles/
    ├── cleanup/tasks/
    ├── docker/tasks/
    ├── vault/tasks/
    ├── gitlab/
    │   ├── tasks/
    │   └── templates/docker-compose.yml.j2   # reference only, not invoked by tasks/main.yml
    ├── dojo/tasks/
    ├── pipeline/
    │   ├── tasks/
    │   └── templates/gitlab-ci.yml.j2
    └── grafana/
        ├── tasks/
        └── templates/
            ├── docker-compose.yml.j2
            ├── ds_dojo.yaml.j2
            ├── dashboard_config.yaml.j2
            └── security_dashboard.json.j2
```

---

## 🔑 Secret Management

All variables (`vault_token`, `root_password`, `dockerhub_pat`, `telegram_token`, etc.) are meant to live in **`group_vars/all.yml`, encrypted with Ansible Vault (AES256)** — nothing sensitive is ever committed in plain text.

At runtime, the flow is:

1. `roles/vault` starts a Vault dev server and seeds it with the initial values from `group_vars/all.yml`
2. Every subsequent role (`gitlab`, `dojo`, `pipeline`, `grafana`) reads what it needs from Vault over its HTTP API using `vault_token`, rather than referencing Ansible variables directly
3. Roles that generate new credentials (GitLab tokens, DefectDojo API keys, project IDs) merge them back into the same Vault path, so the running environment's state is always retrievable from Vault — not scattered across role outputs

### ⚠️ About the `group_vars/all.yml` in this repo

The `group_vars/all.yml` committed here is encrypted with a vault password that is **no longer available**. It cannot be decrypted by anyone, including the original author, and it is kept in the repo purely as a structural reference — **do not attempt to use it**.

**Every user must generate their own `group_vars/all.yml`** before running this playbook:

```bash
rm group_vars/all.yml
ansible-vault create group_vars/all.yml
```

This opens an editor and prompts you to set your own new vault password (choose a strong one — you'll need it every time you run the playbook). Paste in the following structure with your own values:

```yaml
host_ip: <your-server-ip>

vault_token: <choose-a-strong-token>
vault_url: http://<your-server-ip>:8200

dockerhub_username: <your-dockerhub-username>
dockerhub_pat: <your-dockerhub-personal-access-token>

gitlab_port: 9500
registry_port: 5002
root_password: <choose-a-strong-password>
admin_username: adminuser
admin_password: <choose-a-strong-password>
admin_email: admin@example.com
user_username: normaluser
user_password: <choose-a-strong-password>
user_email: user@example.com
import_url: https://gitlab.practical-devsecops.training/pdso/django.nv.git
import_project_name: django-nv-import
gitlab_url: http://<your-server-ip>:9500
registry_url: <your-server-ip>:5002

dojo_port: 8080
dojo_admin_user: admin
dojo_admin_password: <choose-a-strong-password>
dojo_db_password: <choose-a-strong-password>

grafana_password: <choose-a-strong-password>

telegram_token: <your-telegram-bot-token>
telegram_chat_id: <your-telegram-chat-id>
```

Save and exit — Ansible Vault encrypts the file automatically. There is no shared or default vault password; each user creates and keeps their own.

---

## 🚀 Usage

### Prerequisites

- Ubuntu 22.04+ target host (this playbook targets `localhost` by default — see `inventory.ini`)
- Ansible ≥ 2.14
- The `community.docker` collection:
  ```bash
  ansible-galaxy collection install community.docker
  ```
- Docker Hub account (for the `docker` role's registry login) and a Telegram bot (for pipeline alerts)

### 1. Clone the repository

```bash
git clone https://github.com/nargiz5/devsecops-ansible.git
cd devsecops-ansible
```

### 2. Create your own secrets file

Follow the **"About the `group_vars/all.yml` in this repo"** section above to generate your own encrypted `group_vars/all.yml`.

### 3. Run the full playbook

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

### 4. Or run a single stage with tags

Every role is tagged, so you can re-provision just one piece without tearing down the rest:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --tags gitlab
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --tags grafana
```

Available tags: `cleanup`, `docker`, `vault`, `gitlab`, `dojo`, `pipeline`, `grafana`.

---

## ⚠️ Security Notes

- **The `group_vars/all.yml` in this repo is permanently undecryptable** (vault password lost) and is kept only as a reference for the expected variable structure. Never assume an encrypted file you find in a public repo is safe to rely on — always generate your own.
- **Never commit `group_vars/all.yml` unencrypted.** If you ever save it with `ansible-vault` disabled, rotate every credential inside it immediately and re-encrypt.
- The Vault instance here runs in **dev mode** (`vault server -dev`) for lab convenience — it auto-unseals and is not meant for production use. Swap in a properly initialized, unsealed Vault before using this outside a lab.
- `roles/cleanup` force-removes containers and volumes by name — review the list before running it against anything other than this lab's own stack.
- Rotate the vault password itself periodically, and never reuse it as a service password elsewhere in the stack.
- All secrets that were previously hardcoded in template files (`ds_dojo.yaml.j2`, `grafana/docker-compose.yml.j2`) have been parameterized to read from Vault via `dojo_db_password` and `grafana_password` — no plain-text credentials remain in any tracked file.

---

## 📄 License

This project is intended for educational and lab use.
