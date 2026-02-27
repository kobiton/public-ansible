# Kobiton Public Ansible

Ansible code shared with the public. Includes playbooks for setting up macOS hosts, installing deviceConnect on macOS hosts, and installing gigacap on Ubuntu hosts.

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) (with `ansible-playbook` available on your PATH)
- [sshpass](https://sourceforge.net/projects/sshpass/) — the inventory is configured for password-based SSH authentication, which requires `sshpass` on the Ansible controller
- SSH access to all target hosts
- `become` (sudo) privileges on each target host — the inventory configures `ansible_become_password` to use the same password as SSH
- The deviceConnect `.pkg` or gigacap `.deb` installer file downloaded locally

For development:

- [ansible-lint](https://ansible.readthedocs.io/projects/lint/) (`pip install ansible-lint`)

## Getting Started

Get the playbooks onto your Ansible controller by either cloning the repository:

```bash
git clone https://github.com/kobiton/public-ansible.git
cd public-ansible
```

Or by downloading and extracting a ZIP of the repository using the **Code > Download ZIP** button on GitHub.

## Project Structure

```
.
├── .github/workflows/ci.yml   # CI: runs lint on PRs and pushes to main
├── install-deviceconnect.yml   # Playbook: install deviceConnect on macOS
├── install-gigacap.yml         # Playbook: install gigacap on Ubuntu hosts
├── inventory.yml.example       # Example inventory (copy to inventory.yml)
├── Makefile                    # Lint target
└── setup-mac.yml               # Playbook: set up macOS hosts (Corretto)
```

## Setting Up the Inventory

The playbooks expect an inventory file at `inventory.yml` with two host groups: `mac` (macOS hosts) and `gem` (Ubuntu hosts). This file is gitignored since it contains environment-specific values. An example is provided to get you started.

1. Copy the example inventory:

   ```bash
   cp inventory.yml.example inventory.yml
   ```

   Or create your own `inventory.yml` from scratch — just ensure it defines `mac` and `gem` groups.

2. Open `inventory.yml` in your editor. The file has the following structure:

   ```yaml
   all:
     children:
       mac:
         hosts:
           standalone-mac-01:
             ansible_host: 10.0.1.10
           standalone-mac-02:
             ansible_host: 10.0.1.11
         vars:
           ansible_user: your-mac-username  # Provided by your Kobiton contact
           ansible_password: "{{ mac_password }}"
           ansible_become_password: "{{ mac_password }}"
       gem:
         hosts:
           standalone-gem-01:
             ansible_host: 10.0.2.10
           standalone-gem-02:
             ansible_host: 10.0.2.11
         vars:
           ansible_user: your-gem-username  # Provided by your Kobiton contact
           ansible_password: "{{ gem_password }}"
           ansible_become_password: "{{ gem_password }}"
   ```

3. Update the hosts under each group to match your environment. For each host, set `ansible_host` to its IP address or hostname. Add or remove host entries as needed.

4. If needed, update the `ansible_user` under each group's `vars` section to match the SSH user for that group.

5. The SSH passwords are referenced as variables (`mac_password` and `gem_password`). Supply them at runtime using one of:

   - **Ansible Vault** (recommended) — create an encrypted vars file:

     ```bash
     mkdir -p group_vars/all
     ansible-vault create group_vars/all/vault.yml
     ```

     Add the passwords inside:

     ```yaml
     mac_password: "your-mac-password"
     gem_password: "your-gem-password"
     ```

     Then pass `--ask-vault-pass` when running playbooks.

   - **Command-line variables** — pass them directly with `-e`:

     ```bash
     -e mac_password="your-mac-password"
     ```

## Setting Up macOS Hosts

The `setup-mac.yml` playbook sets up macOS hosts in the `mac` group. Currently it installs Amazon Corretto (JDK). It compares the version from the package filename against the currently installed version and skips hosts that are already up to date. It also validates that the package architecture (aarch64 or x64) matches the target host.

**Package filename format:** `amazon-corretto-<version>-macosx-<arch>.pkg` (e.g., `amazon-corretto-21.0.10.7.1-macosx-aarch64.pkg`)

1. Place the Corretto `.pkg` file on your local machine (the Ansible controller).

2. Run the playbook, passing the path to the `.pkg` file.

   Using Ansible Vault:

   ```bash
   ansible-playbook setup-mac.yml \
     -i inventory.yml \
     -e corretto_pkg=/path/to/amazon-corretto-21.0.10.7.1-macosx-aarch64.pkg \
     --ask-vault-pass
   ```

   Using command-line variables:

   ```bash
   ansible-playbook setup-mac.yml \
     -i inventory.yml \
     -e corretto_pkg=/path/to/amazon-corretto-21.0.10.7.1-macosx-aarch64.pkg \
     -e mac_password="your-password"
   ```

3. Review the output. For each host, the playbook will print a version comparison showing whether it will install or skip.

## Installing deviceConnect

The `install-deviceconnect.yml` playbook installs a deviceConnect `.pkg` on all hosts in the `mac` group. It compares the version from the package filename against the currently installed version and skips hosts that are already up to date.

**Package filename format:** `deviceConnect.<version>.pkg` — the `<version>` portion must exactly match the output of `dc-services --version` (e.g., `4.23.0.202501010000.production@abc1234def`).

1. Place the deviceConnect `.pkg` file on your local machine (the Ansible controller).

2. Run the playbook, passing the path to the `.pkg` file.

   Using Ansible Vault:

   ```bash
   ansible-playbook install-deviceconnect.yml \
     -i inventory.yml \
     -e pkg_file=/path/to/deviceConnect.<version>.pkg \
     --ask-vault-pass
   ```

   Using command-line variables:

   ```bash
   ansible-playbook install-deviceconnect.yml \
     -i inventory.yml \
     -e pkg_file=/path/to/deviceConnect.<version>.pkg \
     -e mac_password="your-password"
   ```

3. Review the output. For each host, the playbook will print a version comparison showing whether it will install or skip.

## Installing gigacap

The `install-gigacap.yml` playbook installs a gigacap `.deb` package on all hosts in the `gem` group. It runs one host at a time (`serial: 1`) to limit the blast radius of a failed installation — if one host fails, the remaining hosts are skipped. It restarts the `gigacap` systemd service after a successful install.

**Package filename format:** `gigacap_<version>+<commit>_<arch>.deb` — the `<version>+<commit>` portion is compared against the installed version, which is parsed from `gigacap version` output (e.g., `4.22.0+abc1234`).

1. Place the gigacap `.deb` file on your local machine (the Ansible controller).

2. Run the playbook, passing the path to the `.deb` file.

   Using Ansible Vault:

   ```bash
   ansible-playbook install-gigacap.yml \
     -i inventory.yml \
     -e deb_file=/path/to/gigacap_<version>+<commit>_<arch>.deb \
     --ask-vault-pass
   ```

   Using command-line variables:

   ```bash
   ansible-playbook install-gigacap.yml \
     -i inventory.yml \
     -e deb_file=/path/to/gigacap_<version>+<commit>_<arch>.deb \
     -e gem_password="your-password"
   ```

3. Review the output. For each host, the playbook will print a version comparison showing whether it will install or skip. If a new version is installed, the gigacap service is automatically restarted.

## Failure Recovery

All playbooks are safe to re-run. If a run fails mid-way, the installer file may be left in `/tmp` on the target host. A subsequent successful run will clean it up.

## Linting

Run ansible-lint against the playbooks:

```bash
make lint
```

This requires [ansible-lint](https://ansible.readthedocs.io/projects/lint/) to be installed (`pip install ansible-lint`).

Linting also runs automatically in CI on pull requests and pushes to `main`.
