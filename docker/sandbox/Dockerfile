ARG SANDBOX_BASE_IMAGE=enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:1.11.0
FROM ${SANDBOX_BASE_IMAGE}

ARG DOTNET_VERSION=8.0.425
ARG PYTHON_DOCX_VERSION=1.2.0
ARG MATPLOTLIB_VERSION=3.10.9

ENV DOTNET_ROOT=/opt/dotnet \
    PATH=/opt/dotnet:${PATH} \
    DOTNET_CLI_TELEMETRY_OPTOUT=1 \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1 \
    DOTNET_NOLOGO=1

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        libc6 \
        libgcc-s1 \
        libgssapi-krb5-2 \
        libicu70 \
        libssl3 \
        libstdc++6 \
        wget \
        zlib1g \
    && rm -rf /var/lib/apt/lists/* \
    && wget -q https://dot.net/v1/dotnet-install.sh -O /tmp/dotnet-install.sh \
    && chmod +x /tmp/dotnet-install.sh \
    && /tmp/dotnet-install.sh --version "${DOTNET_VERSION}" --install-dir /opt/dotnet \
    && rm /tmp/dotnet-install.sh \
    && dotnet --info \
    && dotnet --list-sdks | grep -q '^8\.'

RUN python -m pip install --no-cache-dir \
    "python-docx==${PYTHON_DOCX_VERSION}" \
    "matplotlib==${MATPLOTLIB_VERSION}" \
    && python -c "import docx, matplotlib; print('python-docx=' + docx.__version__); print('matplotlib=' + matplotlib.__version__)"
