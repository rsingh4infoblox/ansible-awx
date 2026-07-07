# Infoblox NIOS – Create A Record (Ansible AWX)

Playbooks for managing Infoblox DNS A records using the `infoblox.nios_modules` Ansible collection, designed to run on Ansible AWX.

## Repository layout

```
├── ansible.cfg
├── execution-environment.yml     # ansible-builder definition (includes infoblox-client)
├── requirements.txt              # Python deps: infoblox-client
├── collections/
│   └── requirements.yml          # Ansible collection: infoblox.nios_modules
├── inventories/
│   └── infoblox/
│       ├── hosts.yml             # localhost (NIOS uses REST API, not SSH)
│       └── group_vars/
│           └── all.yml           # nios_provider defaults
├── playbooks/
│   └── create_a_record.yml
└── awx/
    └── credential_type.yml       # Custom credential type definition
```

## AWX setup (one-time)

### 0. Build and push a custom Execution Environment

The `infoblox.nios_modules` collection requires the `infoblox-client` Python package, which is not present in the default AWX EE. Build a custom one with `ansible-builder`:

```bash
pip install ansible-builder

# Build the image (from repo root)
ansible-builder build \
  --file execution-environment.yml \
  --tag your-registry/infoblox-ee:latest \
  --container-runtime docker

# Push to your registry
docker push your-registry/infoblox-ee:latest
```

Then in AWX go to **Administration → Execution Environments → Add** and register the image.

Set that EE on the Job Template (step 4 below).

### 1. Create a custom credential type

Go to **Administration → Credential Types → Add** and create:

- **Name:** `Infoblox NIOS`
- **Kind:** Cloud
- **Input Configuration** – paste the `inputs` block from [awx/credential_type.yml](awx/credential_type.yml)
- **Injector Configuration** – paste the `injectors` block from [awx/credential_type.yml](awx/credential_type.yml)

### 2. Create a credential

Go to **Resources → Credentials → Add**:

| Field | Value |
|---|---|
| Credential Type | Infoblox NIOS |
| NIOS Host | `<grid-manager-ip-or-hostname>` |
| Username | `<nios-username>` |
| Password | `<nios-password>` |
| WAPI Version | `2.1` (or match your NIOS version) |
| SSL Verify | `false` (set `true` for production with valid cert) |

### 3. Add this repo as a Project

Go to **Resources → Projects → Add**:

| Field | Value |
|---|---|
| Source Control Type | Git |
| Source Control URL | `<this-repo-url>` |
| Update Revision on Launch | ✓ |

AWX will automatically install `infoblox.nios_modules` from `collections/requirements.yml` on each project sync.

### 4. Create a Job Template

Go to **Resources → Job Templates → Add**:

| Field | Value |
|---|---|
| Inventory | `infoblox` (or source from the repo) |
| Project | *(the project from step 3)* |
| Playbook | `playbooks/create_a_record.yml` |
| Credentials | *(the Infoblox NIOS credential from step 2)* |

Add a **Survey** with:

| Question | Variable | Required |
|---|---|---|
| DNS record name | `record_name` | Yes |
| IPv4 address | `record_ipv4addr` | Yes |
| DNS view | `dns_view` | No (default: `default`) |
| TTL | `record_ttl` | No (default: `3600`) |
| State | `record_state` | No (default: `present`) |

## Variables reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `nios_host` | Yes | — | NIOS Grid Manager host (injected by credential) |
| `nios_username` | Yes | — | NIOS username (injected by credential) |
| `nios_password` | Yes | — | NIOS password (injected by credential) |
| `nios_wapi_version` | No | `2.1` | WAPI API version |
| `nios_ssl_verify` | No | `false` | SSL certificate verification |
| `record_name` | Yes | — | FQDN for the A record (e.g. `host.example.com`) |
| `record_ipv4addr` | Yes | — | IPv4 address for the A record |
| `dns_view` | No | `default` | NIOS DNS view |
| `record_ttl` | No | `3600` | TTL in seconds |
| `record_state` | No | `present` | `present` to create/update, `absent` to delete |
| `record_comment` | No | `Managed by Ansible AWX` | Record comment |

## Running locally (without AWX)

```bash
pip install ansible infoblox-client
ansible-galaxy collection install -r collections/requirements.yml

ansible-playbook playbooks/create_a_record.yml \
  -i inventories/infoblox/hosts.yml \
  -e nios_host=172.28.83.22 \
  -e nios_username=admin \
  -e nios_password=infoblox \
  -e record_name=myhost.example.com \
  -e record_ipv4addr=10.0.0.100
```

> **Note:** `nios_host` is the bare IP or hostname — no `https://` prefix.
