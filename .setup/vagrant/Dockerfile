# Base image for running the Submitty development environment with Vagrant's
# docker provider (`vagrant up --provider=docker`).
#
# Submitty's install scripts expect a full Ubuntu machine: they manage services
# with systemctl, SSH in as the vagrant/root users, and run Docker themselves for
# autograding. This image therefore boots systemd as PID 1 and runs sshd, so
# that Vagrant can treat the container like any other VM. The container must be
# run privileged (see the docker provider block in the Vagrantfile).
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV container=docker

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        systemd \
        systemd-sysv \
        dbus \
        openssh-server \
        sudo \
        lsb-release \
        ca-certificates \
        curl \
        gnupg \
        iproute2 \
        iptables \
        kmod \
        tzdata \
        locales \
        rsync \
        # Packages that the Ubuntu server image ships with and that the Submitty
        # install scripts use before (or without) installing them.
        needrestart \
        software-properties-common \
        apt-utils \
        adduser \
        netbase \
        python3 \
        git \
        wget \
        cron \
        unzip \
        less \
    && mkdir -p -m 0755 /etc/apt/keyrings \
    && locale-gen en_US.UTF-8 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Units that either cannot work inside a container or just slow boot down.
RUN systemctl mask \
        getty.target \
        console-getty.service \
        systemd-udevd.service \
        systemd-udev-trigger.service \
        systemd-remount-fs.service \
        sys-kernel-debug.mount \
        sys-kernel-tracing.mount \
    && systemctl enable ssh

# The vagrant user, set up the way Vagrant boxes conventionally are: password
# "vagrant", passwordless sudo, and Vagrant's insecure public key (which Vagrant
# replaces with a fresh key pair on first boot).
RUN useradd --create-home --shell /bin/bash vagrant \
    && echo 'vagrant:vagrant' | chpasswd \
    && echo 'root:vagrant' | chpasswd \
    && echo 'vagrant ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/vagrant \
    && chmod 0440 /etc/sudoers.d/vagrant \
    && mkdir -p -m 700 /home/vagrant/.ssh \
    && curl -fsSL https://raw.githubusercontent.com/hashicorp/vagrant/main/keys/vagrant.pub \
        -o /home/vagrant/.ssh/authorized_keys \
    && chmod 600 /home/vagrant/.ssh/authorized_keys \
    && chown -R vagrant:vagrant /home/vagrant/.ssh

# The Vagrantfile logs in as root (password "vagrant") for `vagrant ssh`.
RUN sed -i \
        -e 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' \
        -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' \
        /etc/ssh/sshd_config \
    && mkdir -p /run/sshd

# Docker bind mounts keep the host's file ownership, so the repositories synced into
# /usr/local/submitty/GIT_CHECKOUT are owned by the host user rather than root (as they
# are on the VM providers), and git refuses to use them from root or the Submitty users.
# They are the developer's own checkouts, so mark them safe for every user.
RUN git config --system --add safe.directory '*'

# The ubuntu base image ships a policy-rc.d that stops packages from starting their
# services on install. Submitty's install expects them to start as they would on a VM
# (e.g. slapd, postgresql, docker), so remove it now that the image's own packages are in.
RUN rm -f /usr/sbin/policy-rc.d

# Nested Docker (used by the autograder) keeps its data on volumes, since
# overlayfs cannot be stacked on the container's own overlayfs root. Newer Docker
# releases store images in containerd's snapshotter, under /var/lib/containerd.
VOLUME ["/var/lib/docker", "/var/lib/containerd"]

EXPOSE 22
STOPSIGNAL SIGRTMIN+3
CMD ["/sbin/init"]
