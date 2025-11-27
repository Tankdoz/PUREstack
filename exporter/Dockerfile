# Stage 1: Build the binary using Go
FROM golang:1.22-alpine AS builder

WORKDIR /app
# Clone the source directly from GitHub
RUN apk add --no-cache git \
    && git clone https://github.com/ar51an/unbound-exporter.git . \
    && go build -o unbound-exporter .

# Stage 2: Minimal runtime image
FROM alpine:3.19
WORKDIR /app

# Copy the compiled binary from builder stage
COPY --from=builder /app/unbound-exporter /usr/local/bin/unbound-exporter

# Expose metrics port
EXPOSE 9167

# Run the exporter binary
ENTRYPOINT ["/usr/local/bin/unbound-exporter"]
