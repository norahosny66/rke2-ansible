# RKE2 Single-Node via Ansible (Calico CNI)

Idempotent Ansible project provisioning single-node RKE2 on a remote Linux
host, with Calico as the CNI installed via Helm, all configuration
externalized to `group_vars`, and secrets encrypted with ansible-vault.

## Structure

```
ansible.cfg                      # vault password file + defaults
site.yml                         # orchestrates the 5 roles
requirements.yml                 # kubernetes.core, community.general, ansible.posix
inventory/hosts.ini              # rke2_servers group
group_vars/rke2_servers/
  main.yml                       # ALL non-secret config, externalized
  vault.yml                      # ansible-vault encrypted (join token)
roles/
  prereqs/                       # packages, swap, kernel modules, sysctl
  rke2_install/                  # get.rke2.io installer, version-idempotent
  rke2_config/                   # templates config.yaml; handler on change
  calico_cni/                    # Helm install + Calico via helm module
  cluster_verify/                # asserts Ready + Calico Running; fetches kubeconfig
```

## How each requirement maps to code

- **Roles, not one main.yaml** — five focused roles wired in `site.yml`.
- **Handler restarts only on config change** —
  `roles/rke2_config/tasks/main.yml` templates `config.yaml` and
  `audit-policy.yaml`, `notify`ing `Restart rke2-server` in the role's
  handlers. The `template` module reports `changed` only on a real content
  diff, so the handler fires only then. Re-runs on a settled cluster do
  **not** restart the service.
- **Jinja2 template + externalized vars** —
  `roles/rke2_config/templates/config.yaml.j2` has no literals; every value
  comes from `group_vars/rke2_servers/main.yml`.
- **Calico via Helm** — `roles/calico_cni` installs Helm, adds the Tigera
  repo via `kubernetes.core.helm_repository`, deploys with `kubernetes.core.helm`.
- **Verify** — `roles/cluster_verify` waits for `Ready` node and all
  `calico-system` pods `Running`, then prints `kubectl get nodes -o wide`.
- **ansible-vault** — the join token lives in the AES256-encrypted
  `vault.yml`. The fetched kubeconfig is written locally at mode `0600`.

## Usage

```bash
# 1. install collections
ansible-galaxy collection install -r requirements.yml

# 2. rotate the vault password + token (demo values are placeholders!)
echo 'your-vault-password' > .vault_pass && chmod 600 .vault_pass
ansible-vault edit group_vars/rke2_servers/vault.yml

# 3. run
ansible-playbook site.yml

# 4. use the fetched kubeconfig
KUBECONFIG=fetched/rke2-rke2-node-1.yaml kubectl get nodes -o wide
```
