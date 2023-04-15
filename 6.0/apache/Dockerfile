FROM docker.io/php:8.1-apache
LABEL maintainer="markus@martialblog.de"

# Install OS dependencies
RUN set -ex; \
        apt-get update && \
        DEBIAN_FRONTEND=noninteractive \
        apt-get install --no-install-recommends -y \
        \
        libldap2-dev \
        libfreetype6-dev \
        libjpeg-dev \
        libonig-dev \
        zlib1g-dev \
        libc-client-dev \
        libkrb5-dev \
        libpng-dev \
        libpq-dev \
        libzip-dev \
        libtidy-dev \
        libsodium-dev \
        freetds-dev \
        freetds-bin \
        netcat \
        curl \
        \
        && apt-get -y autoclean; apt-get -y autoremove; \
        rm -rf /var/lib/apt/lists/*

# Link LDAP library for PHP ldap extension
RUN set -ex; \
        ln -fs /usr/lib/x86_64-linux-gnu/libldap.so /usr/lib/

# Install PHP Plugins and Configure PHP imap plugin
RUN set -ex; \
        docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr && \
        docker-php-ext-configure imap --with-kerberos --with-imap-ssl && \
        docker-php-ext-install -j5 \
        exif \
        gd \
        imap \
        ldap \
        mbstring \
        pdo \
        pdo_mysql \
        pdo_pgsql \
        pgsql \
        pdo_dblib \
        sodium \
        tidy \
        zip

# # sqlsrv packages
# RUN set -ex; \
#         apt-get update && apt-get install -y gnupg && \
#         curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
#         curl https://packages.microsoft.com/config/debian/11/prod.list > /etc/apt/sources.list.d/mssql-release.list && \
#         apt-get update && \
#         ACCEPT_EULA=Y apt-get install -y libltdl-dev libltdl7 msodbcsql18 unixodbc-dev

# # sqlsrv extensions
# RUN set -ex; \
#         pecl install sqlsrv && \
#         pecl install pdo_sqlsrv && \
#         mkdir -p /etc/php.d/ && \
#         printf "; priority=20\nextension=sqlsrv.so\n" > /etc/php.d/sqlsrv.ini && \
#         printf "; priority=30\nextension=pdo_sqlsrv.so\n" > /etc/php.d/pdo_sqlsrv.ini && \
#         phpenmod sqlsrv pdo_sqlsrv
# install and setup custom extensions
RUN apt-get -qq -y update \
  && apt-get --no-install-recommends -qq -y install apt-transport-https gnupg \
  # mssql odbc driver
  && curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
  && curl https://packages.microsoft.com/config/debian/11/prod.list > /etc/apt/sources.list.d/mssql-release.list \
  && apt-get -qq -y update \
  && ACCEPT_EULA=Y apt-get install -y msodbcsql17 odbcinst=2.3.7 odbcinst1debian2=2.3.7 unixodbc-dev=2.3.7 unixodbc=2.3.7 \
  # install and enable extensions
  && pecl install pdo_sqlsrv-5.10.1 \
  && docker-php-ext-enable pdo_sqlsrv \
  && pecl install sqlsrv-5.10.1 \
  && docker-php-ext-enable sqlsrv \
  && ACCEPT_EULA=Y apt-get install -y mssql-tools \
  && echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc

# Apache configuration
RUN a2enmod headers rewrite remoteip; \
        {\
        echo RemoteIPHeader X-Real-IP ;\
        echo RemoteIPTrustedProxy 10.0.0.0/8 ;\
        echo RemoteIPTrustedProxy 172.16.0.0/12 ;\
        echo RemoteIPTrustedProxy 192.168.0.0/16 ;\
        } > /etc/apache2/conf-available/remoteip.conf;\
        a2enconf remoteip

# Use the default production configuration
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

ARG version="6.0.0+230405"
ARG sha256_checksum="ee636330911150b5f109323b6226c48d9fd7634856b251bbf66a75a29d4b4756"
ARG archive_url="https://github.com/LimeSurvey/LimeSurvey/archive/${version}.tar.gz"
ARG USER=www-data
ARG LISTEN_PORT=8080
ENV LIMESURVEY_VERSION=$version

# Download, unzip and chmod LimeSurvey from GitHub (defaults to the official LimeSurvey/LimeSurvey repository)
RUN set -ex; \
        curl -sSL "${archive_url}" --output /tmp/limesurvey.tar.gz && \
        echo "${sha256_checksum}  /tmp/limesurvey.tar.gz" | sha256sum -c - && \
        \
        tar xzvf "/tmp/limesurvey.tar.gz" --strip-components=1 -C /var/www/html/ && \
        rm -f "/tmp/limesurvey.tar.gz" && \
        chown -R "$USER:$USER" /var/www/html /etc/apache2

EXPOSE $LISTEN_PORT

WORKDIR /var/www/html
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
COPY vhosts-access-log.conf /etc/apache2/conf-enabled/other-vhosts-access-log.conf
USER $USER
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["apache2-foreground"]
