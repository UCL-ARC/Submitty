# Running the Submitty development environment with Docker

As well as VirtualBox, Parallels, VMware, libvirt and QEMU, the Vagrantfile supports
Vagrant's `docker` provider. With this provider, the development environment runs in a
Docker container on your machine instead of in a virtual machine.

> [!WARNING]
> The container runs **privileged** and is far less isolated from your machine than a
> virtual machine. Use it only for local development on a machine you control. Read
> [Security warnings](#security-warnings) before you use it.

## How it works

Submitty's install scripts expect a full Ubuntu machine. They manage services with
`systemctl`, Vagrant connects to the machine over SSH, and the autograder runs Docker
itself to grade submissions. So the container is set up to behave like a VM:

- The image is built from [`.setup/vagrant/Dockerfile`](.setup/vagrant/Dockerfile). It is
  Ubuntu 22.04 with systemd running as PID 1, an SSH server, and a `vagrant` user.
- The first `vagrant up` runs the full Submitty install (`.setup/vagrant/setup_vagrant.sh`)
  inside the container. Unlike the VirtualBox provider, the Docker provider has no prebuilt
  box to download.
- The autograder's Docker runs **inside** the container ("Docker-in-Docker"). Its images and
  containers are kept on Docker volumes mounted at `/var/lib/docker` and
  `/var/lib/containerd`.
- Your Submitty checkout, and any of the optional sibling repositories next to it (Lichen,
  RainbowGrades, SubmittyCLI, etc.), are bind-mounted into
  `/usr/local/submitty/GIT_CHECKOUT/`, the same as on the VM providers.

## Requirements

- **Linux host.** This setup has been tested on Linux only. On macOS and Windows, Docker
  Desktop runs containers inside its own VM, and whether systemd and Docker-in-Docker work
  there depends on your Docker Desktop version.
- **Docker Engine**, running, with your user able to use it (in the `docker` group).
- **Vagrant** 2.4 or newer. The docker provider is built in; no plugin is needed.
- **A host user with uid and gid 1000** (check with `id -u` and `id -g`). Bind mounts keep
  your host file ownership, and the container's `vagrant` user is uid/gid 1000, so the
  files show up inside the container as owned by `vagrant`, which is what the install
  expects. With a different uid or gid, expect permission errors.
- **Disk space:** at least 30 GB free. The install, the sample courses and the autograding
  images (gcc, clang, python, java, jupyter, pdflatex, rust and others) need a lot of
  space, and Docker stores them on your host's disk.
- **Free ports** 1511, 8443, 16442, 7001 and 2222 on the host (they can be changed; see
  [Ports](#ports)).

Repository layout: the Vagrantfile must sit in a complete Submitty checkout, including the
`.setup/` directory. If you keep optional repositories, put them next to `Submitty/` (for
example `~/software/Submitty/Submitty` and `~/software/Submitty/Lichen`).

## Running it

From the Submitty repository:

```bash
vagrant up --provider=docker
```

The first run builds the image, starts the container, and installs Submitty. Expect it to
take an hour or more, depending on your network and CPU. When it finishes you will see
`INSTALLATION SUCCESS!` in the output, and Submitty is at <http://localhost:1511>.

The full install log is also written to `.vagrant/logs/ubuntu-22.04.log` in the repository.

To avoid passing `--provider=docker` every time, set it as your default:

```bash
export VAGRANT_DEFAULT_PROVIDER=docker
```

Once the container has been created, other commands (`vagrant ssh`, `vagrant provision`,
`vagrant halt`, ...) find it automatically.

### Resources

The container is limited to 4096 MB of memory and 4 CPUs by default. Change them with the
same variables as for the VMs:

```bash
VM_MEMORY=8192 VM_CPUS=6 vagrant up --provider=docker
```

These limits are set when the container is created, so changing them later needs
`vagrant destroy` and `vagrant up` again.

The other environment variables from the top of the Vagrantfile also work, for example
`NO_SUBMISSIONS=1` to skip generating sample submissions, or `EXTRA=rpi`.

### Ports

| Service              | Container port | Host port | Variable to change it |
| -------------------- | -------------- | --------- | --------------------- |
| Submitty website     | 1511           | 1511      | `VM_PORT_SITE`        |
| WebSocket server     | 8443           | 8443      | `VM_PORT_WS`          |
| PostgreSQL           | 5432           | 16442     | `VM_PORT_DB`          |
| SAML test IdP        | 7000           | 7001      | `VM_PORT_SAML`        |
| SSH                  | 22             | 2222      | `VM_PORT_SSH`         |

Docker publishes these ports on all of your host's network interfaces, not just
`localhost`. See [Security warnings](#security-warnings).

## Day-to-day commands

| Task                                    | Command                                   |
| --------------------------------------- | ----------------------------------------- |
| Open a shell in the container (as root) | `vagrant ssh`                             |
| Stop the container                      | `vagrant halt`                            |
| Start it again                          | `vagrant up`                              |
| Check its status                        | `vagrant status`                          |
| Re-run the Submitty install             | `vagrant provision` (answer `y` when asked) |
| Delete the container                    | `vagrant destroy`                         |

`vagrant ssh` logs you in as `root`, as on the VM providers. The usual Submitty aliases
(`submitty_help`, `submitty_install`, `submitty_restart_services`, `migrator`, ...) are set
up for root.

You can also reach the container directly with Docker; it is named `submitty`:

```bash
docker exec -it submitty bash
```

`vagrant provision` resets the Submitty install and reinstalls it from the start, as it
does on the VMs.

### Starting over

`vagrant destroy` removes the container. To pick up changes to the Dockerfile, destroy and
recreate it:

```bash
vagrant destroy -f
vagrant up --provider=docker
```

`vagrant destroy` also deletes the container's volumes, so the autograder's images and
containers (several GB) are removed with it and downloaded again on the next install. The
image Vagrant built is kept; remove old ones with `docker image prune` if you rebuild often.

## Security warnings

This setup trades isolation for convenience. Only use it for local development, on a
machine you control, and never for a real Submitty server or real student data.

- **The container is privileged.** It runs with `--privileged`, which gives it every Linux
  capability and access to all of the host's devices. Root inside the container can
  effectively become root on your machine: load kernel modules, mount host disks, change
  host network settings, and so on. Treat root in the container as root on your host.
  It is privileged because systemd and Docker-in-Docker need it.

- **It shares the host's cgroups.** The container is started with `--cgroupns=host` and
  mounts `/sys/fs/cgroup` read-write, so processes in it can see and change the resource
  controls of the host and of your other containers.

- **The autograder sandbox is weaker.** Autograding runs student code in Docker containers
  nested inside this privileged container. Those inner containers are not as well
  isolated as on a real Submitty server or a VM, so untrusted code that escapes its
  grading container ends up in a privileged container, one step away from your host.
  Do not grade code you don't trust.

- **Your files are shared read-write.** Your Submitty checkout and the sibling repositories
  are bind-mounted straight into the container. Anything that runs as root, or as a
  Submitty user, inside the container can change or delete your real files on the host,
  including the `.git` directories. Commit or push work you care about.

- **Default passwords and keys.** The image sets the passwords of `root` and `vagrant` to
  `vagrant`, allows root to log in over SSH with a password, and starts with Vagrant's
  public "insecure" key (Vagrant replaces it with a new key pair the first time it
  connects). The sample Submitty data and databases also use well-known development
  passwords.

- **Ports are open on your network.** Docker publishes the ports in the table above on all
  of your host's network interfaces (`0.0.0.0`), and Docker's port publishing bypasses
  host firewalls such as `ufw`. Anyone who can reach your machine on the network can
  reach SSH (with the default passwords), PostgreSQL and the website. Use this only on a
  trusted network, or block those ports with rules in Docker's `DOCKER-USER` iptables
  chain.

- **Git trusts every directory.** Because the shared repositories are owned by your host
  user rather than root, the image sets git's `safe.directory` to `*` for all users inside
  the container. That turns off git's protection against running hooks and config from a
  repository owned by someone else.

- **Images are fetched from the internet.** Building the image downloads packages from
  Ubuntu, Vagrant's public key from GitHub, and, during the install, packages and Docker
  images from several third parties (packages.sury.org, Docker Hub, NodeSource and others).

## Troubleshooting

- **`Cannot connect to the Docker daemon`**: start Docker (`sudo systemctl start docker`)
  and check that your user can run `docker ps`.

- **The install fails with `INSTALLATION FAILURE`**: look further up the output, or in
  `.vagrant/logs/ubuntu-22.04.log`, for the first error. Download errors (such as `503
  Service Unavailable`) are usually temporary; run `vagrant provision` again later.

- **`no space left on device`**: free space on the host, for example with
  `docker image prune` or `docker builder prune`.

- **`detected dubious ownership in repository`**: your container was built from an older
  version of the Dockerfile. Rebuild it with `vagrant destroy -f` and
  `vagrant up --provider=docker`.

- **A Submitty service is not running**: open a shell with `vagrant ssh`, then check it
  with `systemctl status <service>` and `journalctl -u <service>`.
  `submitty_restart_services` restarts them all.

- **Lots of files show as modified in `git status` after an install**: an older version of
  the install scripts changed file permissions through the bind mount. If
  `git diff --summary` shows only `mode change 100644 => 100755` lines, restore the
  permissions with:

  ```bash
  git diff --summary | sed -n 's/^ mode change 100644 => 100755 //p' | tr '\n' '\0' | xargs -0 chmod 644
  ```
