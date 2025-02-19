FROM fluent/fluentd:v1.17-debian-1
USER root
RUN \
    --mount=type=cache,target=/var/lib/apt/lists \
    --mount=type=cache,target=/var/cache/apt/archives \
    apt update \
    && apt install -y g++ make
RUN gem install fluent-plugin-rewrite-tag-filter

USER fluent