# Runs the unmodified ./install.sh inside a systemd-enabled Ubuntu 24.04
# container, since install.sh manages every service (nginx, mariadb, tor,
# postfix, dovecot, php-fpm, ...) through systemctl.
#
# Build context must be the repository root, e.g.:
#   docker build -f docker/Dockerfile -t hosting .
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    systemd systemd-sysv ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# The official Ubuntu image ships a policy-rc.d that blocks services from
# being started by their postinst scripts during "apt install" (normally
# desired so a docker build doesn't spawn stray daemons). install.sh relies
# on a real machine's default behavior, where mariadb/nginx/postfix/dovecot
# etc. start themselves right after being installed, so remove it to match.
RUN rm -f /usr/sbin/policy-rc.d

# Units that don't make sense (or don't work) inside a container
RUN find /etc/systemd/system /lib/systemd/system -path '*.wants/*' \
        \( -name '*getty*' -o -name '*udev*' \) -exec rm -f {} \; ; \
    systemctl mask -- \
        systemd-udevd.service systemd-udevd-kernel.socket \
        systemd-udevd-control.socket systemd-modules-load.service \
        sys-kernel-config.mount sys-kernel-debug.mount sys-fs-fuse-connections.mount \
        getty.target getty-static.service console-getty.service \
    || true

COPY . /root/hosting
COPY docker/hosting-firstboot.service /etc/systemd/system/hosting-firstboot.service
RUN systemctl enable hosting-firstboot.service

STOPSIGNAL SIGRTMIN+3

VOLUME ["/sys/fs/cgroup"]

CMD ["/lib/systemd/systemd"]
