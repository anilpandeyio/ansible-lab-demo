# Remote Windows WSL development environment

This project uses Ansible on a Linux control machine to configure a remote
Windows VM with Ubuntu 24.04 on WSL2, Git, Docker CE, NVM, and Node.js LTS.
It can also configure Git SSH authentication, clone repositories, authenticate
to container registries, and download commonly used Docker images.

## 1. Architecture

```text
Linux Ansible control machine
  |
  | WinRM: Windows inventory account
  v
Remote Windows VM
  | Windows features, WSL engine, reboots, downloads
  |
  | wsl.exe -d Ubuntu-24.04 -u root --exec ...
  v
Ubuntu 24.04 / WSL2
  | root: apt packages, systemd, Docker service, Linux user
  |
  | runuser / wsl.exe -u developer
  v
developer
  | NVM, Node.js, Git configuration, repositories
  | Docker CLI, registry login, image pulls, smoke tests
```

Ansible does **not** connect to Ubuntu through SSH. It connects to Windows
through WinRM and uses `wsl.exe` to execute Linux commands. Ubuntu does not
need a separately configured SSH server or Ansible installation.

Docker Engine runs inside Ubuntu; Docker Desktop is not installed.

## 2. Files and how they relate

| File | Purpose | Used by |
|---|---|---|
| [inventory.ini](./inventory.ini) | Defines the Windows target, connection settings, and Windows account. | Ansible through `-i inventory.ini`. |
| [setup-wsl.yml](./setup-wsl.yml) | Main playbook: validates settings, prepares Windows, imports Ubuntu, runs the Linux script, and performs optional setup and verification. | `ansible-playbook`. |
| [templates/ubuntu-bootstrap.sh.j2](./templates/ubuntu-bootstrap.sh.j2) | Jinja2 Bash template for Ubuntu package installation, Docker, NVM, Node.js, and optional Git identity. | The playbook's `win_template` task. |
| [setup-vars.example.yml](./setup-vars.example.yml) | Commented example of optional overrides and credential references. | A human copies and edits it; it is not automatically loaded. |
| [inventory_bk.ini](./inventory_bk.ini) | Separate inventory file; not referenced by the playbook. | Only used if explicitly selected with `-i`. |

Optional files you create:

- `setup-vars.yml`: your non-secret settings, loaded using `-e @setup-vars.yml`.
- `secrets.yml`: encrypted Ansible Vault variables, explicitly loaded using
  `-e @secrets.yml --ask-vault-pass`.

```text
inventory.ini ----------------------> which Windows machines and account?
setup-vars.yml + secrets.yml -------> optional configuration overrides
                                      |
                                      v
                                setup-wsl.yml
                                      |
                                      | renders Jinja2 variables
                                      v
                     templates/ubuntu-bootstrap.sh.j2
                                      |
                                      | copies generated Bash to Windows
                                      v
             C:\ProgramData\AnsibleWSL\ubuntu-bootstrap.sh
                                      |
                                      | executes through wsl.exe as root
                                      v
                     Ubuntu packages and user environment
```

The generated script is temporary. Edit the source template, not the staged
copy. Extra variables supplied with `-e` override the playbook defaults.
Putting a variables file next to the playbook alone does not load it.

## 3. Prerequisites

### Control machine

- Linux with Ansible and the `ansible.windows` collection.
- Python WinRM support (`pywinrm`) in the Python environment used by Ansible.
- Network access to the Windows WinRM listener.

Inspect the existing installation:

```bash
ansible-playbook --version
ansible-galaxy collection list
python3 -c 'import winrm; print("pywinrm available")'
```

Install missing dependencies using the control machine's approved package
manager or Python environment; the playbook does not install its own
controller dependencies.

### Windows VM and hypervisor

- A Windows edition/build supporting WSL2; Windows Server 2022+ or a supported
  Windows 10/11 installation is the intended target.
- Nested virtualization exposed to the VM by the hypervisor.
- An administrator account with a password and permission to use `runas`.
- WinRM already enabled, reachable, and allowed by the firewall.
- Enough disk space for Ubuntu, downloaded packages, and container images.
- Internet access from both Windows and Ubuntu.

The build check in the playbook is a basic guard, not a complete Windows
support or nested-virtualization check.

The existing inventory uses NTLM over HTTP on port 5985. For production,
prefer an approved WinRM HTTPS configuration with certificate validation.
Do not expose WinRM broadly or commit inventory passwords to source control.
Configure corporate proxies and trusted CA certificates separately.

> The playbook may reboot Windows, restart the selected Ubuntu distribution,
> upgrade installed Linux packages, and grant root-equivalent Docker access.
> Run it during an appropriate maintenance window.

## 4. Execution workflow

| Phase | What happens | Why |
|---|---|---|
| 1. Connectivity | Runs `win_ping` using the inventory. | Confirms the controller can authenticate to WinRM. |
| 2. Validation | Checks usernames, versions, repository/registry inputs, Windows build, administrator rights, and CPU architecture. | Fails early on invalid configuration. |
| 3. Windows features | Enables WSL and VirtualMachinePlatform; reboots when required. | Provides the Windows virtualization components. |
| 4. WSL engine | Runs `wsl --update --web-download`, handles reboot code 3010, sets WSL2 as the default, then checks `wsl --version`. | Upgrades older inbox WSL before using modern version commands. |
| 5. Ubuntu import | Checks whether the named distribution already exists for the account. If absent, reads Microsoft's WSL catalog, selects the official Ubuntu 24.04 image for the CPU, verifies SHA256, and imports it as WSL2. | Avoids interactive Store/first-launch user setup. |
| 6. Linux configuration | Checks the registered WSL version before attempting conversion, verifies Ubuntu 24.04, creates `developer`, and updates `/etc/wsl.conf`. | Keeps the distro on WSL2 and enables systemd and the default Linux user. |
| 7. Distro restart | Terminates only the selected distribution if its WSL configuration changed. | Applies systemd and default-user settings. |
| 8. Template execution | Renders and stages the Bash template, converts its Windows path with `wslpath`, then executes it as Ubuntu root. | Installs the Linux environment without another transport. |
| 9. Optional access | Configures Git SSH access and container registry logins when supplied. | Makes authentication available before private repository clones/image pulls. |
| 10. Repositories/images | Clones missing repositories and pulls the configured images as `developer`. | Creates the user's working environment and image cache. |
| 11. Verification | Runs Docker, Compose, Buildx, Node.js, npm, and Git checks as the non-root Linux user. | Verifies actual use rather than package presence alone. |
| 12. Cleanup | Prints the environment summary and removes the downloaded Ubuntu archive and staged script after successful provisioning. | Keeps temporary installation files out of the final setup. |

### What the Linux template does

The [Bash template](./templates/ubuntu-bootstrap.sh.j2):

1. Confirms systemd is PID 1 and waits for its management interface.
2. Runs non-interactive `apt-get update` and `apt-get upgrade`.
3. Installs Git, curl, certificates, OpenSSH client, build tools, and Python.
4. Rejects conflicting container runtime packages instead of removing them.
5. Adds Docker's official signing key and Ubuntu apt repository.
6. Installs Docker CE, CLI, containerd, Buildx, and Compose.
7. Enables and starts Docker, and adds the Linux user to the `docker` group.
8. Switches to the Linux user, installs the selected NVM Git tag, and adds
   an NVM initialization block to the user's Bash configuration.
9. Installs the requested Node.js version and sets the default NVM alias.
10. Configures Git author name/email if provided, then checks Docker is active.

NVM is installed as `developer`, not root, because NVM and Node installations
are per-user. Non-interactive checks explicitly source NVM rather than assuming
an interactive shell has loaded it.

### Accounts and permissions

There are three distinct execution contexts:

| Context | Role |
|---|---|
| Windows inventory account | Owns the WSL registration and performs Windows administration. |
| Ubuntu `root` | Installs packages and changes Linux system settings. |
| Ubuntu `developer` | Owns NVM, Node.js, repositories, Git configuration, and Docker credentials. |

The playbook uses `become_method: runas` with the same Windows account and its
password to obtain a full logon context rather than relying on the restricted
WinRM token. This is separate from Linux sudo.

**WSL distributions are registered per Windows user.** Run commands using the
same Windows account as the inventory. Another account may see no distribution.

## 5. Run commands

Run these commands from this project directory on the Linux control machine.

### Basic setup

```bash
ansible -i inventory.ini windows -m ansible.windows.win_ping
ansible-playbook -i inventory.ini setup-wsl.yml --syntax-check
ansible-playbook -i inventory.ini setup-wsl.yml
```

The basic setup does not clone application repositories or configure credentials:
those lists/values are empty by default.

### Customized setup

```bash
cp setup-vars.example.yml setup-vars.yml
# Edit setup-vars.yml, then:
ansible-playbook -i inventory.ini setup-wsl.yml -e @setup-vars.yml
```

For encrypted secrets:

```bash
ansible-vault create secrets.yml
ansible-playbook -i inventory.ini setup-wsl.yml \
  -e @setup-vars.yml -e @secrets.yml --ask-vault-pass
```

Ansible Vault protects secrets at rest on the controller, not after they have
been installed for use on the target. Exclude local unencrypted configuration
and credentials from source control.

### Common configuration variables

| Variable | Default | Meaning |
|---|---|---|
| `wsl_distribution` | `Ubuntu-24.04` | WSL registration name; the selected OS image remains Ubuntu 24.04. |
| `wsl_linux_user` | `developer` | Linux user for development tools and Docker commands. |
| `wsl_install_path` | `C:\WSL\Ubuntu-24.04` | Location used when initially importing Ubuntu. |
| `wsl_staging_path` | `C:\ProgramData\AnsibleWSL` | Temporary download/script directory. |
| `nvm_version` | `v0.40.8` | NVM Git tag. |
| `node_version` | `lts/*` | Latest LTS resolved by NVM; use an exact version for repeatability. |
| `git_user_name`, `git_user_email` | Empty | Optional Git commit identity, not authentication. |
| `git_ssh_private_key`, `git_ssh_known_hosts` | Empty | Optional dedicated SSH key and trusted host-key entries. |
| `required_repositories` | `[]` | Repository entries with `name` and `url`. |
| `container_registries` | `[]` | Registry entries with `registry`, `username`, and `password`. |
| `common_docker_images` | hello-world, Ubuntu 24.04, Alpine | Images to pull; see the example for exact tags. |

Example non-secret settings for `setup-vars.yml`:

```yaml
wsl_linux_user: developer
node_version: "lts/*"
git_user_name: "Your Name"
git_user_email: "you@example.com"

git_ssh_private_key: "{{ vault_git_ssh_private_key }}"
git_ssh_known_hosts: "{{ vault_git_ssh_known_hosts }}"

required_repositories:
  - name: application
    url: git@github.com:YOUR-ORG/YOUR-REPO.git

container_registries:
  - registry: ghcr.io
    username: YOUR-GITHUB-USERNAME
    password: "{{ vault_ghcr_token }}"
  - registry: artifactory.example.com
    username: YOUR-ARTIFACTORY-USERNAME
    password: "{{ vault_artifactory_token }}"

common_docker_images:
  - hello-world:latest
  - ubuntu:24.04
  - alpine:latest
  - ghcr.io/YOUR-ORG/YOUR-IMAGE:YOUR-TAG
```

Define the referenced `vault_*` variables inside the encrypted secrets file.
Use a dedicated, unencrypted automation SSH key with minimal repository access,
and obtain SSH known-host entries from a trusted source. The automation does not
configure a passphrase agent or an HTTPS Git credential helper.

Git SSH setup writes dedicated key/known-hosts files and sets the user's global
`core.sshCommand` to use them with strict host-key checking. This affects all
Git SSH operations for that Linux user.

Registry authentication uses `docker login --password-stdin` as the Linux user.
Secret-handling tasks have `no_log: true`. Without a separately configured Docker
credential helper, Docker stores reversible credentials in the user's Docker
configuration; Vault does not encrypt that target-side file.

## 6. Using Ubuntu and sudo

From Windows PowerShell, under the Windows account that owns the distribution:

```powershell
wsl -d Ubuntu-24.04 -u developer
```

The automation creates `developer` **without a password and without sudo group
membership**. There is no default sudo password, and the Windows password is not
automatically a Linux password.

To grant password-based sudo access manually:

```powershell
wsl -d Ubuntu-24.04 -u root
```

Inside that root Linux shell:

```bash
passwd developer
usermod -aG sudo developer
exit
```

Open a new developer shell, then test:

```bash
sudo whoami
# Expected: root; enter the Linux password set with passwd.
```

This manual sudo change is not part of the current playbook. Rerunning the
playbook does not intentionally remove it or reset the password.

Docker group membership already allows running Docker without sudo and grants
root-equivalent control inside Ubuntu. Only grant it to trusted users.

## 7. Verification and rerun behavior

The automated smoke test confirms:

- Commands run as a non-root Linux user.
- Docker's API responds and `hello-world` runs successfully.
- Compose and Buildx commands work.
- NVM loads its default Node.js, and Node.js/npm report versions.
- Node.js executes a small JavaScript check.
- Git reports its version.

For interactive checks inside Ubuntu:

```bash
whoami
cat /etc/os-release
docker info
docker run --rm hello-world
docker compose version
docker buildx version
source "$HOME/.nvm/nvm.sh"
nvm --version
node --version
npm --version
git --version
```

The remote provisioning run completed with `failed=0` and `unreachable=0`.
Docker/Node smoke tests passed. Optional repository and credential tasks were
skipped because their configuration was empty. These paths are available but
were not validated against your private services.

### Safe to rerun does not mean unchanged

- Existing Ubuntu registration is reused; the image is not reimported.
- WSL2 conversion is skipped when the registry already records version 2.
- Existing WSL configuration sections are retained when setting systemd/user
  options, although formatting/comments can change.
- Windows features are only changed when needed.
- WSL updates and Ubuntu package upgrades are attempted again.
- NVM is checked out at the configured tag; a modified tracked NVM checkout
  causes failure instead of being silently overwritten.
- `lts/*` and floating image tags can resolve to newer versions.
- Existing application repositories are checked for matching origin, but are
  not pulled, reset, or switched to another branch.
- Supplied Git SSH files/configuration and registry logins are applied again.
- Installation command tasks can report `changed` even when versions stay equal;
  this is not a fully change-free/idempotent run.

`systemctl enable docker` starts Docker when Ubuntu/systemd starts. It does
**not** create a Windows startup task or guarantee WSL remains running after
the last Linux command exits.

## 8. Troubleshooting

| Symptom | Explanation and next step |
|---|---|
| WinRM connection/authentication fails | Check the inventory account, listener, firewall, routing, and transport. `win_ping` must succeed first. |
| `--install --no-distribution` is unrecognized | Older inbox WSL does not support this bootstrap option. The current playbook uses `--update --web-download` before checking `--version`. |
| WSL2 cannot start | Verify nested virtualization, enabled Windows features, pending reboot, and Windows/WSL support. Windows feature installation alone cannot enable virtualization on the hypervisor. |
| `--set-version` reports the distro is already WSL2 and returns an error | The current playbook checks the per-user registry version first and avoids unnecessary conversion. |
| WSL reports a systemd user-session timeout | Cold systemd startup can be slow. The package script waits for systemd's interface; inspect `journalctl -b` if the problem persists. The waits do not repair a genuinely broken boot. |
| Docker socket is missing after opening WSL | Docker may still be starting. Image pulls and smoke tests wait for its API. Inspect `systemctl status docker` and `journalctl -u docker -b` if it remains unavailable. |
| `node` or `nvm` not found | Confirm you are `developer`; source `$HOME/.nvm/nvm.sh`. Root has a different home and tool configuration. |
| `sudo` rejects the user/password | Follow the explicit password and sudo-group setup above, then open a fresh shell. |
| Private Git clone fails | Check repository permissions, key format, trusted host-key entries, and whether SSH authentication is configured. Git author name/email is not a credential. |
| Private image pull fails | Check registry hostname, exact image path/tag, token permissions, and corporate CA/proxy setup. |
| Conflicting Docker package detected | Review the named existing runtime/package manually; the script deliberately does not uninstall it. |
| Ubuntu does not appear in `wsl --list --verbose` | Check which Windows account is running the command; WSL registration is per user. |

Long-running tasks use Ansible `async` with polling, so `ASYNC POLL` output is
normal. Most WSL/import/image tasks allow 30 minutes; Linux provisioning allows
60 minutes. The systemd/Docker readiness loops use up to 60 two-second sleeps,
in addition to command execution time.

On failure, read the first failed task and its native exit code/output. Fix
the underlying problem and rerun the full playbook. Do not ignore errors to
produce a successful-looking recap.

Successful cleanup removes the staged script and root filesystem archive.
Because cleanup is a normal post-task rather than an unconditional failure
handler, a failed run may leave those installation files for diagnosis/retry.
It never removes the installed Ubuntu distribution or container images.
