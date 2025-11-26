# =====================================
# Dockerfile para Nagios Core con correcciones de errores
# =====================================

FROM ubuntu:latest

LABEL maintainer="v.portus@duocuc.cl"

# Establecer variables de versión y entorno
ENV NAGIOS_VERSION=4.5.2 \
    PLUGIN_VERSION=2.3.3 \
    DEBIAN_FRONTEND=noninteractive \
    # Directorio de trabajo temporal para compilación
    WORKDIR=/tmp

# 1. Instalar dependencias necesarias
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        wget build-essential unzip apache2 apache2-utils php \
        libapache2-mod-php libgd-dev libperl-dev libssl-dev \
        daemon make iputils-ping && \
    # Limpiar caché para reducir el tamaño de la imagen
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 2. Crear usuario/grupo y Compilar/Instalar Nagios Core
# ATENCIÓN: Se añade --no-check-certificate para resolver el Error de SSL (exit code 5).
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

# 3. Compilación e instalación de Plugins
# ATENCIÓN: Se añade --no-check-certificate para la descarga de plugins.
RUN wget --no-check-certificate -O ${WORKDIR}/plugins.tar.gz https://nagios-plugins.org/download/nagios-plugins-${PLUGIN_VERSION}.tar.gz && \
    tar -zxvf ${WORKDIR}/plugins.tar.gz -C ${WORKDIR} && \
    cd ${WORKDIR}/nagios-plugins-${PLUGIN_VERSION} && \
    ./configure --with-nagios-user=nagios --with-nagios-group=nagios > /dev/null && \
    make && make install && \
    rm -rf ${WORKDIR}/plugins*

# 4. Configuración de autenticación y Apache
# Crea el usuario 'nagiosadmin' con contraseña 'nagios'
RUN htpasswd -b -c /usr/local/nagios/etc/htpasswd.users nagiosadmin nagios && \
    # Habilitar CGI y evitar advertencias
    a2enmod cgi && \
    a2enmod rewrite && \
    echo "ServerName localhost" >> /etc/apache2/apache2.conf

# 5. Script de arranque y Punto de Entrada
# Inicia Apache y Nagios en segundo plano, y mantiene el contenedor en ejecución
RUN echo '#!/bin/bash
# Iniciar Apache
service apache2 start
# Iniciar Nagios en modo daemon
/usr/local/nagios/bin/nagios -d /usr/local/nagios/etc/nagios.cfg
# Mantener el contenedor vivo y monitorear el log principal
tail -f /usr/local/nagios/var/nagios.log' > /start.sh && \
chmod +x /start.sh
EXPOSE 80 443
