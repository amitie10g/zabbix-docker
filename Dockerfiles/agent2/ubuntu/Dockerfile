# syntax=docker/dockerfile:1
ARG MAJOR_VERSION=6.4
ARG ZBX_VERSION=${MAJOR_VERSION}.15
ARG BUILD_BASE_IMAGE=zabbix-build-mysql:ubuntu-${ZBX_VERSION}

FROM ${BUILD_BASE_IMAGE} AS builder

FROM ubuntu:jammy

ARG MAJOR_VERSION
ARG ZBX_VERSION
ARG ZBX_SOURCES=https://git.zabbix.com/scm/zbx/zabbix.git

ENV TERM=xterm \
    ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES}

LABEL org.opencontainers.image.authors="Alexey Pustovalov <alexey.pustovalov@zabbix.com>" \
      org.opencontainers.image.description="Zabbix agent 2 is deployed on a monitoring target to actively monitor local resources and applications" \
      org.opencontainers.image.documentation="https://www.zabbix.com/documentation/${MAJOR_VERSION}/manual/installation/containers" \
      org.opencontainers.image.licenses="GPL v2.0, Apache v2.0 for plugins" \
      org.opencontainers.image.source="${ZBX_SOURCES}" \
      org.opencontainers.image.title="Zabbix agent 2" \
      org.opencontainers.image.url="https://zabbix.com/" \
      org.opencontainers.image.vendor="Zabbix SIA" \
      org.opencontainers.image.version="${ZBX_VERSION}"

STOPSIGNAL SIGTERM

COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/go/bin/zabbix_agent2", "/usr/sbin/zabbix_agent2"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_get/zabbix_get", "/usr/bin/zabbix_get"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_sender/zabbix_sender", "/usr/bin/zabbix_sender"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/go/conf/zabbix_agent2.conf", "/etc/zabbix/zabbix_agent2.conf"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/go/conf/zabbix_agent2.d/plugins.d", "/etc/zabbix/zabbix_agent2.d/plugins.d"]
COPY --from=builder ["/tmp/mongodb_plugin/mongodb.conf", "/etc/zabbix/zabbix_agent2.d/plugins.d/mongodb.conf"]
COPY --from=builder ["/tmp/postgresql_plugin/postgresql.conf", "/etc/zabbix/zabbix_agent2.d/plugins.d/postgresql.conf"]
COPY --from=builder ["/tmp/mongodb_plugin/zabbix-agent2-plugin-mongodb", "/usr/sbin/zabbix-agent2-plugin/zabbix-agent2-plugin-mongodb"]
COPY --from=builder ["/tmp/postgresql_plugin/zabbix-agent2-plugin-postgresql", "/usr/sbin/zabbix-agent2-plugin/zabbix-agent2-plugin-postgresql"]

RUN --mount=type=cache,target=/var/lib/apt/,sharing=locked \
    set -eux && \
    echo "#!/bin/sh\nexit 101" > /usr/sbin/policy-rc.d && \
    INSTALL_PKGS="tini \
            tzdata \
            ca-certificates \
            smartmontools \
            sudo \
            libssl3 \
            libcurl4 \
            libpcre2-8-0 \
            libldap-2.4" && \
    apt-get -y update && \
    DEBIAN_FRONTEND=noninteractive apt-get -y \
            --no-install-recommends install \
        ${INSTALL_PKGS} && \
    groupadd \
            --system \
            --gid 1995 \
        zabbix && \
    useradd \
            --system \
            --comment "Zabbix monitoring system" \
            -g zabbix \
            --uid 1997 \
            --shell /sbin/nologin \
            --home-dir /var/lib/zabbix/ \
        zabbix && \
    echo "zabbix ALL=(root) NOPASSWD: /usr/sbin/smartctl" >> /etc/sudoers.d/zabbix && \
    mkdir -p /etc/zabbix && \
    mkdir -p /etc/zabbix/zabbix_agentd.d && \
    mkdir -p /var/lib/zabbix && \
    mkdir -p /var/lib/zabbix/enc && \
    mkdir -p /var/lib/zabbix/modules && \
    mkdir -p /var/lib/zabbix/buffer && \
    chown --quiet -R zabbix:root /etc/zabbix/ /var/lib/zabbix/ /usr/sbin/zabbix-agent2-plugin/ && \
    chgrp -R 0 /etc/zabbix/ /var/lib/zabbix/ /usr/sbin/zabbix-agent2-plugin/ && \
    chmod -R g=u /etc/zabbix/ /var/lib/zabbix/ /usr/sbin/zabbix-agent2-plugin/ && \
    apt-get -y autoremove && \
    apt-get -y clean

EXPOSE 10050/TCP

WORKDIR /var/lib/zabbix

COPY ["docker-entrypoint.sh", "/usr/bin/"]

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/bin/docker-entrypoint.sh"]

USER 1997

CMD ["/usr/sbin/zabbix_agent2", "--foreground", "-c", "/etc/zabbix/zabbix_agent2.conf"]
