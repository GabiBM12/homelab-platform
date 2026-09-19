# Management Laptop Bootstrap

This directory is the first executable part of the homelab. It bootstraps a
fresh Ubuntu Server management laptop from a trusted administration laptop.
After bootstrap, the management laptop hosts the GitHub Actions self-hosted
runner and executes Terraform, Packer, and Ansible against Proxmox and the lab
VMs.

The bootstrap does **not** require the management laptop to accept connections
from the Internet. The administration laptop connects over the home LAN, and
the GitHub runner connects outbound to GitHub over HTTPS.

## Bootstrap topology

```text
Administration laptop                    Management laptop
Ansible control node                     Ubuntu Server
<ADMIN_LAPTOP_IP>                        <MANAGEMENT_LAPTOP_IP>
        |                                         |
        +------------- SSH over LAN --------------+
                                                  |
                                                  +-- outbound HTTPS --> GitHub
                                                  +-- HTTPS -----------> Proxmox API
                                                  +-- SSH -------------> lab VMs
```

No home-router port forwarding is required. Do not expose SSH on the
management laptop to the public Internet.

## Responsibilities

The administration laptop is the recovery and bootstrap control node. Keep a
working copy of this repository and the bootstrap SSH key there.

The management laptop provides:

- a dedicated, unprivileged GitHub Actions runner account;
- Git, Python, Ansible, Terraform, and Packer;
- SSH clients and validation utilities;
- network access to the Proxmox API and, later, private lab networks;
- a stable execution environment for image, infrastructure, and configuration
  pipelines.

The management laptop should not host normal application workloads. It should
also not be the only place where Terraform state, source code, or recovery
credentials exist.

## Prerequisites

### Network and home router

Before running Ansible:

1. Reserve addresses on the home router:
   - administration laptop: `<ADMIN_LAPTOP_IP>`;
   - management laptop: `<MANAGEMENT_LAPTOP_IP>`;
   - Proxmox: `<PROXMOX_HOST_IP>`;
   - future `router01` WAN: `<ROUTER01_WAN_IP>`.
2. Confirm both laptops are on your home LAN, `<HOME_LAN_CIDR>`, and can
   reach each other.
3. Do not configure a public port forward for TCP 22.
4. Disable UPnP on the home router if it is not required.

Replace the inventory placeholders before running the playbook. Documentation
uses `<NAME>` placeholders; YAML uses `REPLACE_WITH_...` strings. Neither is
automatically substituted. IP placeholders mean a host address without a prefix;
CIDR placeholders mean a network address including its prefix length.

### Management laptop

Install a supported Ubuntu Server release and complete only the minimum manual
bootstrap:

1. Create a non-root administration account with `sudo` access.
2. Install Python and OpenSSH if the Ubuntu installer did not include them:

   ```bash
   sudo apt update
   sudo apt install openssh-server python3 python3-apt
   sudo systemctl enable --now ssh
   ```

3. Add the administration laptop's SSH public key to that account.
4. Assign or reserve `<MANAGEMENT_LAPTOP_IP>` on your home LAN.
5. Verify the laptop has working DNS and outbound HTTPS access.
6. Test an interactive SSH connection from the administration laptop once so
   that its host key is recorded in `known_hosts`.

The playbook intentionally does not create the first administrator or install
the first SSH key. Losing either during remote bootstrap would make recovery
needlessly difficult.

### Administration laptop

Required locally:

- Git;
- Python 3;
- Ansible Core 2.16 or newer;
- an SSH private key corresponding to the public key installed above;
- permission to install the collection listed in `ansible/requirements.yml`.

On Ubuntu/Debian, Ansible can be installed using the distribution package or a
Python virtual environment. On macOS, Homebrew or a Python virtual environment
can be used. Keep Ansible isolated from the system Python where practical.

Verify the installation:

```bash
ansible --version
ansible-playbook --version
```

Enter the bootstrap Ansible directory so its `ansible.cfg` and role path are
automatically discovered, then install the required collection:

```bash
cd bootstrap/management/ansible
ansible-galaxy collection install -r requirements.yml
```

## Configuration before the first run

Edit:

```text
bootstrap/management/ansible/inventory/bootstrap/hosts.yml
bootstrap/management/ansible/inventory/bootstrap/group_vars/all.yml
```

At minimum, replace:

- `REPLACE_WITH_MANAGEMENT_LAPTOP_IP` in `ansible_host` with the management
  laptop's reserved address;
- `ansible_user` with the Ubuntu administrator created during installation;
- `REPLACE_WITH_ADMIN_LAPTOP_IP` in `management_ssh_allowed_cidr` with the
  administration laptop's reserved IPv4 address, keeping the `/32` suffix;
- `github_runner_url` with the private repository or organization URL;
- the runner version and SHA-256 checksum shown by GitHub.

Do not put passwords, private keys, GitHub tokens, or Proxmox tokens in these
files.

The `common` role also has a placeholder default for the SSH source. Set the
actual value in `group_vars/all.yml`, which overrides that default. The existing
source-address assertion rejects an unreplaced placeholder; do not bypass it.

## Phase 1: verify access

Run all remaining commands from `bootstrap/management/ansible`:

```bash
ansible management -m ansible.builtin.ping
```

If privilege escalation requires a password, confirm it works:

```bash
ansible management \
  -b \
  -K \
  -m ansible.builtin.command \
  -a 'id'
```

## Phase 2: bootstrap and harden the management laptop

Validate the playbook syntax first:

```bash
ansible-playbook \
  playbooks/bootstrap-management.yml \
  --syntax-check
```

On a completely fresh host, check mode cannot reliably simulate adding the
external HashiCorp repository and then installing packages from it. Use
`--check --diff` for later runs, after the first bootstrap has completed.

Apply them:

```bash
ansible-playbook \
  playbooks/bootstrap-management.yml \
  --diff \
  --ask-become-pass
```

This playbook:

- installs base administration and security packages;
- enables automatic security updates;
- disables SSH password and root authentication;
- enables UFW with default-deny incoming traffic;
- allows SSH only from `management_ssh_allowed_cidr`;
- installs Ansible, Terraform, Packer, and validation tools;
- optionally installs Docker, but does not enable it by default.

The SSH allow rule is created before UFW is enabled. Keep the original SSH
session open and test a second connection before disconnecting.

## Phase 3: register the GitHub Actions runner

Use a private repository. In GitHub, open:

```text
Repository or organization Settings
  -> Actions
  -> Runners
  -> New self-hosted runner
  -> Linux
```

GitHub displays the current runner version, download checksum, and a short-lived
registration token. Put the version and checksum in `group_vars/all.yml`.

Run the registration playbook:

```bash
ansible-playbook \
  playbooks/register-github-runner.yml \
  --ask-become-pass
```

The playbook prompts for the registration token without echoing it. The token
is marked `no_log` and is never written to the repository. The runner is
installed as the unprivileged `github-runner` user and as a systemd service.

The runner makes an outbound HTTPS connection to GitHub. GitHub does not SSH
into the laptop.

## Phase 4: verify the result

```bash
ansible-playbook \
  playbooks/verify-management.yml
```

Also confirm in GitHub that the runner is online, then run a harmless manual
workflow that prints tool versions. Do not give the first test workflow
infrastructure secrets.

## Security model

### Network access

The desired initial policy is:

| Source | Destination | Access |
|---|---|---|
| Administration laptop | Management TCP 22 | Allow |
| Other home devices | Management TCP 22 | Deny |
| Internet | Management inbound | Deny |
| Management | GitHub TCP 443 | Allow outbound |
| Management | Proxmox API TCP 8006 | Allow |
| Management | Future lab SSH TCP 22 | Allow after routing exists |

Proxmox UI/API access is direct over the home LAN. The private lab route is not
created during this bootstrap because `router01` does not exist yet.

After `router01` has been deployed and configured, add the route:

```text
<LAB_NETWORK_CIDR> via <ROUTER01_WAN_IP>
```

That route should be implemented in a separate, OS-network-specific playbook
and applied from the administration laptop. Verify `router01` forwarding and
firewall rules before making it persistent.

### Runner trust boundary

A workflow running on a self-hosted runner can execute code on the management
laptop. Therefore:

- use a private repository;
- never run untrusted pull-request code on this runner;
- use GitHub-hosted runners for formatting, linting, and PR validation;
- restrict the runner to selected repositories or workflows;
- deploy only from a protected branch or a manually approved environment;
- grant the workflow `contents: read` unless more is explicitly needed;
- pin third-party actions to full commit SHAs;
- do not give the runner user passwordless `sudo`;
- do not add the runner user to the Docker group unless container builds are
  required—the Docker socket is effectively root-equivalent;
- use a restricted Proxmox API token instead of a root password;
- keep Terraform state in a backed-up remote backend, not only on the runner.

The management bootstrap playbook should remain manually executable from the
administration laptop. Do not make the management runner solely responsible for
repairing itself.

## What comes next

Once this bootstrap is complete:

1. Create a least-privilege Proxmox automation account and API token.
2. Create the internal `vmbr10` bridge as part of the Proxmox bootstrap.
3. Build the first golden image with Packer.
4. Deploy `router01` with Terraform.
5. Configure forwarding, NAT, and firewall rules on `router01` with Ansible.
6. Add the private lab route to the management laptop.
7. Deploy workload VMs with Terraform.
8. Configure the workload VMs with Ansible.

Initially, creating `vmbr10` through the Proxmox UI is reasonable. Automating
the host's management networking before a console recovery path exists can
disconnect Proxmox.

## Recovery

If the runner is offline or damaged:

1. Connect from the administration laptop over the home LAN.
2. Re-run `bootstrap-management.yml`.
3. Inspect the runner service under `/opt/actions-runner`.
4. If re-registration is necessary, remove the stale runner entry in GitHub and
   run `register-github-runner.yml` with a new registration token.

The playbooks are idempotent and are intended to be safe to run repeatedly.

## Upstream references

- [GitHub self-hosted runner communication requirements](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
- [GitHub secure use guidance](https://docs.github.com/en/actions/reference/security/secure-use)
- [GitHub runner access controls](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/manage-access)
- [HashiCorp's official Terraform package installation](https://developer.hashicorp.com/terraform/install)
