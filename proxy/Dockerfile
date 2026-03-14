FROM caddy:2-builder AS builder

# Build Caddy with the official forward proxy plugin.
# https://github.com/caddyserver/forwardproxy
RUN xcaddy build \
    --with github.com/caddyserver/forwardproxy@caddy2

FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
