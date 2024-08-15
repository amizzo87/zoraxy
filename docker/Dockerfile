FROM docker.io/golang:alpine AS build

RUN mkdir -p /opt/zoraxy/source/ &&\
    mkdir -p /opt/zoraxy/config/ &&\
    mkdir -p /usr/local/bin/

RUN chmod -R 770 /opt/zoraxy/

COPY ./src/ /opt/zoraxy/source/

WORKDIR /opt/zoraxy/source/

RUN go mod tidy &&\
    go build -o /usr/local/bin/zoraxy &&\
    rm -r /opt/zoraxy/source/

RUN chmod 755 /usr/local/bin/zoraxy &&\
    chmod +x /usr/local/bin/zoraxy

FROM docker.io/alpine:3.17

# Install required packages including ZeroTier
RUN apk add --no-cache bash netcat-openbsd sudo zerotier-one

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

# Expose ports
EXPOSE ${PORT}
EXPOSE ${ZTPORT}

# Create a startup script
RUN echo '#!/bin/sh' > /start.sh && \
    echo 'if [ ! -z "$ZTAUTH" ] && [ "$ZTAUTH" != "'\''" ]; then' >> /start.sh && \
    echo '    echo "$ZTAUTH" > /var/lib/zerotier-one/authtoken.secret' >> /start.sh && \
    echo '    chmod 0600 /var/lib/zerotier-one/authtoken.secret' >> /start.sh && \
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
