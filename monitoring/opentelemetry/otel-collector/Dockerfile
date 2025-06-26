# Creating a builder container to extract the binary file into an alpine image
# The collector image is present at /otelcontribcol
FROM otel/opentelemetry-collector-contrib:latest AS builder

# Set a parent alpine image
FROM alpine:latest

# Install necessary dependencies
RUN apk add --no-cache \
            openjdk17-jre-headless \
            ca-certificates \
            bash \
        && mkdir -p /tmp /opt \
        && chmod 1777 /tmp

COPY --from=builder /otelcol-contrib /usr/local/bin/otelcol-contrib

COPY --chmod=444 ./otel-collector-config.yaml /etc/otelcol-contrib/config.yaml

# Set JAVA_HOME and update PATH
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk
ENV PATH="$JAVA_HOME/bin:$PATH"

ENTRYPOINT [ "/usr/local/bin/otelcol-contrib", "--config", "/etc/otelcol-contrib/config.yaml"]

# Expose Prometheus metrics, Opentelemtetry gRPC and http port respectively
EXPOSE 8889 4317 4318