FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl jq ca-certificates unzip && \
    rm -rf /var/lib/apt/lists/*

# Install mc (MinIO client) with checksum verification
RUN curl -fsSL https://dl.min.io/client/mc/release/linux-amd64/mc -o /usr/local/bin/mc && \
    curl -fsSL https://dl.min.io/client/mc/release/linux-amd64/mc.sha256sum -o /tmp/mc.sha256sum && \
    cd /usr/local/bin && sha256sum -c /tmp/mc.sha256sum && rm /tmp/mc.sha256sum && \
    chmod +x /usr/local/bin/mc

# Install vault CLI with checksum verification
RUN curl -fsSL https://releases.hashicorp.com/vault/1.21.2/vault_1.21.2_linux_amd64.zip -o /tmp/vault.zip && \
    curl -fsSL https://releases.hashicorp.com/vault/1.21.2/vault_1.21.2_SHA256SUMS -o /tmp/vault_sums.txt && \
    cd /tmp && grep vault_1.21.2_linux_amd64.zip vault_sums.txt | sha256sum -c - && \
    unzip vault.zip && mv vault /usr/local/bin/ && rm vault.zip vault_sums.txt

# Python HTTP client
RUN pip install --no-cache-dir requests

USER 1000
WORKDIR /opt/sentinel-ops
