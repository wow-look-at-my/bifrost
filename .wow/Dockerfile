# syntax=docker/dockerfile:1
FROM node:25-alpine3.23@sha256:bdf2cca6fe3dabd014ea60163eca3f0f7015fbd5c7ee1b0e9ccb4ced6eb02ef4 AS ui-builder
WORKDIR /app
RUN apk upgrade --no-cache
COPY ui/package*.json ./
RUN npm ci
COPY ui/ ./
RUN npm run build-enterprise

FROM golang:1.27.2-alpine3.24@sha256:f92b6ef800e499660581efdabdf25d9d817a9d124eaf900924f0504e7e27e12d AS builder
WORKDIR /src
RUN apk upgrade --no-cache && \
	apk add --no-cache gcc musl-dev sqlite-dev binutils binutils-gold
ENV CGO_ENABLED=1 GOOS=linux

# The source tree is built as a Go workspace so the image contains this
# checkout's core, framework and plugins instead of the released module versions.
COPY core/ core/
COPY framework/ framework/
COPY plugins/ plugins/
COPY transports/ transports/
RUN go work init ./core ./framework ./transports && \
	find ./plugins -maxdepth 2 -name go.mod -exec dirname {} \; | xargs go work use

COPY --from=ui-builder /app/out ./transports/bifrost-http/ui

ARG VERSION=unknown
RUN cd transports && go build \
	-ldflags="-w -s -X main.Version=v${VERSION} -extldflags '-static'" \
	-a -trimpath \
	-tags "sqlite_static" \
	-o /app/main \
	./bifrost-http
RUN test -f /app/main

FROM alpine:3.23.5@sha256:fd791d74b68913cbb027c6546007b3f0d3bc45125f797758156952bc2d6daf40
WORKDIR /app
RUN apk upgrade --no-cache && \
	apk add --no-cache musl libgcc ca-certificates zlib
COPY --from=builder /app/main .
COPY --from=builder /src/transports/docker-entrypoint.sh .

ARG ARG_APP_PORT=8080
ARG ARG_APP_HOST=0.0.0.0
ARG ARG_LOG_LEVEL=info
ARG ARG_LOG_STYLE=json
ARG ARG_APP_DIR=/app/data

ENV APP_PORT=$ARG_APP_PORT \
	APP_HOST=$ARG_APP_HOST \
	LOG_LEVEL=$ARG_LOG_LEVEL \
	LOG_STYLE=$ARG_LOG_STYLE \
	APP_DIR=$ARG_APP_DIR

ENV GOGC="" \
	GOMEMLIMIT=""

RUN mkdir -p "$APP_DIR/logs" && \
	adduser -D -s /bin/sh appuser && \
	chown -R appuser:appuser /app && \
	chown -R appuser:0 "$APP_DIR" && \
	{ [ "$APP_DIR" = "/app" ] || chmod -R g=rwX "$APP_DIR"; } && \
	chmod +x /app/docker-entrypoint.sh
USER 1000:0

VOLUME ["${APP_DIR}"]
EXPOSE $APP_PORT

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
	CMD wget -q -O /dev/null http://127.0.0.1:${APP_PORT}/health || exit 1

ENTRYPOINT ["/app/docker-entrypoint.sh"]
CMD ["/app/main"]
