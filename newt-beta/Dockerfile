# Use the official Home Assistant add-on base image
FROM ghcr.io/hassio-addons/base:14.0.0

# Define the Newt version
ARG NEWT_VERSION=1.20.1
ENV NEWT_VERSION=${NEWT_VERSION}
# SHA256 checksums for NEWT_VERSION above. Must be updated together with NEWT_VERSION.
# Fetch them with:
#   for a in amd64 arm64 arm32 arm32v6; do printf '%s ' "$a"; \
#     curl -sL "https://github.com/fosrl/newt/releases/download/${NEWT_VERSION}/newt_linux_$a" \
#     | sha256sum | cut -d' ' -f1; done
ARG NEWT_SHA256_AMD64="ac5556844efa38308b54c474c2bb9c512ea13a1c584d0ee1fc9d7b6fee01ddbb"
ARG NEWT_SHA256_ARM64="6a0e7c5775e4cfa1668e554ef2137722533fd7046c2be5495e7d01055c61f6b5"
ARG NEWT_SHA256_ARM32="25561bd2ae2e477fdc20f05094f4bc47b180bf211eca52b70ad74a57bd86b155"
ARG NEWT_SHA256_ARM32V6="d0c1b891cfe6fed7dd0cdfe45bcf0e988ab0cff502bcc96aa2b4d89f881dd77a"

# Enable Newt health file
ENV HEALTH_FILE=/tmp/healthy

# Install runtime dependencies and temporary build tooling
RUN apk add --no-cache bash jq && apk add --no-cache --virtual .build-deps curl

# Use Home Assistant's built-in BUILD_ARCH for safe architecture detection
ARG BUILD_ARCH
RUN \
    if [ "${BUILD_ARCH}" = "amd64" ]; then \
        DOWNLOAD_URL="https://github.com/fosrl/newt/releases/download/${NEWT_VERSION}/newt_linux_amd64"; \
        EXPECTED_SHA="${NEWT_SHA256_AMD64}"; \
    elif [ "${BUILD_ARCH}" = "aarch64" ]; then \
        DOWNLOAD_URL="https://github.com/fosrl/newt/releases/download/${NEWT_VERSION}/newt_linux_arm64"; \
        EXPECTED_SHA="${NEWT_SHA256_ARM64}"; \
    elif [ "${BUILD_ARCH}" = "armv7" ]; then \
        DOWNLOAD_URL="https://github.com/fosrl/newt/releases/download/${NEWT_VERSION}/newt_linux_arm32"; \
        EXPECTED_SHA="${NEWT_SHA256_ARM32}"; \
    elif [ "${BUILD_ARCH}" = "armhf" ]; then \
        # armhf is ARMv6 in Home Assistant and needs the v6 binary, not arm32 (v7)
        DOWNLOAD_URL="https://github.com/fosrl/newt/releases/download/${NEWT_VERSION}/newt_linux_arm32v6"; \
        EXPECTED_SHA="${NEWT_SHA256_ARM32V6}"; \
    else \
        echo "❌ ERROR: Unsupported Home Assistant architecture: ${BUILD_ARCH}"; exit 1; \
    fi && \
    echo "🔹 Downloading Newt for ${BUILD_ARCH} from ${DOWNLOAD_URL}..." && \
    curl --ipv4 --proto '=https' --tlsv1.2 --retry 3 --retry-delay 2 --fail --silent --show-error -L -o /usr/bin/newt "$DOWNLOAD_URL" && \
    if [ -z "$EXPECTED_SHA" ]; then \
        echo "❌ ERROR: No SHA256 configured for ${BUILD_ARCH}; refusing to ship an unverified binary"; exit 1; \
    fi && \
    # Important: exactly two regular spaces between the hash and the file path
    echo "${EXPECTED_SHA}  /usr/bin/newt" | sha256sum -c - && \
    chmod +x /usr/bin/newt && \
    apk del .build-deps

# Copy and prepare run script
COPY run.sh /run.sh
RUN chmod +x /run.sh

# Docker healthcheck. Newt owns the health file itself: it's written when the tunnel
# ping succeeds and removed when the connection is lost. start-period gives the tunnel
# time to come up before the container can be marked unhealthy.
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=60s \
  CMD test -f "$HEALTH_FILE" || exit 1

# Start the add-on
ENTRYPOINT ["/run.sh"]
