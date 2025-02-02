# syntax=docker/dockerfile:1
ARG MAJOR_VERSION=6.4
ARG ZBX_VERSION=${MAJOR_VERSION}.0
ARG BUILD_BASE_IMAGE=zabbix-build-oracle:centos-${ZBX_VERSION}

FROM ${BUILD_BASE_IMAGE} as builder

FROM quay.io/centos/centos:stream9

ARG MAJOR_VERSION
ARG ZBX_VERSION
ARG ZBX_SOURCES=https://git.zabbix.com/scm/zbx/zabbix.git

ENV TERM=xterm \
    ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES} \
    MIBDIRS=/usr/share/snmp/mibs:/var/lib/zabbix/mibs MIBS=+ALL \
    JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.18.0.10-3.el9.x86_64 PATH="$JAVA_HOME/bin:$PATH" LD_LIBRARY_PATH=/usr/lib/oracle/19.18/client64/lib

LABEL org.opencontainers.image.authors="Alexey Pustovalov <alexey.pustovalov@zabbix.com>" \
      org.opencontainers.image.description="Zabbix server with Oracle database support" \
      org.opencontainers.image.documentation="https://www.zabbix.com/documentation/${MAJOR_VERSION}/manual/installation/containers" \
      org.opencontainers.image.licenses="GPL v2.0" \
      org.opencontainers.image.source="${ZBX_SOURCES}" \
      org.opencontainers.image.title="Zabbix server (Oracle)" \
      org.opencontainers.image.url="https://zabbix.com/" \
      org.opencontainers.image.vendor="Zabbix LLC" \
      org.opencontainers.image.version="${ZBX_VERSION}"

STOPSIGNAL SIGTERM

COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_server/zabbix_server", "/usr/sbin/zabbix_server"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_get/zabbix_get", "/usr/bin/zabbix_get"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/src/zabbix_sender/zabbix_sender", "/usr/bin/zabbix_sender"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/conf/zabbix_server.conf", "/etc/zabbix/zabbix_server.conf"]
COPY --from=builder ["/tmp/zabbix-${ZBX_VERSION}/database/oracle/create_server.sql.gz", "/usr/share/doc/zabbix-server-oracle/create.sql.gz"]
COPY --from=builder ["/usr/lib/oracle/19.18/client64/lib/", "/usr/lib/oracle/19.18/client64/lib/"]
COPY --from=builder ["/usr/lib/jvm/", "/usr/lib/jvm/"]
COPY --from=builder ["/usr/share/", "/usr/share/"]

COPY ./create.sql.gz /usr/share/doc/zabbix-server-oracle/create.sql.gz

RUN set -eux && \
    REPOLIST="baseos,appstream,crb,epel" && \
    INSTALL_PKGS="fping \
            tini \
            sudo \
            traceroute \
            nmap \
            file-libs \
            tzdata \
            iputils \
            traceroute \
            libevent \
            libxml2 \
            libssh \
            net-snmp-libs \
            OpenIPMI-libs \
            openldap \
            openssl-libs \
            libnsl \
            libaio \
            pcre2 \
            zlib \
            wget \
            unzip \
            unixODBC" && \
    dnf -y install epel-release && \
    dnf -y install \
            --disablerepo "*" \
            --enablerepo "${REPOLIST}" \
            --setopt=tsflags=nodocs \
            --setopt=install_weak_deps=False \
            --best \
        ${INSTALL_PKGS} && \
    wget -O /sqlcl.zip https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip && \
    unzip -d /usr/local sqlcl.zip && \
    rm -rf sqlcl.zip && \
    ln -sT /usr/local/sqlcl/bin/sql /usr/local/bin/sqlcl && \
    groupadd \
            --system \
            --gid 1995 \
        zabbix && \
    useradd \
            --system \
            --comment "Zabbix monitoring system" \
            -g zabbix \
            -G root,dialout \
            --uid 1997 \
            --shell /sbin/nologin \
            --home-dir /var/lib/zabbix/ \
        zabbix && \
    echo "zabbix ALL=(root) NOPASSWD: /usr/bin/nmap" >> /etc/sudoers.d/zabbix && \
    mkdir -p /etc/zabbix && \
    mkdir -p /var/lib/zabbix && \
    mkdir -p /usr/lib/zabbix/alertscripts && \
    mkdir -p /usr/lib/zabbix/dbscripts && \
    mkdir -p /var/lib/zabbix/dbscripts && \
    mkdir -p /var/lib/zabbix/enc && \
    mkdir -p /var/lib/zabbix/export && \
    mkdir -p /usr/lib/zabbix/externalscripts && \
    mkdir -p /var/lib/zabbix/mibs && \
    mkdir -p /var/lib/zabbix/modules && \
    mkdir -p /var/lib/zabbix/snmptraps && \
    mkdir -p /var/lib/zabbix/ssh_keys && \
    mkdir -p /var/lib/zabbix/ssl && \
    mkdir -p /var/lib/zabbix/ssl/certs && \
    mkdir -p /var/lib/zabbix/ssl/keys && \
    mkdir -p /var/lib/zabbix/ssl/ssl_ca && \
    mkdir -p /usr/share/doc/zabbix-server-oracle && \
    chown --quiet -R zabbix:root /etc/zabbix/ /var/lib/zabbix/ && \
    chgrp -R 0 /etc/zabbix/ /var/lib/zabbix/ && \
    chmod -R g=u /etc/zabbix/ /var/lib/zabbix/ && \
    dnf -y clean all && \
    rm -rf /var/cache/yum /var/lib/yum/yumdb/* /usr/lib/udev/hwdb.d/* && \
    rm -rf /var/cache/dnf /etc/udev/hwdb.bin /root/.pki

EXPOSE 10051/TCP

WORKDIR /var/lib/zabbix

VOLUME ["/var/lib/zabbix/snmptraps", "/var/lib/zabbix/export"]

COPY ["docker-entrypoint.sh", "/usr/bin/"]

ENTRYPOINT ["/usr/bin/tini", "--", "/usr/bin/docker-entrypoint.sh"]

USER 1997

CMD ["/usr/sbin/zabbix_server", "--foreground", "-c", "/etc/zabbix/zabbix_server.conf"]
