FROM alpine:latest

LABEL org.opencontainers.image.source=https://github.com/n-Arno/nfs-kapsule

RUN apk add --no-cache --update nfs-utils flock tini && \
    rm -rf /var/cache/apk && \
    mkdir -p /var/lib/nfs/rpc_pipefs /var/lib/nfs/v4recovery /exports

COPY init /init

VOLUME /exports

EXPOSE 111 2049 20048

ENTRYPOINT ["/init"]
