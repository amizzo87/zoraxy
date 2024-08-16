FROM docker.io/golang:1.23-bookworm AS build

RUN mkdir -p /opt/zoraxy/source/ &&\
    mkdir -p /opt/zoraxy/config/ &&\
    mkdir -p /usr/local/bin/

# If you build it yourself, you will need to add the src directory into the docker directory.
COPY ./src/ /opt/zoraxy/source/

WORKDIR /opt/zoraxy/source/

RUN go mod tidy &&\
    go build -o /usr/local/bin/zoraxy &&\
    rm -r /opt/zoraxy/source/

RUN chmod +x /usr/local/bin/zoraxy

FROM debian:bookworm-slim

# Install required packages
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    netcat-openbsd \
    sudo \
    gnupg \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install ZeroTier
RUN curl -s https://install.zerotier.com | bash || (echo "Failed to install ZeroTier" && exit 1)

# Install Twingate
RUN curl -s https://binaries.twingate.com/client/linux/install.sh | bash || (echo "Failed to install Twingate" && exit 1)

# Verify installations
RUN zerotier-cli -v && twingate --version

COPY --from=build /usr/local/bin/zoraxy /usr/local/bin/zoraxy
COPY --from=build /opt/zoraxy/config/ /opt/zoraxy/config

VOLUME [ "/opt/zoraxy/config/" ]

WORKDIR /opt/zoraxy/config/

ENV AUTORENEW="86400"
ENV EARLYRENEW="30"
ENV FASTGEOIP="false"
ENV MDNS="true"
ENV MDNSNAME="''"
ENV NOAUTH="false"
ENV PORT="8000"
ENV SSHLB="false"
ENV VERSION="false"
ENV WEBFM="true"
ENV WEBROOT="./www"
ENV ZTAUTH="''"
ENV ZTPORT="9993"
ENV ZTNETWORK=""
ENV TWINGATE_TOKEN=""

# Expose ports
EXPOSE ${PORT}
EXPOSE ${ZTPORT}

# Create a startup script
RUN echo '#!/bin/bash' > /start.sh && \
    echo 'TWINGATE_TOKEN_FILE="/opt/zoraxy/config/twingate_token.json"' >> /start.sh && \
    echo 'if [ ! -f "$TWINGATE_TOKEN_FILE" ] && [ ! -z "$TWINGATE_TOKEN" ]; then' >> /start.sh && \
    echo '    echo "$TWINGATE_TOKEN" > "$TWINGATE_TOKEN_FILE"' >> /start.sh && \
    echo 'fi' >> /start.sh && \
    echo 'if [ -f "$TWINGATE_TOKEN_FILE" ]; then' >> /start.sh && \
    echo '    twingate setup --headless "$TWINGATE_TOKEN_FILE"' >> /start.sh && \
    echo '    twingate start' >> /start.sh && \
    echo 'fi' >> /start.sh && \
    echo 'zerotier-one -d' >> /start.sh && \
    echo 'sleep 5' >> /start.sh && \
    echo 'if [ ! -z "$ZTNETWORK" ]; then' >> /start.sh && \
    echo '    zerotier-cli join $ZTNETWORK' >> /start.sh && \
    echo 'fi' >> /start.sh && \
    echo 'exec zoraxy -docker=true -autorenew=${AUTORENEW} -earlyrenew=${EARLYRENEW} -fastgeoip=${FASTGEOIP} -mdns=${MDNS} -mdnsname=${MDNSNAME} -noauth=${NOAUTH} -port=:${PORT} -sshlb=${SSHLB} -version=${VERSION} -webfm=${WEBFM} -webroot=${WEBROOT} -ztauth=${ZTAUTH} -ztport=${ZTPORT}' >> /start.sh && \
    chmod +x /start.sh

ENTRYPOINT ["/start.sh"]

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 CMD nc -vz 127.0.0.1 $PORT || exit 1
