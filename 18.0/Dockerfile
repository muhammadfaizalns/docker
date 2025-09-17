FROM ubuntu:noble
MAINTAINER Odoo S.A. <info@odoo.com>

SHELL ["/bin/bash", "-xo", "pipefail", "-c"]

# Generate locale C.UTF-8 for postgres and general locale data
ENV LANG en_US.UTF-8

# Retrieve the target architecture to install the correct wkhtmltopdf package
ARG TARGETARCH

# Install system dependencies, Python packages, and wkhtmltopdf
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        dirmngr \
        fonts-noto-cjk \
        gnupg \
        libssl-dev \
        node-less \
        npm \
        python3-magic \
        python3-num2words \
        python3-odf \
        python3-pdfminer \
        python3-pip \
        python3-phonenumbers \
        python3-pyldap \
        python3-qrcode \
        python3-renderpm \
        python3-setuptools \
        python3-slugify \
        python3-vobject \
        python3-watchdog \
        python3-xlrd \
        python3-xlwt \
        python3-geoip2 \
        python3-openpyxl \
        python3-gtts \
        python3-pandas \
        python3-sqlalchemy \
        python3-lxml-html-clean \
        wget \
        tar \
        xz-utils && \
    if [ -z "${TARGETARCH}" ]; then \
        TARGETARCH="$(dpkg --print-architecture)"; \
    fi; \
    WKHTMLTOPDF_ARCH=${TARGETARCH} && \
    case ${TARGETARCH} in \
    "amd64") WKHTMLTOPDF_ARCH=amd64 && WKHTMLTOPDF_SHA=967390a759707337b46d1c02452e2bb6b2dc6d59  ;; \
    "arm64")  WKHTMLTOPDF_SHA=90f6e69896d51ef77339d3f3a20f8582bdf496cc  ;; \
    "ppc64le" | "ppc64el") WKHTMLTOPDF_ARCH=ppc64el && WKHTMLTOPDF_SHA=5312d7d34a25b321282929df82e3574319aed25c  ;; \
    esac \
    && curl -o wkhtmltox.deb -sSL https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_${WKHTMLTOPDF_ARCH}.deb \
    && echo ${WKHTMLTOPDF_SHA} wkhtmltox.deb | sha1sum -c - \
    && apt-get install -y --no-install-recommends ./wkhtmltox.deb \
    && rm -rf /var/lib/apt/lists/* wkhtmltox.deb

# Install additional Python packages
RUN pip3 install --break-system-packages qifparse

# Setup GeoIP with automated download using your MaxMind license key
ARG MAXMIND_LICENSE_KEY=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
RUN mkdir -p /usr/share/GeoIP && \
    echo "🌍 Setting up GeoIP databases..." && \
    # Download GeoLite2 databases with your license key
    if [ ! -z "${MAXMIND_LICENSE_KEY}" ]; then \
        echo "📥 Downloading GeoLite2 databases..." && \
        ( \
            wget -q --timeout=30 -O /tmp/GeoLite2-City.tar.gz "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-City&license_key=${MAXMIND_LICENSE_KEY}&suffix=tar.gz" && \
            wget -q --timeout=30 -O /tmp/GeoLite2-Country.tar.gz "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=${MAXMIND_LICENSE_KEY}&suffix=tar.gz" && \
            echo "📦 Extracting databases..." && \
            tar -xzf /tmp/GeoLite2-City.tar.gz -C /tmp --strip-components=1 && \
            tar -xzf /tmp/GeoLite2-Country.tar.gz -C /tmp --strip-components=1 && \
            mv /tmp/GeoLite2-City.mmdb /usr/share/GeoIP/ && \
            mv /tmp/GeoLite2-Country.mmdb /usr/share/GeoIP/ && \
            echo "✅ GeoIP databases downloaded and installed successfully" \
        ) || echo "❌ Failed to download GeoIP databases - check license key"; \
        rm -rf /tmp/GeoLite2-* 2>/dev/null || true; \
    else \
        echo "⚠️ No MaxMind license key provided"; \
    fi && \
    # Set proper permissions
    chmod -R 755 /usr/share/GeoIP && \
    # Final status check
    if [ -f "/usr/share/GeoIP/GeoLite2-City.mmdb" ] && [ -f "/usr/share/GeoIP/GeoLite2-Country.mmdb" ]; then \
        echo "🎯 GeoIP ready: $(ls -lh /usr/share/GeoIP/)"; \
    else \
        echo "⚠️ GeoIP databases not found - website visitor location detection will not work"; \
    fi

# Install latest PostgreSQL client
RUN echo 'deb http://apt.postgresql.org/pub/repos/apt/ noble-pgdg main' > /etc/apt/sources.list.d/pgdg.list \
    && GNUPGHOME="$(mktemp -d)" \
    && export GNUPGHOME \
    && repokey='B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8' \
    && gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "${repokey}" \
    && gpg --batch --armor --export "${repokey}" > /etc/apt/trusted.gpg.d/pgdg.gpg.asc \
    && gpgconf --kill all \
    && rm -rf "$GNUPGHOME" \
    && apt-get update  \
    && apt-get install --no-install-recommends -y postgresql-client \
    && rm -f /etc/apt/sources.list.d/pgdg.list \
    && rm -rf /var/lib/apt/lists/*

# Install rtlcss
RUN npm install -g rtlcss

# Install Odoo
ENV ODOO_VERSION 18.0
ARG ODOO_RELEASE=20250909
ARG ODOO_SHA=3bdf96c966b7b0434b6358e07e7027e3d9b6d92f

RUN curl -o odoo.deb -sSL http://nightly.odoo.com/${ODOO_VERSION}/nightly/deb/odoo_${ODOO_VERSION}.${ODOO_RELEASE}_all.deb \
    && echo "${ODOO_SHA} odoo.deb" | sha1sum -c - \
    && apt-get update \
    && apt-get -y install --no-install-recommends ./odoo.deb \
    && rm -rf /var/lib/apt/lists/* odoo.deb

# Copy configuration files and scripts
COPY ./entrypoint.sh /
COPY ./odoo.conf /etc/odoo/
COPY wait-for-psql.py /usr/local/bin/wait-for-psql.py

# Update Odoo configuration for GeoIP and proxy mode (for Easypanel)
RUN echo "" >> /etc/odoo/odoo.conf && \
    echo "# GeoIP Configuration" >> /etc/odoo/odoo.conf && \
    echo "proxy_mode = True" >> /etc/odoo/odoo.conf && \
    echo "# GeoIP databases are automatically located at /usr/share/GeoIP/" >> /etc/odoo/odoo.conf && \
    echo "# Uncomment below to specify custom paths:" >> /etc/odoo/odoo.conf && \
    echo "# geoip_city_db = /usr/share/GeoIP/GeoLite2-City.mmdb" >> /etc/odoo/odoo.conf && \
    echo "# geoip_country_db = /usr/share/GeoIP/GeoLite2-Country.mmdb" >> /etc/odoo/odoo.conf

# Set up directories and permissions
RUN chown odoo /etc/odoo/odoo.conf \
    && mkdir -p /mnt/extra-addons \
    && chown -R odoo /mnt/extra-addons \
    && chown -R root:root /usr/share/GeoIP

# Define volumes
VOLUME ["/var/lib/odoo", "/mnt/extra-addons"]

# Expose Odoo services
EXPOSE 8069 8071 8072

# Set the default config file
ENV ODOO_RC /etc/odoo/odoo.conf

# Add health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8069/web/health || exit 1

# Set default user when running the container
USER odoo

ENTRYPOINT ["/entrypoint.sh"]
CMD ["odoo"]

# 🌍 GeoIP Setup Complete!
# ✅ Your MaxMind license key is embedded
# ✅ Proxy mode enabled for Easypanel
# ✅ Health check included
# 🚀 Ready to deploy in Easypanel!
