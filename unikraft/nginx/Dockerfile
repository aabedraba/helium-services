FROM alpine:3.24.3@sha256:be2656061b536c1883c4034790e21d0a09c5aaaf51968b89368a3c0371ea2c9a AS prep

ARG BANGS_URL="https://raw.githubusercontent.com/imputnet/helium-services/refs/heads/main/svc/bangs/bangs.json"
ARG DICT_COMMIT="cee14e319bb7603a1157bb4d1e216be64ee82b77"

RUN apk add --no-cache curl

RUN set -eux; \
    mkdir -p /srv/bangs /srv/dictionaries/dict; \
    curl -fsSL -o /srv/bangs/bangs.json "$BANGS_URL"; \
    gzip -k9 /srv/bangs/bangs.json; \
    curl -fsSL "https://chromium.googlesource.com/chromium/deps/hunspell_dictionaries/+archive/$DICT_COMMIT.tar.gz" \
        | tar xz -C /srv/dictionaries/dict; \
    find /srv/dictionaries/dict -type f -not -name '*.gz' -exec gzip -9 {} \;

FROM nginx:1.30.1-alpine3.23-slim@sha256:a5688ea5d06cccc298505dddac20e8e28bfe7de67b015a374006593bc23700b1

COPY --from=prep /srv /srv
COPY ./nginx.conf /etc/nginx/nginx.conf
