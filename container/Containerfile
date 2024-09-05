FROM docker.io/louislam/uptime-kuma:1.23.13-debian@sha256:96510915e6be539b76bcba2e6873591c67aca8a6075ff09f5b4723ae47f333fc AS app-donor

FROM docker.io/library/node:20.17.0-bookworm-slim@sha256:ee799af8710c0c414361d0c71f53a501cfc7bd6081336ae4fdcc223688a1e213

ARG UID=3310
ARG GID=3310

# renovate: datasource=pypi depName=apprise versioning=pep440
ARG APPRISE_VERSION=1.9.0

# renovate: datasource=github-releases depName=cloudflare/cloudflared
ARG CLOUDFLARED_VERSION=2024.8.3

COPY --from=app-donor /app /app

RUN apt-get update -qqy \
    # Install Uptime-Kuma dependencies
    && DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -qqy \
        python3 \
        python3-pip \
        python3-cryptography \
        python3-six \
        python3-yaml \
        python3-click \
        python3-markdown \
        python3-requests \
        python3-requests-oauthlib \
        sqlite3 \
        iputils-ping \
        util-linux \
        dumb-init \
        curl \
        ca-certificates \
        bash \
    && pip --no-cache-dir install --break-system-packages apprise==${APPRISE_VERSION} \
    && setcap -r /usr/bin/ping \
    && rm -rf /var/lib/apt/lists/* \
    \
    # Download and install cloudflared
    && ARCH= && dpkgArch="$(dpkg --print-architecture)" \
    && case "${dpkgArch##*-}" in \
      amd64) ARCH='amd64';; \
      arm64) ARCH='arm64';; \
      armhf) ARCH='arm';; \
      *) echo "unsupported architecture"; exit 1 ;; \
    esac \
    && curl -fsSLo /usr/local/bin/cloudflared https://github.com/cloudflare/cloudflared/releases/download/${CLOUDFLARED_VERSION}/cloudflared-linux-${ARCH} \
    && chmod +x /usr/local/bin/cloudflared \
    \
    # Setup non-root system account + group
    && addgroup --system --gid ${GID} uptime-kuma || true \
    && adduser --system --disabled-login --ingroup uptime-kuma --no-create-home --home /nonexistent --gecos "uptime-kuma" --shell /bin/false --uid ${UID} uptime-kuma || true \
    && mkdir -p /app/data \
    && chown -R uptime-kuma:0 /app \
    && chmod -R g=u /app \
    \
    # Smoke Tests
    && set -ex || exit $?; \
      cloudflared version; \
      apprise --version;

ENV HOME=/app
WORKDIR /app
USER uptime-kuma
EXPOSE 3001
VOLUME ["/app/data"]
CMD ["/usr/bin/dumb-init", "--", "node", "server/server.js"]
