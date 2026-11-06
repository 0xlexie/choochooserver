FROM rust:1.97.1-alpine3.23 AS builder

ARG ANKI_VERSION

RUN apk add --no-cache build-base protobuf
RUN cargo install --git https://github.com/ankitects/anki.git --tag "${ANKI_VERSION}" --root /anki-server --locked anki-sync-server

FROM alpine:3.23

RUN apk add --no-cache ca-certificates libgcc \
    && addgroup -S -g 1000 anki \
    && adduser -S -u 1000 -G anki anki \
    && mkdir -p /anki_data \
    && chown anki:anki /anki_data
COPY --from=builder /anki-server/bin/anki-sync-server /usr/local/bin/anki-sync-server

USER anki
ENV SYNC_PORT=8080
ENV SYNC_BASE=/anki_data

EXPOSE 8080
VOLUME /anki_data
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 CMD anki-sync-server --healthcheck
CMD ["anki-sync-server"]
