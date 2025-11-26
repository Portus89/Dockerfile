# =====================================
# Dockerfile para Nagios Core con correcciones de errores
# =====================================
FROM ubuntu:latest

LABEL maintainer="v.portus@duocuc.cl"

ENV NAGIOS_VERSION=4.5.2 \
    PLUGIN_VERSION=2.3.3 \
    DEBIAN_FRONTEND=noninteractive \
    WORKDIR=/tmp

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        wget build-essential unzip apache2 apache2-utils php \
        libapache2-mod-php libgd-dev libperl-dev libssl-dev \
        daemon make iputils-ping && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN useradd nagios && \
    groupadd nagcmd && \
    usermod -aG nagcmd nagios && \
    usermod -aG nagcmd www-data && \
    wget --no-check-certificate -O ${WORKDIR}/nagios.tar.gz https://assets.nagios.com/downloads/nagioscore/releases/nagios-${NAGIOS_VERSION}.tar.gz && \
    tar -zxvf ${WORKDIR}/nagios.tar.gz -C ${WORKDIR} && \
    cd ${WORKDIR}/nagios-${NAGIOS_VERSION} && \
    ./configure --with-command-group=nagcmd > /dev/null && \
    make all && make install && \
    make install-init && make install-config && make install-commandmode && make install-webconf && \
    rm -rf ${WORKDIR}/nagios*

RUN wget --no-check-certificate -O ${WORKDIR}/plugins.tar.gz https://nagios-plugins.org/download/nagios-plugins-${PLUGIN_VERSION}.tar.gz && \
    tar -zxvf ${WORKDIR}/plugins.tar.gz -C ${WORKDIR} && \
    cd ${WORKDIR}/nagios-plugins-${PLUGIN_VERSION} && \
    ./configure --with-nagios-user=nagios --with-nagios-group=nagios > /dev/null && \
    make && make install && \
    rm -rf ${WORKDIR}/plugins*

RUN htpasswd -b -c /usr/local/nagios/etc/htpasswd.users nagiosadmin nagios && \
    a2enmod cgi && \
    a2enmod rewrite && \
    echo "ServerName localhost" >> /etc/apache2/apache2.conf

RUN echo '#!/bin/bash\n\
service apache2 start\n\
/usr/local/nagios/bin/nagios -d /usr/local/nagios/etc/nagios.cfg\n\
tail -f /usr/local/nagios/var/nagios.log' > /start.sh && \
    chmod +x /start.sh

EXPOSE 80 443

CMD ["/start.sh"]




