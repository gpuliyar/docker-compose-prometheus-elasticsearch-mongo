FROM alpine/git as git
WORKDIR /app
RUN git clone https://github.com/dcu/mongodb_exporter.git

FROM golang:alpine as builder
COPY --from=git /app/ /go/src/github.com/dcu/
RUN set -ex; \
    apk --no-cache add curl git make perl; \
    curl -s https://glide.sh/get | sh; \
    cd /go/src/github.com/dcu/mongodb_exporter; \
    make release

FROM alpine
ENV MONGODB_URL "mongodb://${MONGO_SERVICE}:${MONGO_PORT}"
RUN apk add --update ca-certificates
COPY --from=builder /go/src/github.com/dcu/mongodb_exporter/release/mongodb_exporter-linux-amd64 /usr/local/bin/mongodb_exporter
EXPOSE 9001
ENTRYPOINT [ "mongodb_exporter" ]
