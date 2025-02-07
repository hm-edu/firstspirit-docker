ARG IMAGE_CREATED="2023-06-21T15:00:00CEST"
ARG IMAGE_VERSION="5.2.230607"
ARG IMAGE_VERSION_SHORT="2023.6"
ARG FS_DOWNLOAD_SERVER
ARG FS_DOWNLOAD_SERVER_USERNAME
ARG FS_DOWNLOAD_SERVER_PASSWORD
ARG FS_DOWNLOAD_SERVER_NAME="fs-isolated-server.jar"
ARG FS_DOWNLOAD_INSTALL_NAME="fs-install-3.0.5.tar.gz"
ARG FS_BASE_DIRECTORY="/opt/firstspirit5"
ARG FS_INSTALL_DIRECTORY="/install/firstspirit5"
ARG FS_DEBUG_PORT="*:8585"

FROM ubuntu:jammy AS install
# Reuse the global variables
ARG IMAGE_VERSION
ARG IMAGE_VERSION_SHORT
ARG FS_DOWNLOAD_SERVER
ARG FS_DOWNLOAD_SERVER_USERNAME
ARG FS_DOWNLOAD_SERVER_PASSWORD
ARG FS_DOWNLOAD_SERVER_NAME
ARG FS_DOWNLOAD_INSTALL_NAME

COPY /config /tmp

RUN apt-get update && apt-get install curl -y && \
    cd /tmp && \
    curl -SsLfO -u ${FS_DOWNLOAD_SERVER_USERNAME}:${FS_DOWNLOAD_SERVER_PASSWORD} ${FS_DOWNLOAD_SERVER}/${IMAGE_VERSION_SHORT}/install/${FS_DOWNLOAD_INSTALL_NAME} && \
    curl -SsLfO -u ${FS_DOWNLOAD_SERVER_USERNAME}:${FS_DOWNLOAD_SERVER_PASSWORD} ${FS_DOWNLOAD_SERVER}/${IMAGE_VERSION_SHORT}/${FS_DOWNLOAD_SERVER_NAME} && \
    tar zxvf /tmp/${FS_DOWNLOAD_INSTALL_NAME} -C /tmp/ && \
    rm /tmp/${FS_DOWNLOAD_INSTALL_NAME} && \
    echo "${IMAGE_VERSION}" >> /tmp/firstspirit5/.version


FROM eclipse-temurin:11-jdk AS jdk

ENV LANG="de_DE.UTF-8" \
    LANGUAGE="de_DE.UTF-8"

RUN apt-get update && apt-get install procps tini unzip net-tools curl fontconfig -y &&\
    cp /usr/share/zoneinfo/Europe/Berlin /etc/localtime &&\
    echo "Europe/Berlin"> /etc/timezone

FROM jdk AS base
# Reuse the global variables
ARG IMAGE_CREATED
ARG IMAGE_VERSION
ARG IMAGE_VERSION_SHORT
ARG FS_DOWNLOAD_SERVER_NAME
ARG FS_BASE_DIRECTORY
ARG FS_INSTALL_DIRECTORY
ARG HEALTHCHECK_INTERVAL
ARG HEALTHCHECK_TIMEOUT
ARG HEALTHCHECK_START_PERIOD
ARG HEALTHCHECK_RETRIES

ENV FS_BASEDIR="${FS_BASE_DIRECTORY}" \
    FS_INSTALLDIR="${FS_INSTALL_DIRECTORY}" \
    FS_VERSION="${IMAGE_VERSION}" \
    FS_INSTALL_VERSION="${FS_INSTALL_VERSION}" \
    FS_JAVA_HOME=$JAVA_HOME

LABEL org.opencontainers.image.base.name="eclipse-temurin:17-jdk" \
    org.opencontainers.image.authors="Monday Consulting GmbH <helpdesk@monday-consulting.com>" \
    org.opencontainers.image.created=${IMAGE_CREATED} \
    org.opencontainers.image.description="FirstSpirit in a Container" \
    org.opencontainers.image.licenses="Apache-2.0" \
    org.opencontainers.image.title="FirstSpirit" \
    org.opencontainers.image.version=${IMAGE_VERSION}

HEALTHCHECK --interval="30s" \
    --timeout="3s" \
    --start-period="5m" \
    --retries="3" \
    CMD health-check.sh

RUN mkdir -p ${FS_INSTALL_DIRECTORY} && \
    useradd -d ${FS_INSTALL_DIRECTORY} -s /bin/sh fs

COPY --from=install --chown=fs /tmp/firstspirit5 ${FS_INSTALL_DIRECTORY}

COPY --from=install --chown=fs /tmp/${FS_DOWNLOAD_SERVER_NAME} ${FS_INSTALL_DIRECTORY}/server/lib-isolated/

COPY --from=install --chown=fs /tmp/banner.txt ${FS_INSTALL_DIRECTORY}

COPY --from=install --chown=root /tmp/*.sh /usr/local/bin/

RUN sed -i '1 i\allowedRedirectHosts=ALLOW_ALL' ${FS_INSTALL_DIRECTORY}/conf/fs-server.conf && \
    chmod +x /usr/local/bin/docker-entrypoint.sh && \
    chmod +x /usr/local/bin/health-check.sh && \
    apt-get update && apt-get upgrade -y && \
    apt-get clean && rm -rf /var/lib/apt/lists /var/cache/apt/archives

ENTRYPOINT [ "/usr/bin/tini", "--", "docker-entrypoint.sh" ]

CMD ["start"]

FROM base
# Reuse the global variables
ARG FS_INSTALL_DIRECTORY
ARG FS_DEBUG_PORT

ENV FS_DEBUGPORT=${FS_DEBUG_PORT}

RUN sed -i '/wrapper.java.additional.85/c\wrapper.java.additional.85=-Xdebug' ${FS_INSTALL_DIRECTORY}/conf/fs-wrapper.isolated.conf &&\
    sed -i "/wrapper.java.additional.86/c\wrapper.java.additional.86=-Xrunjdwp:transport=dt_socket,address=$FS_DEBUGPORT,server=y,suspend=n" ${FS_INSTALL_DIRECTORY}/conf/fs-wrapper.isolated.conf